# AI 微服务的设计

这是整个项目我花心思最多的一块，基本是我一个人写的。技术栈是 FastAPI + LangGraph + Neo4j，对外提供"AI 学长"的全部能力：知识问答、帮你找帖、看图发帖、还有异步的长期记忆。app 目录大概 4200 行 Python、49 个模块，配了 19 个测试文件 98 个用例（全 mock，秒级，进 pre-push 钩子），对外 18 个 HTTP 接口（其中 9 个是给 Java 后端调的 GraphRAG 契约）。

下面挑几块讲讲。

## 整个 agent 是怎么转的

一次问答大致是这么走的：

```
小程序问 → 指代消解（把"它"还原成上文）→ 看图增强（如果带图）
   → 规划（Planner 决定调哪些工具）
   → 执行工具：找帖走 Neo4j 帖子向量索引 + 相关性判定
              知识问答走 Neo4j 文档向量索引 + 图遍历补事实
   → 一道闸：证据不够且没超时 → 带着反馈重新规划一次
   → 严格基于证据合成答案
   → 对话结束后异步抽取长期记忆
```

它是 LangGraph 的 Plan-Execute 加一次条件重规划，不是纯 ReAct。我特意做得保守一点——一方面是对齐 Java 那边的行为，另一方面是控制对大模型的调用次数（原因下面讲）。

模型分三个角色，都是本地优先、云端兜底：对话模型负责意图识别、抽三元组、合成答案；embedding 用 bge-m3（1024 维）；还有个看图模型，用来支持"拍张图问这是哪/找类似的帖"和看图发帖。

## 让它别瞎答（我最得意的一块）

这块的来龙去脉在[贡献说明](./portfolio.md)里讲过，这里补代码和细节。

核心是一道相关性判定。问法很关键——不是"为什么匹配"，而是"这几个候选里哪些是真能满足诉求的"，让模型从严判断、只回真正相关的帖子 ID。判定不可用的时候才退回去用分数阈值兜底，两者是二选一。

代码里还藏着两个我后来补的兜底。一个是意图硬过滤的空召回兜底：找帖默认会先猜个意图（丢东西 / 二手 / 兼职……）去 `WHERE` 过滤，但模型猜错的时候（比如"我把这本书丢了"被猜成丢失而不是二手）会在召回阶段就把真正相关的帖子滤掉，判定根本没机会救。所以我让它在"带意图过滤却一条都没召回"的时候，去掉意图再全量召回一次，把精度交给后面那道从严的判定。另一个是"匹配理由"的下沉——只给判定后留下的那一两条帖子生成理由，不给会被丢掉的候选白算。

```python
# app/agent/tools.py
def search_posts(state: AgentState, args: dict, deps: AgentDeps) -> dict:
    query = (args.get("query") or state.get("rewritten_question") or "").strip()
    intent = args.get("intent")
    # with_reason=False：召回阶段不算「匹配理由」，留到判定后只对保留帖生成（省模型调用）
    res = deps.store.match_posts(query, intent, settings.AGENT_TOP_K, with_reason=False)
    matches: list[dict] = res.get("matches", []) or []

    # 意图硬过滤兜底：模型猜错意图会在召回阶段就筛掉真正相关的帖、判定都没机会救。
    # 带意图过滤却空召回时，去掉意图全召回一次，把精度交给后面从严的判定。
    if not matches and res.get("intent"):
        res = deps.store.match_posts(query, None, settings.AGENT_TOP_K,
                                     with_reason=False, filter_intent=False)
        matches = res.get("matches", []) or []

    # 主闸：相关性判定；异常时退回分数阈值（二选一，不是叠加）
    relevant_ids: set[int] = set()
    used_judge = False
    try:
        relevant_ids = relevance.judge(query, matches)
        used_judge = True
    except Exception as e:  # noqa: BLE001
        logger.warning("相关性判定不可用，退化为分数阈值(%.2f): %s",
                       settings.AGENT_MATCH_SCORE_THRESHOLD, e)

    kept: list[dict] = []
    for m in matches:
        try:
            pid = int(m.get("post_id"))
        except (TypeError, ValueError):
            continue
        score = _to_float(m.get("score"))
        keep = (pid in relevant_ids) if used_judge else (score >= settings.AGENT_MATCH_SCORE_THRESHOLD)
        if keep:
            kept.append(m)

    # 只给判定后留下的帖子生成匹配理由（通常 1~3 条）
    for m in kept:
        if not m.get("reason"):
            m["reason"] = deps.store.reason_for(query, m.get("description") or "")
```

跑评测，判定之后的 precision 是 0.97、judge_f1 0.93，"绝对不该出现"的帖子漏出是 0，知识问答的对抗样本拒答率 100%。

