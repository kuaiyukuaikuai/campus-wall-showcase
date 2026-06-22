# 前端 / 运营后台 / 运维 / 数据管道

这篇把剩下几块凑一起讲，每块挑一两个我觉得有意思的地方加点代码。前端里 AI 对话和实时私信这两条是我写的核心链路，运维监控和数据管道基本也是我做的，运营后台的框架和权限是我搭的。

## 微信小程序

uni-app 3 加 Vue 3，约 1.6 万行、32 个页面。

### WebSocket 这块踩得最实

私信要长连接实时收发，但小程序的环境挺难伺候：没有 EventSource，网络一切换、后台一回收就断连，而且连接还没 OPEN 的时候直接 `send` 会静默把消息丢了，每个会话页各自建连还会冒出好几条连接。我做成了一个全局单例，配心跳保活、断线重连、和一个"没就绪就先入队"的队列。

```javascript
// src/utils/websocket.js
class WebSocketManager {
  constructor() {
    this.ws = null
    this.reconnectCount = 0
    this.maxReconnectCount = 5
    this.reconnectInterval = 3000
    this.heartbeatInterval = 30000
    this.isReady = false
    this.messageQueue = [] // 连接未就绪时先攒着
  }

  send(data) {
    if (!this.ws) { console.warn('WebSocket 未初始化，消息已丢弃'); return false }
    if (!this.isReady) {            // 没就绪 → 入队，OPEN 后补发
      this.messageQueue.push(data)
      return true
    }
    return this.doSend(data)
  }

  flushMessageQueue() {
    if (this.messageQueue.length === 0) return
    const queue = [...this.messageQueue]
    this.messageQueue = []
    for (const msg of queue) this.doSend(msg)
  }

  reconnect() {
    if (this.reconnectCount >= this.maxReconnectCount) return  // 上限保护，别无限重试
    this.stopReconnect()
    this.reconnectTimer = setTimeout(() => {
      this.reconnectCount++
      this.connect(this.userId)
    }, this.reconnectInterval)
  }

  startHeartbeat() {
    this.stopHeartbeat()
    this.heartbeatTimer = setInterval(() => {
      if (this.ws) this.doSend({ type: 'ping' })   // 30s 应用层心跳
    }, this.heartbeatInterval)
  }
  // onSocketOpen 里：isReady=true、reconnectCount 归零、起心跳、把攒着的消息补发出去
}
export default new WebSocketManager()
```

重连设了 5 次上限，不无限重试，免得后端真挂了的时候前端疯狂建连搞出连接风暴。心跳用应用层自己发 ping，不指望底层 keepalive，因为小程序各端表现不一致。

### AI 发帖的草稿卡片，和对后端易失态的防御

让用户在对话里口述发帖，我做成了消息流里的草稿卡片（待确认、已被新版替代、已发布、已取消四个状态），而不是跳到另一个页面——这样更贴合聊天的心智。意图识别我全部下沉到后端了，前端不再写一堆客户端正则去猜该找帖还是发帖，带不带图都走同一个入口，由后端返回的 action 决定。

这里有个对后端的防御我觉得值得说。后端那个 LangGraph 的草稿执行态是存在进程内存里的，进程一重启就没了。如果前端傻乎乎地把历史里"待确认"的草稿照原样渲染出来，用户点"确认发布"会必然失败（后端找不到那个挂起点）。所以历史回放的时候，我把"待确认"强制降级成只读，不恢复它的可操作状态。

```javascript
// src/pages/ai/chat.vue
// 发送：意图识别一律下沉后端 —— 带不带图都走 askAgent，由后端 action 决定 找帖/答疑/发帖
const sendMessage = async () => {
  if (!inputText.value.trim() || isThinking.value) return
  const userText = inputText.value
  const hasImages = !activeDraftId.value && pendingObjectNames().length > 0
  // 带图时图还没传完就拦住，避免静默掉图
  if (hasImages && pendingImages.value.some(p => p.status === 'uploading')) {
    uni.showToast({ title: '图还在上传，稍等一下再发', icon: 'none' })
    return
  }
  if (activeDraftId.value) {
    if (/^(算了|不发了|取消|退出|不改了|结束)/.test(userText.trim())) {
      cancelInlineDraft(activeDraftId.value)   // 文本里的退出意图 → 直接取消
    } else {
      await editActiveDraft(userText)          // 草稿编辑中，这条就是改稿指令
    }
  } else {
    await askAgent(userText, hasImages ? pendingObjectNames() : [])
  }
}

// 历史回放：把"待确认"降级成只读 —— 进程重启后那个草稿已经没法 resume 了，
// 不恢复它的可操作态，免得用户点了发布却必然失败
if (record.role !== 'USER' && (mt === 'post_draft' || mt === 'post_published')) {
  const meta = parseMetaObj(record.metadataJson) || {}
  let status = meta.status || (mt === 'post_published' ? 'published' : 'awaiting')
  if (status === 'awaiting') status = 'superseded'
  return { ...base, role: 'ai', type: 'draft', draftId: meta.draftId || '', draft: meta.draft || {}, status }
}
```

