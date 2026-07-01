## ADDED Requirements

### Requirement: 进入会话只加载最近 k 条消息
首次进入聊天会话时,前端 MUST 仅请求并渲染最近 k 条消息(默认 k=20),而非拉取该会话的全部历史。

#### Scenario: 首次进入一个已有 50 条消息的会话
- **WHEN** 用户从会话列表进入 ChatPage 且 `realSessionId` 已存在
- **THEN** 前端以 `limit=20&order=desc&cursor=` 请求 `GET /session/{id}/message`
- **AND** `messages` 数组初始仅包含返回的 20 条消息,按时间正序展示
- **AND** 视口自动滚动到列表底部(最新消息)

#### Scenario: 首次进入一个新会话(后端尚无消息)
- **WHEN** 用户从会话列表进入 ChatPage 且该会话没有 `realSessionId` 或尚无历史
- **THEN** 不发起消息分页请求,`messages` 保持空数组

### Requirement: 上滑到顶部自动加载更老消息
当用户向上滚动到消息列表顶端时,前端 MUST 使用上一页返回的 `cursor.previous` 作为游标请求更早的消息,并把新消息插入到当前列表**头部**。

#### Scenario: 上滑到顶部且仍有更老消息
- **WHEN** 用户在 ChatPage 中将消息列表滚到顶端(`onReachStart` 触发)且 `historyCursor` 非空
- **THEN** 前端以 `limit=20&order=desc&cursor={historyCursor}` 请求下一页
- **AND** 请求成功后,新消息按时间正序插入到 `messages` 头部
- **AND** 视口位置保持(插入前记录的 `currentOffset` 在插入后被还原),不出现"瞬间跳到底部"的现象
- **AND** `historyCursor` 更新为本次响应的 `cursor.previous`

#### Scenario: 加载过程中重复触发
- **WHEN** 在上一次"加载更老消息"请求尚未完成时,用户再次上滑到顶部
- **THEN** 忽略后续触发(以 `isLoadingHistory` 标志去重),不发起并发请求

#### Scenario: 加载更老消息失败
- **WHEN** 分页请求因网络或非 2xx 失败
- **THEN** 顶部展示"加载失败,点击重试"提示,`historyCursor` 保持不变,可由用户重试

### Requirement: 到达历史尽头时给出边界提示
当上一页响应中的 `cursor.previous` 为空时,前端 MUST 判定为已到最早消息并停止后续请求,同时在列表顶部展示"已到最早消息"提示。

#### Scenario: 拉取到最旧一页
- **WHEN** 分页请求返回的 `cursor.previous` 为空字符串或缺失
- **THEN** `historyCursor` 置为空,后续 `onReachStart` 不再发起请求
- **AND** 列表顶部常驻显示"已到最早消息"文案,样式与现有"没有更多消息"一致

### Requirement: 滚动位置保持
在分页追加更老消息时,前端 MUST 保持用户的当前可视区域不发生跳变。

#### Scenario: 在列表中部时加载更老消息
- **WHEN** 用户停留在第 30~40 条之间并触发上滑加载
- **THEN** 在请求发起前记录当前 `scroller.currentOffset().yOffset`
- **AND** 新消息插入头部后,新的 `currentOffset` 至少为 `原值 + 新增消息总高度`
- **THEN** 调用 `scroller.scrollTo({ xOffset: 0, yOffset: 原值 + 新增高度 })` 还原视口

#### Scenario: 新增消息高度为 0 的边界情况
- **WHEN** 本次分页接口返回空数组
- **THEN** 不更新 `messages`,不调整滚动位置,直接进入"已到最早消息"判定

### Requirement: 缓存兜底
当分页请求失败(网络异常或非 200),前端 MUST 回退到 `OpenCodeCore.getCachedMessages(sessionId)` 渲染本地缓存。

#### Scenario: 首屏请求失败但本地有缓存
- **WHEN** 首次进入会话的 `loadHistory` 因网络异常失败
- **THEN** 从 `OpenCodeCore` 读取该 `realSessionId` 的本地缓存并渲染
- **AND** `historyCursor` 置空,顶部展示"无法连接,已显示本地缓存"

#### Scenario: 首屏失败且本地无缓存
- **WHEN** `loadHistory` 失败且本地无任何缓存
- **THEN** `messages` 保持空数组,顶部展示"暂无历史消息"

### Requirement: 发送新消息后的增量刷新
发送完一轮对话后,前端 MUST 仅拉取自上次 `lastCachedMsgTime` 以来新增的消息,避免重拉整段历史。

#### Scenario: 发送消息后回流
- **WHEN** `sendMessage` 收到 200 响应且 `beforeCursor` 存在
- **THEN** 以 `cursor=beforeCursor&order=asc&limit=50` 拉取增量
- **AND** 仅追加 `timestamp > lastCachedMsgTime` 的消息到 `messages` 末尾
- **AND** 更新 `lastMsgCursor` 为新响应反转后的 `cursor.next`

#### Scenario: 首条消息(无 beforeCursor)
- **WHEN** 该会话此前从未加载过(`lastMsgCursor` 为空)
- **THEN** 跳过增量拉取,直接把响应中的 `response` 渲染为最新一条 assistant 消息
