## Context

`OCodeController` 鸿蒙端(ArkTS)的 `ChatPage` 当前实现里,`loadHistory` 直接 `GET /session/{id}/message` 不带 `limit`,后端返回该会话的**全部**消息,前端通过 `toDisplayMessages` 全部塞进 `@State messages`。当一个会话跑出 50+ 轮后,出现以下问题:

1. 首次进入 `ChatPage` 时一次性把整段历史插入 `messages`,ArkUI 的 diff 成本与首次布局时间随消息数线性增长。
2. 内存里常驻整段消息对象(含 parts、reasoning、tool 状态),冷启动多个会话时内存压力明显。
3. 长会话上下滚动容易丢帧。

`ChatViewModel.loadHistoryPage` 与 `ChatPage.loadPreviousMessages` 已实现"分页+上滑加载"骨架,但存在以下缺口:
- 滚动位置保持:在 `messages` 头部插入新项后,Scroller 仍停留在原 yOffset,导致用户实际看到的内容被"新插入的消息"顶到视口下方,出现"加载老消息瞬间视口跳变"的体验问题。
- 边界态:当 `cursor.previous` 为空时,没有"已到最早消息"的常驻提示。
- 失败态:首次失败时仅回退到缓存,UI 缺一个明确的"无法连接,已显示本地缓存"提示。
- 重复触发:`onReachStart` 没有去重,快速连续触顶会产生并发请求。
- 增量刷新:发消息后回流逻辑已有但 `lastMsgCursor` 的初始值依赖 `loadHistoryPage` 返回的 `cursor.next`,在非 `latest` 模式下未初始化。

`OpenCodeCore.cacheMessages` / `getCachedMessages` 仍作为离线兜底来源,本变更不修改其后端协议。

## Goals / Non-Goals

**Goals:**
- 首屏仅拉取最近 k 条消息(默认 20),可交互时间与消息总量解耦。
- 上滑到顶时分页追加更老消息,滚动位置不跳变。
- 到达历史尽头有明确边界提示,且不再发请求。
- 首屏失败时给出"本地缓存/无历史"两种状态的明确提示。
- 复用现有 `OpenCodeApiClient` / `OpenCodeCore` / `loadHistoryPage` 能力,**不引入新依赖、不改后端协议**。

**Non-Goals:**
- 不实现下拉刷新(从上向下拉)加载"更新的消息"——opencode 后端在分页维度没有"未来游标"概念,且本场景下 SSE 已承担实时推送。
- 不实现消息搜索、消息收藏、消息删除等业务能力。
- 不重构 `OpenCodeApiClient` 的整体结构,只在 `ChatViewModel` 内部加参数(`direction`)以复用 `loadHistoryPage`。
- 不引入虚拟滚动 / `LazyForEach` 改造(超出本次范围,留作后续优化)。

## Decisions

### 1. 复用 `loadHistoryPage`,不新增 API
**决定**:`ChatViewModel.loadHistoryPage` 已经接受 `cursor` / `limit` / `order` 参数,继续复用;在调用方按需传入 `order=desc` 拉取"更老"和 `order=asc` 拉取"更新"。
**理由**:opencode 后端的 `cursor` 是 base64-url 编码的复合键,同一接口配合 `order` 参数即可实现双向翻页,新增 API 会带来协议分裂风险。
**备选**:
- 新增 `loadNewerPage` / `loadOlderPage` 两个方法:被否决,会把简单分页拆成两份近似代码,后续 `cursor` 编码变动需要双改。
- 引入无限滚动库(如 `IntersectionObserver`):ArkTS 没有对应原生能力,自造轮子成本高且 ArkUI 已提供 `onReachStart`,够用。

### 2. 滚动位置保持:在插入前后双次读 `yOffset`
**决定**:触发 `onReachStart` 加载更老消息时,先在请求**前**记录 `this.scroller.currentOffset().yOffset` 记为 `prevY`;请求成功、插入新消息到 `messages` 头部后,记录此时 `yOffset` 记为 `postY`,计算 `delta = postY - prevY`,再调用 `scroller.scrollTo({ xOffset: 0, yOffset: prevY + delta + (新增消息估算高度) })` 还原。
**理由**:ArkUI 的 `Scroller` 在内容总高度变化后会自动保持"原视口顶部的元素仍在视口顶部",但只有当插入位置在**当前可见区域上方**时才有效;此处新消息确实插在视口上方,因此 `delta` 即可视为新增消息总高度。
**简化方案**(最终采用):由于新增消息**确定**插在视口上方且 `Scroller` 本身在 List 长度变化时会保留锚定元素,可以不显式 `scrollTo`,只确保:
1. 插入前 `isLoadingHistory=true` 抑制重复触发;
2. 插入后 `setTimeout(() => this.scroller.scrollTo({ xOffset: 0, yOffset: prevY }), 0)` 做一次补偿。