### 其它几个小处理

请求层统一注入 JWT、按 `code===200` 解包响应。401 我加了个模块级的布尔锁，因为 token 过期时往往好几个并发请求一起返回 401，不加锁会触发好几次跳登录页；加了锁就 2 秒内只跳一次。请求超时也是分情况的：普通问答 10 秒，但带图、出草稿、改稿这些走大模型/看图模型的，单独放到 90 秒，而不是把全局超时一刀切放大（那样普通接口出故障时会挂很久）。点赞收藏用 Pinia 做乐观更新——先本地翻转 + 持久化、再发请求，成功了用服务端返回的真值校正，失败回滚到之前的快照。

## 运营后台

Vue 3 加 Element Plus，约 2800 行、14 个权限页面。这块我做的主要是一套前端 RBAC 框架。

权限分三级拦截：路由级（路由表上声明 `meta.perms`，全局守卫校验）、菜单级（按权限过滤只渲染有权的菜单）、按钮级（一个 `v-permission` 指令，没权限直接把 DOM 元素删掉）。

```javascript
// src/directives/permission.js
import { useUserStore } from '../stores/user'

// v-permission="'moderation:approve'" 或 v-permission="['a','b']"
// 只做界面隐藏（后端的 @RequirePermission 才是真正鉴权的地方）。没权限就把元素移除。
export default {
  mounted(el, binding) {
    const store = useUserStore()
    const val = binding.value
    const ok = Array.isArray(val) ? val.some((c) => store.hasPerm(c)) : store.hasPerm(val)
    if (!ok && el.parentNode) {
      el.parentNode.removeChild(el)
    }
  }
}
```

有两个点我想强调。一是我很清楚前端权限只是"体验"不是"安全"——它能被绕过，真正的鉴权在后端，前端只是不让用户看到点了也没用的入口。按钮级我选择直接移除 DOM 而不是 disable，就是不想留一个能点、点了又被后端拒绝的元素。二是权限敏感的数据（角色、权限列表）我故意只放在 Pinia 内存里、不落 localStorage，防止本地篡改提权；代价是 SPA 刷新后内存清空、权限就丢了，所以路由守卫里检测到没加载就先去拉一次 `/admin/auth/me` 把权限重建出来。

```javascript
// src/router/index.js — 全局前置守卫
router.beforeEach(async (to) => {
  const store = useUserStore()
  if (to.meta.public) {                                  // 登录页：放行；已登录就别停在登录页
    if (to.name === 'login' && store.token) return { name: 'dashboard' }
    return true
  }
  if (!store.token) {                                    // 未登录 → 拦下，记下原目标
    return { name: 'login', query: { redirect: to.fullPath } }
  }
  if (!store.loaded) {                                   // 刷新后内存权限丢了 → 重新拉取
    try {
      await store.fetchMe()
    } catch (e) {
      store.logout()
      return { name: 'login' }
    }
  }
  if (to.meta.perms && !store.hasPerm(to.meta.perms)) {  // 没权限 → 统一 403
    return { name: 'forbidden' }
  }
  return true
})
```

顺便提一句，后台所有接口走相对路径 `/api`，由 nginx 同源反代到后端和 Grafana，这样构建产物里一个内网地址都不硬编码，IP 变了只改反代一处，也省了 CORS。图表统一封了个 `BaseChart`，按需引入 ECharts 模块控制打包体积，看板四组数据用 `Promise.all` 并发加载。新增一个功能页基本只要改两处（路由表加一条、API 加一个函数），菜单和权限会自动生效。

## 运维与监控

整套基础设施是一份 Docker Compose 拉起的，17 个容器（数据层、AI、监控采集、可视化告警、展示）。Prometheus 每 15 秒抓一次，配了 8 条告警规则分 4 组，告警走 Alertmanager 分组抑制后再转出去。

这里有个我自己写的小服务。Alertmanager 原生只支持 PagerDuty、Slack、邮件那一套，不支持国内的企业微信和钉钉群机器人，而我们团队就俩人、没人盯邮件。我没去上商业告警平台（重，还要公网），而是写了个 150 行左右的 FastAPI 转发器，接收 Alertmanager 的 webhook、按告警级别渲染成 markdown、再按配置转给企微或钉钉。这个转发器我特意做了三层兜底，因为它是"监控的监控"，最不能在关键时刻自己挂掉：

```python
# campus-wall-ops/alert-adapter/app.py
@app.post("/alert")
async def alert(request: Request):
    try:
        payload = await request.json()
    except Exception as e:                       # 兜底1：请求体解析失败就当空 dict 继续
        logger.warning("收到无法解析的 alert 请求: %s", e)
        payload = {}

    markdown = build_markdown(payload)

    if not ALERT_WEBHOOK:                         # 兜底2：没配 webhook 就只记日志，不崩
        logger.warning("ALERT_WEBHOOK 未配置，仅记录告警内容（不转发）:\n%s", markdown)
        return {"status": "logged", "forwarded": False}

    body = to_robot_payload(markdown)             # 按渠道切企微/钉钉的包体格式
    try:
        async with httpx.AsyncClient(timeout=10) as client:
            resp = await client.post(ALERT_WEBHOOK, json=body)
        return {"status": "forwarded", "forwarded": True, "upstream_status": resp.status_code}
    except Exception as e:
        # 下游发送失败也绝不抛异常，只记日志
        logger.error("转发告警到 %s 失败: %s", ALERT_WEBHOOK, e)
        return {"status": "error", "forwarded": False, "error": str(e)}
```

