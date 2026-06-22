# Java 后端的设计

后端是个 Spring Boot 3.4 的单体，承载用户、帖子、评论、私信、AI 学长、内容审核、还有整个管理端的业务，按领域分了包（社交、社区、AI、管理端、基础设施）。规模是 285 个 Java 文件、约 2.4 万行，33 个 Controller、75 个 Service、约 38 张表、20 个 Flyway 迁移脚本。这块基本是我一个人写的。

技术上没什么花哨的：MyBatis-Plus 做持久化，JWT 鉴权，统一的 `Result<T>` 响应加全局异常，Flyway 管迁移。AI agent 的推理在 v2 之后转发给了独立的 Python 服务，Java 这边是转发层，同时保留了一套进程内的兜底。下面挑几块我觉得能聊的讲。

## 反幻觉判定的 Java 侧

AI 找帖那道相关性判定，Java 这边也有一份对应实现（agent 的进程内兜底版本）。思路跟 Python 侧一样——把"为什么匹配"换成"哪些是真匹配"，但实现上我用了 Spring AI 的结构化输出，直接让它吐一个 `List<Long>` 的帖子 ID 回来。这里我试过让它返回一个包着数组的对象，发现不如直接返回裸 JSON 数组稳，Qwen 更习惯后者。

```java
// ai/agent/engine/RelevanceJudge.java
/**
 * 帖子相关性判定闸。
 * 背景：向量检索永远返回"最近的 topK"，而 bge-m3 对中文短文本相似度普遍偏高
 * （无关项也能到约 0.7），单纯卡分数阈值分不开相关与无关。
 * 这里换个问法：把"为何匹配"改成"是否真的匹配"，从严判定、只放行真正相关的帖子。
 */
@Slf4j
@Component
public class RelevanceJudge {

    private static final String SYSTEM_PROMPT = """
            你是校园墙的帖子相关性判定器。给定用户诉求和若干候选帖子，判断【哪些帖子真的能满足该诉求】。
            判定从严，宁缺毋滥：内容明显无关、仅是测试/占位、物品或品类对不上的，一律不要。
            只输出真正相关帖子的 postId 组成的 JSON 数组，例如 [294]；若没有，输出空数组 []。
            """;

    private static final ParameterizedTypeReference<List<Long>> LONG_LIST =
            new ParameterizedTypeReference<>() {};

    private final ChatClient agentChatClient;

    public Set<Long> judge(String question, List<Map<String, Object>> candidates) {
        if (candidates == null || candidates.isEmpty()) {
            return Set.of();
        }
        // ... 拼候选帖子文本：postId / 分类 / 内容 ...
        List<Long> ids = agentChatClient.prompt()
                .system(SYSTEM_PROMPT)
                .user(user)
                .call()
                .entity(LONG_LIST);
        if (ids == null) {
            return Set.of();
        }
        Set<Long> result = ids.stream().filter(Objects::nonNull).collect(Collectors.toSet());
        log.info("相关性判定: 候选{}条 → 相关{}条 {}", candidates.size(), result.size(), result);
        return result;
    }
}
```

这里还有个跨仓的暗雷我专门加了注释。Python 那边为了省模型调用，把"匹配理由"的生成关掉了；但 Java 的找帖工具会去读返回里的 `reason` 字段填进帖子卡片。所以 `/match-posts` 这个给 Java 调的接口必须保持返回 reason，只在 Python agent 自己走的路径里关掉，不然卡片上的匹配理由会静默变成 null。

## 管理端权限：物理隔离

后台账号和 C 端学生账号是完全分开的两套，不复用同一张表，连 JWT 密钥都用两套并在 token 里打 `type=admin`。鉴权在一个拦截器里做：解析 token、查账号是否启用、过一道 Redis 黑名单（用来强制踢人下线）、把当前管理员和数据范围写进上下文，最后反射读方法上的 `@RequirePermission` 注解校验权限码。