## 围着内网单槽大模型做的优化

对话模型是内网一台机器上的 35B，用 llama.cpp、单槽，同一时刻只能处理一个请求，这是真瓶颈。下面这两段是相关的代码。

第一段是构造模型客户端的地方。我把 SDK 的自动重试关掉了——默认重试两次，遇到超时会把单次 60 秒放大成 180 秒，还会把更多请求怼回排队的槽位，越堵越死。同时关掉了走环境代理（内网端点不该走代理，否则开发机挂了梯子会劫持这条连接）。下面那个 `_with_fallback` 是我抽出来的链式降级，对话模型走"本地主→本地备→云端"三级。

```python
# app/graphrag/llm.py
def _make(api_key: str, base_url: str, *, max_retries: int = 0) -> OpenAI:
    # 内网 35B 是单槽，同一时刻只处理一个请求。
    # SDK 默认 max_retries=2：超时(60s)后还会再重试 2 次，把单次放大成最长 180s，
    # 并把更多请求怼回排队的槽位 → 雪崩。所以默认不重试、超时即快速失败。
    return OpenAI(api_key=api_key or "ollama", base_url=base_url, max_retries=max_retries,
                  http_client=httpx.Client(trust_env=False))  # 内网端点不走环境代理


def _with_fallback(role: str, primary: Callable[[], T], *fallbacks: Optional[Callable[[], T]]) -> T:
    """先跑 primary()；失败按顺序降级到各 fallback()（None 跳过），全失败抛最后一个异常。
    对话模型走 本地主 → 本地备 → 云端 三级链。"""
    chain = [primary] + [f for f in fallbacks if f is not None]
    last_exc: Optional[Exception] = None
    for i, fn in enumerate(chain):
        try:
            return fn()
        except Exception as e:  # noqa: BLE001
            last_exc = e
            tier = "primary" if i == 0 else f"fallback{i}"
            logger.warning("[%s] %s 调用失败: %s", role, tier, e)
    raise last_exc  # type: ignore[misc]
```

这里有个硬约束：embedding 默认不做降级。因为 Neo4j 里已经落盘的向量都是 bge-m3 的 1024 维，一旦换模型或换维度，存量向量跟查询向量对不上，整库检索就废了。所以这条降级我直接锁死成关。

第二段是"墙钟预算"。我给整轮设一个截止时间，在每个节点的边界上判断。超了就降级：跳过最慢的看图、不再重规划、直接拿已有证据兜底合成。我没用硬中断——同步线程池里没法把一个正在阻塞的模型调用掐断，强行 kill 会留脏连接；节点边界判断虽然掐不断正在跑的那一次，但能止住最大的放大源，也就是多次串行调用的叠加。还有个细节：`gate` 这道闸判断"证据够不够"，用的是 `fruitful`（真的捞到有用东西）而不是 `ok`（没抛异常）——因为找帖检索到 0 条的时候是 `ok=True` 但 `fruitful=False`，早期用 `ok` 判会把空结果误当成"够了"直接去合成。

```python
# app/agent/nodes.py
def over_budget(state: AgentState) -> bool:
    """整轮墙钟预算是否耗尽。只在【节点边界】判定——掐不断正在跑的那次调用，
    但足以止住最大的放大源：跳过看图、不再重规划、超时拿已有证据兜底。"""
    dl = state.get("_deadline") or 0
    return bool(dl) and time.monotonic() >= dl


def gate(state: AgentState, deps: AgentDeps) -> dict:
    obs = state.get("observations") or []
    # 用 fruitful 而非 ok：找帖检索 0 命中时 ok=True 但 fruitful=False
    insufficient = (not obs) or all(not o.get("fruitful") for o in obs)
    replan_count = state.get("replan_count", 0)
    # 预算耗尽就不再重规划（避免第二轮规划+执行把单轮耗时翻倍），直接合成
    if insufficient and replan_count < 1 and over_budget(state):
        metrics.inc_timeout("replan")
        return {"_route": "synthesize"}
    if insufficient and replan_count < 1:
        feedback = f"已尝试：{tried}；均未获得可用证据。"   # 把"试了什么/为何不够"喂回 planner
        metrics.inc_replan()
        return {"replan_count": replan_count + 1, "_route": "replan", "replan_feedback": feedback}
    return {"_route": "synthesize"}
```

预算（默认 100 秒）我特意设得比 Java 那边的读超时（120 秒）低，让 AI 服务先优雅降级，而不是干等到 Java 抛硬错。三处降级我都埋了 Prometheus 计数，能看到到底降级了多少次。

## 长期记忆：异步 + 冲突消解