**备选**:
- 用 `LazyForEach` + `cachedCount`:ArkTS 支持但要重写消息渲染,改动面大,本变更暂不做。
- 强制滚动到原 yOffset 而不读 delta:会出现短暂闪烁,体验差。

### 3. 边界判定:`cursor.previous` 为空即终止
**决定**:`loadHistoryPage` 返回的 `cursor.previous` 为空字符串或缺失时,设置 `historyCursor = ''`,并在 `Scroll` 内容顶部用 `@State reachedEnd: boolean = true` 常驻显示"已到最早消息"。
**理由**:opencode 后端对翻页边界的语义就是 `cursor.previous` 为空,与现有 `loadPreviousMessages` 行为一致。
**备选**:用 `total` 字段做边界判断——当前 `OpenCodeApiClient.getMessages` 没解析 `total`,改造意义不大。

### 4. 失败态:缓存兜底 + UI 显式提示
**决定**:`loadHistory` 失败时,先尝试 `core.getCachedMessages(realSessionId)`,并设 `loadError: 'offline' | 'empty' | null` 三态:
- `offline`:有缓存,显示"无法连接,已显示本地缓存";
- `empty`:无缓存,显示"暂无历史消息";
- `null`:成功,不显示错误条。
**理由**:用户需要明确知道"现在看到的是不是最新",含糊处理会引发误解。

### 5. 增量刷新保留现有 `lastMsgCursor` 逻辑
**决定**:`sendMessage` 完成后,继续以 `lastMsgCursor` 反转后的 cursor + `order=asc` 拉增量,与 `loadPreviousMessages` 共用 `loadHistoryPage`。
**理由**:该路径已经走通,仅需要在 `loadHistory` 的 `latest` 分支初始化 `lastMsgCursor`(已用 `invertCursorForAfter(page.cursor.next)` 初始化)。
**注意**:`getMessageLoadMode() === 'latest'` 是已存在的 `OpenCodeCore` 模式开关,本变更强制走 `latest` 分支,旧的 `else` 分支保留作为兜底。

## Risks / Trade-offs

- [R1: `scroller.scrollTo` 偶发失效] → 改用 `requestAnimationFrame`(ArkTS 通过 `setTimeout(_, 0)` 近似)在下一帧再 `scrollTo`,并在调试期打日志。
- [R2: `onReachStart` 在 iOS / HarmonyOS 上的触发时机差异] → 在 `Scroll` 上同时监听 `onScrollFrameBegin` 作为兜底,只有当 `currentOffset().yOffset <= 4` 才允许触发,避免误触。
- [R3: `loadHistoryPage` 返回 `cursor` 字段缺失] → 当前实现已用 `resp.cursor || {}` 兜底,继续保留。
- [R4: 并发请求] → `isLoadingHistory` 互斥锁,新请求前 `cancelRequest()`(已有)。
- [R5: 缓存膨胀] → `OpenCodeCore` 现有缓存策略(整段覆盖 `cacheMessages`)在分页场景下可能丢失更老消息;本期不优化,标注为后续工作。

## Migration Plan

1. 提交 `ChatViewModel` 与 `ChatPage` 改动到默认分支,功能开关先以常量 `const PAGINATION_ENABLED = true` 控制,便于快速回滚。
2. 在 AboutPage 增加 "消息分页" 开关(`StorageLink`),允许老用户一键关闭回到旧版行为。
3. 验证流程:新建会话跑 50 轮,杀掉应用冷启动,确认首屏只拉 20 条、视口停在底部;上滑到顶触发分页,确认滚动不跳变、视口稳定;断网测试缓存兜底提示。
4. 回滚:关闭 `PAGINATION_ENABLED` 即可退回旧版 `loadHistory`(整段拉取),不涉及数据迁移。

## Open Questions

- 是否需要在 ChatPage 顶部加一个 "回到最新" 浮动按钮?用户向下滚动较深时,直接跳到底部的能力可能有用。本期不做,留作后续评估。
- `OpenCodeCore.cacheMessages` 整段覆盖的策略是否需要改为"按段合并"?目前分页追加更老消息后会整段覆盖缓存,理论上不会丢数据(因为旧消息已包含在新请求的 `rawMessages` 里),但需要验证。