```java
// admin/security/AdminAuthInterceptor.java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    if ("OPTIONS".equalsIgnoreCase(request.getMethod()) || !(handler instanceof HandlerMethod)) {
        return true;
    }
    String token = extractToken(request);
    if (token == null || token.isBlank()) {
        throw new BusinessException(401, "请先登录");
    }
    Long adminUserId;
    try {
        adminUserId = adminJwtUtil.parse(token);
    } catch (Exception e) {
        throw new BusinessException(401, "登录已失效，请重新登录");
    }
    // 踢人下线：黑名单命中则强制重新登录
    if (Boolean.TRUE.equals(stringRedisTemplate.hasKey(AdminManageService.blacklistKey(adminUserId)))) {
        throw new BusinessException(401, "登录已失效（已被强制下线）");
    }
    // ... 校验账号存在且启用、写 RequestContext（含数据范围）...
    RequestContext.setDataScope(adminAuthService.resolveDataScope(adminUserId));
    // 方法级权限校验
    RequirePermission rp = ((HandlerMethod) handler).getMethodAnnotation(RequirePermission.class);
    if (rp != null) {
        List<String> owned = adminAuthService.loadPermissions(adminUserId);
        if (!hasPermission(owned, rp)) {
            log.warn("管理端越权: adminId={}, 需要={}, 拥有={}", adminUserId, String.join(",", rp.value()), owned);
            throw new BusinessException(403, "无权限执行该操作");
        }
    }
    return true;
}
```

权限码是 `module:action` 的形式（`moderation:queue`、`user:ban`、`role:assign` 这些），注解支持 AND/OR 组合。权限表里 `perm_type=menu` 的项直接复用成菜单，没单独建菜单表。数据范围分 ALL 和 SCHOOL，落在拦截器上下文里，支持按学校隔离运营。还有个小防护：生产环境如果还用着默认密钥，启动时会直接告警。

## 横切的东西都收进 AOP

管理端所有写操作都要能审计，但我不想让审计逻辑散进每个 Service，也不能因为记日志失败或者慢就拖垮主流程。所以做了个 AOP 切面，`try-catch-finally` 包着主逻辑：成功失败都在 finally 里拼一条审计记录异步落库，连失败的错误摘要和耗时都记下来。审计写库失败只打个 warning，绝不影响主流程。只标写操作、不标读接口，免得日志爆量。

```java
// admin/audit/OperationLogAspect.java
@Aspect
@Component
public class OperationLogAspect {

    @Around("@annotation(operationLog)")
    public Object around(ProceedingJoinPoint pjp, OperationLog operationLog) throws Throwable {
        long start = System.currentTimeMillis();
        int status = 1;
        String errorMsg = null;
        try {
            return pjp.proceed();
        } catch (Throwable t) {
            status = 0;
            errorMsg = t.getMessage();
            throw t;                                  // 记失败后原样抛出
        } finally {
            try {
                AdminOperLog entry = new AdminOperLog();
                entry.setAdminUserId(RequestContext.getAdminUserId());
                entry.setOperatorName(RequestContext.getAdminName());
                entry.setModule(operationLog.module());
                entry.setAction(operationLog.action());
                entry.setTargetId(resolveTargetId(pjp.getArgs()));   // 启发式取第一个 Number
                // ... 补 reqMethod / reqUri / clientIp（取 X-Forwarded-For 首段）...
                entry.setStatus(status);
                entry.setErrorMsg(truncate(errorMsg));               // 截 500
                entry.setCostMs(System.currentTimeMillis() - start);
                auditLogService.saveAsync(entry);                    // 异步落库
            } catch (Exception e) {
                log.warn("审计日志记录失败: {}", e.getMessage());      // 审计失败不影响主流程
            }
        }
    }
}
```

限流也是统一的，用 Bucket4j 令牌桶按接口分级：登录 5 次/分钟、发帖 10 次、点赞收藏 30 次、评论 20 次、其余 60 次。

## 线程池：按"丢得起丢不起"分策略

AI 帖子入库（要调看图模型，慢）和长期记忆抽取这两个异步任务，我用了不同的拒绝策略。这是个我想了挺久的点：同样是异步，但它俩对"队列满了怎么办"的容忍度完全不一样。入库丢了，帖子就进不了知识图谱；记忆抽取丢了，下一轮对话还能再抽。