告警规则覆盖了服务存活、探针失败、主机 CPU/内存/磁盘、JVM 堆、HTTP 5xx 错误率，还有一条业务的——审核积压超过 50 条就报。

资源这块也得管，全套东西挤在一台 4 核 8G 的机器上，容器不限内存会跟宿主机上 systemd 跑的 Java 后端互相挤爆。我用一个 override 文件给每个容器单独设了内存上限，加起来压到 5.75G 左右，给宿主机的 Java 和系统留出余量。用 override 分层而不是直接改主 compose，是想让主文件保持环境无关，资源约束当成可叠加的一层。

## 数据管道

Python 写的 ETL，约 660 行，只依赖 pymysql 和 requests 两个第三方包。它把校园墙的帖子评论、和微信群导出，清洗、脱敏、假名化之后收敛成"问题—参考答案"的知识单元，推给 AI 服务建图谱。

脱敏我是当成"地基"来做的——放在最前面强制执行，而且整个管道默认是 dry-run、只写本地文件，必须显式加 `--push` 才真的写库。这个默认值是故意的：把"手抖跑错"的后果从"泄露"降级成"只是落了个本地文件"。

```python
# ingest.py
def _push(units, url, batch_size) -> int:
    import requests  # 延迟导入，dry-run 根本不需要网络依赖
    endpoint = url.rstrip("/") + "/index"
    batch, total = [], 0
    for ku in units:
        batch.append(ku.to_index_doc())
        if len(batch) >= batch_size:
            total += _post_batch(requests, endpoint, batch)
            batch = []
    if batch:
        total += _post_batch(requests, endpoint, batch)
    return total

# main()
    if args.push:
        total = _push(units, url, args.batch_size)
    else:
        n = _emit_jsonl(units, args.out)
        print(f"[dry-run] 已写出 {n} 个脱敏知识单元 → {args.out}")
        print("         请人工抽检脱敏质量，确认后加 --push 灌入知识图谱")
```

脱敏覆盖了 11 类 PII，处理里有几个细节是我从真实数据里发现问题才补的。比如微信的 @ 提及，显示名内部可能夹着普通空格甚至手机号，常规正则吃到空格就停会把后面的姓名电话漏成正文，所以得一直吃到那个特殊分隔符为止：

```python
# desensitize.py
# 微信真实 @提及一律以特殊分隔符终止；显示名内部可能含普通空格甚至手机号
# （如 "@2xxx 同学A"、"@2xxx 同学B<phone>"），所以必须吃到分隔符为止，
# 否则空格后的姓名/电话会漏成正文。@所有人/@全体成员 这种公告语义保留。
_MENTION = re.compile(r"@([^@\n ]+)(?: |$)")
_MENTION_KEEP = {"所有人", "全体成员"}

# 顺序敏感：先匹配长的、特异的，避免被宽泛规则先吃掉
_PATTERNS: list[tuple[str, re.Pattern]] = [
    ("idcard",   re.compile(r"(?<!\d)\d{17}[\dXx](?!\d)")),       # 身份证放手机号/银行卡前面
    ("bankcard", re.compile(r"(?<!\d)\d{16,19}(?!\d)")),
    ("email",    re.compile(r"[A-Za-z0-9._%+\-]+@[A-Za-z0-9.\-]+\.[A-Za-z]{2,}")),
    ("phone",    re.compile(r"(?<!\d)1[3-9]\d{9}(?!\d)")),
    # ... QQ / 微信号 略 ...
    ("studentid",re.compile(r"(?:学号|学籍号|考生号)[:：\s]*\d{8,14}")),  # 强制前缀，免得误伤年份/金额
    ("dorm",     re.compile(r"[东西南北A-Za-z0-9一二三四五六七八九十]{1,4}[栋幢座号][\s\-]*\d{2,4}[室房]?")),
]
```

类似的细节还有：花名册替换要长名字优先，不然"张三丰"会被"张三"先截断；学号正则强制要前缀，不然会误伤年份、金额这些长数字；发言人假名化是会话内独立计数的（楼主、同学 A、同学 B……），跨会话不可关联，宁可牺牲跨会话的关联能力，换"不能被拼图还原个人轨迹"。知识单元的 ID 是按来源做哈希而不是按内容，这样同一篇帖子反复跑管道是覆盖而不是产生重复文档。最后一次真实导出是 248 个知识单元，人工抽检手机号、wxid、抽样姓名的残留都是 0。