记忆这块要解决三个问题：抽取要调大模型、不能卡住对话主链路；记忆之间会打架（旧的说"喜欢详细"、这轮说"想要简短"）；还有时效性的记忆（"这周丢了校园卡"）不能永久占着注入窗口。

做法是对话结束往 Redis Streams 里 `XADD` 一条，单独一个 worker 用 consumer group 去消费、抽取、落库。失败按次数重投，超了进死信队列；还有 `XAUTOCLAIM` 兜 worker 崩溃的情况。冲突消解我用的是软失效而不是物理删——被新信息取代的旧记忆，把过期时间置成现在，这样既复用了"按过期时间过滤"那套逻辑（零改动）、又保留了审计痕迹。时效记忆落库时就给个 TTL，过期自动退出注入窗口。落库幂等靠 `sha256(user_id|content)` 唯一键，防 Streams 重投造成重复。

```python
# app/persistence/user_memory.py
def content_hash(user_id: int, content: str) -> str:
    return hashlib.sha256(f"{user_id}|{(content or '').strip()}".encode()).hexdigest()


def save_new_facts(user_id: int, session_id: Optional[str], items: list[dict]) -> int:
    # event 类记忆落库即给 TTL，过期后 list_for_user 自动过滤，不再永久占着 Top-N
    event_ttl = timedelta(days=max(1, settings.MEMORY_EVENT_TTL_DAYS))
    with get_session() as s:
        for it in items or []:
            content = (it.get("content") or "").strip()
            h = content_hash(user_id, content)            # 唯一键，幂等防 Streams 重投
            if s.execute(select(AiUserMemory.id)
                         .where(AiUserMemory.user_id == user_id,
                                AiUserMemory.content_hash == h)).first():
                continue
            mtype = it.get("type") or "fact"
            expire = (datetime.utcnow() + event_ttl) if mtype == "event" else None
            s.add(AiUserMemory(user_id=user_id, memory_type=mtype, content=content[:500],
                               content_hash=h, expire_time=expire))
        s.commit()


def expire_contents(user_id: int, contents: list[str]) -> int:
    """冲突消解：把被取代的旧记忆软失效（过期时间置成现在）。
    软失效而非物理删——复用过期过滤逻辑零改动，还留审计痕迹。"""
    now = datetime.utcnow()
    hashes = [content_hash(user_id, c) for c in cleaned]
    rows = s.execute(select(AiUserMemory)
                     .where(AiUserMemory.user_id == user_id, AiUserMemory.content_hash.in_(hashes))
                     .where(or_(AiUserMemory.expire_time.is_(None),
                                AiUserMemory.expire_time > now))).scalars().all()
    for row in rows:
        row.expire_time = now
```

之所以选 Redis Streams 而不是上 RabbitMQ，是因为 Redis 我们本来就在用（做缓存和 checkpoint），consumer group 自带的 PEL/ACK/重试语义已经够了，没必要为这个再引入一个新中间件、多一份运维负担。

## 看图发帖

这是个多轮的"人在回路"：你说一句、它出个草稿、你改、确认了再发。我用 LangGraph 的子图，在"展示草稿"那个节点 `interrupt()` 挂起等你，你回话了再 `resume` 唤醒。

有几个我刻意守住的约束：第一，绝不让 AI 服务越过 Java 直接写帖子表——发帖必须转发用户自己的 JWT 去调 Java 的 `/posts/publish`，否则就绕过了封禁校验、内容审核、限流这些。第二，草稿的权威状态在数据库的 `ai_post_draft` 表里，resume 之前用状态做幂等闸，已发布/已取消的拒绝再操作。第三，多机开发的时候一个 AI 实例可能要回调不同机器的后端，回调地址过一个白名单防 SSRF。

这里有个务实的取舍：interrupt 的执行态我用的是进程内的 MemorySaver，进程一重启就丢了。当时我们的 Redis 版本不支持持久化执行态那套，单容器部署能接受重启丢——因为草稿本身还在数据库里，用户重新描述一下就行。前端那边也对这个易失性做了防御（历史回放时把待确认的草稿降级成只读，免得用户点了"发布"却必然失败）。

## 怎么保证不把它改坏

测试分两层。`tests/` 全是 mock、秒级、进 pre-push 钩子，每次推送都跑。`evals/` 是连真实模型和 Neo4j 的评测，慢，手动跑、不进钩子。评测的金标集是我自己标的：意图识别 80 条（每类 20），找帖 18 条（16 条锚定帖加 2 条负样本），知识问答 14 条（9 条可答加 5 条对抗，配套导了 16 篇真实文档）。跑的时候对着基线比涨跌、留 5% 容忍度当门禁。改提示词、改检索参数、动判定逻辑之前之后，我都会跑一遍——它是我所有 AI 改动的安全网。