```java
// config/AsyncConfig.java
// AI 帖子入库：队列满由调用线程兜底执行，保证不丢
@Bean("aiIngestExecutor")
public Executor aiIngestExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(2);
    executor.setMaxPoolSize(4);
    executor.setQueueCapacity(100);
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.initialize();
    return executor;
}

// 长期记忆抽取：尽力而为，队列满直接丢弃本次（下轮还会再抽），
// 绝不像入库那样回灌调用线程（那会卡住用户回复的请求线程）
@Bean("aiMemoryExecutor")
public Executor aiMemoryExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(2);
    executor.setMaxPoolSize(4);
    executor.setQueueCapacity(100);
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());
    executor.initialize();
    return executor;
}
```

关键就在拒绝策略：入库用 `CallerRunsPolicy`，队列满了回灌到调用线程保证不丢；记忆抽取用 `DiscardPolicy`，丢了就丢了——它绝对不能回灌到用户请求的线程上，那会拖慢用户收到回复的速度。如果一刀切都用 CallerRuns，高峰期记忆抽取就会把请求线程占满。

## 运营看板用 HyperLogLog 算活跃

看板要日活、周活、月活。如果用 SQL `DISTINCT` 或者拿 Set 存全量用户 ID，成本会随用户量线性涨。我用的是 Redis 的 HyperLogLog：每次鉴权通过就把用户 ID `PFADD` 进当天的 key，算周活月活直接对最近 7 天/30 天的多个 key 做 `PFCOUNT` 求并集去重。

```java
// admin/stat/service/StatService.java
private static final DateTimeFormatter DAY_FMT = DateTimeFormatter.BASIC_ISO_DATE; // yyyyMMdd

/** 单日 DAU = PFCOUNT(当日 HLL) */
public long dau(LocalDate date) {
    try {
        Long n = stringRedisTemplate.opsForHyperLogLog().size(dauKey(date));
        return n == null ? 0L : n;
    } catch (Exception e) {
        return 0L;
    }
}

/** 滚动窗口活跃 = 最近 days 天 HLL 并集去重 */
private long activeWindow(int days) {
    try {
        LocalDate today = LocalDate.now();
        String[] keys = new String[days];
        for (int i = 0; i < days; i++) {
            keys[i] = dauKey(today.minusDays(i));
        }
        Long n = stringRedisTemplate.opsForHyperLogLog().size(keys);  // 多 key PFCOUNT 求并集
        return n == null ? 0L : n;
    } catch (Exception e) {
        return 0L;
    }
}
```

HLL 单个 key 就十几 KB、标准误差大约 0.8%，对看板这种趋势性数字完全够用，三个指标共用一套 HLL。打点的时候顺手给 key 续 40 天 TTL 覆盖周活月活的窗口；日活还会被每天凌晨的快照任务固化进 `stat_daily` 表。

## 数据库这块

约 38 张表，挑几张关键的说：管理端 RBAC 是 5 张表（账号、角色、权限、加两张关联），跟 C 端 user 表物理隔开；`post` 是社区的核心表，随着板块演进（二手、兼职、推广、组队）加了价格、薪资、联系方式这些字段；`ai_user_memory` 上有个 `content_hash` 唯一键，给 Python 的记忆 worker 做跨语言幂等去重用。

迁移有个我处理过的坑：MySQL 8 不支持 `ADD COLUMN IF NOT EXISTS`，Flyway 又要求脚本能重复执行。我统一用 `information_schema` 先查列/索引在不在，再用 `PREPARE/EXECUTE` 拼出真正的 ALTER 来守卫，做到幂等。加 `content_hash` 唯一键的那次还特意没回填存量数据——因为存量里有重复内容，回填会让建唯一键失败，而 MySQL 的唯一键允许多个含 NULL 的组合，所以让去重只对新写入生效，保证生产能平滑升级。

## 其它一些细节

私信用 WebSocket，会话管理目前是单机内存版，预留了换 Redis 分布式实现的口子。缓存是 Redis 多级 TTL（用户 2 小时、帖子详情 30 分钟、列表 5 分钟）加统一失效。审核服务和指标采集之间一度互相依赖形成了循环，我用 `@Lazy` 打破的。AI 代发帖多机回调，是靠请求头 `X-Backend-Base` 告诉 AI 服务回调哪台后端，再转发用户 JWT 让 Python 以用户身份去发帖、复用审核和限流。
