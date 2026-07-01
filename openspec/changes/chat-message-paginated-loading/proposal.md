## Why

长会话(50+ 轮)场景下,用户每次进入 ChatPage 都会一次性拉取该会话的全部消息并塞入 `messages` 数组,首屏渲染、内存占用、滚动体验都随历史长度线性恶化。本变更要落地"递进式数据拉取 + 历史回放":进入会话只展示最近 k 条消息,用户主动向上滑到顶端时再分页加载更老的消息,并在到达历史尽头时给出明确边界提示,从而把首屏开销与可交互时间控制在常数级别。

## What Changes

- **首次进入会话只拉取最近 k 条**(`limit=k, order=desc`),用返回的 `cursor.previous` 作为向上翻页的入口游标。
- **上滑到顶部时按游标分页拉取更老消息**,在已有列表**头部**插入,并保持当前视口位置(不滚回底部)。
- **到达历史尽头**(`cursor.previous` 为空)时,顶部展示"已到最早消息"提示,不再发起请求。
- **本地缓存协同**:OpenCodeCore 的 `cacheMessages` 继续按会话维度缓存,用于离线/失败回退,不替代分页。
- **发送新消息后增量刷新**:复用现有 `lastMsgCursor` 配合 `asc` 拉取新增消息,避免每次发完消息都重拉整段历史。
- **不改动后端协议**:opencode web 的 `GET /session/{id}/message?limit=&order=&cursor=` 接口已支持分页,本变更只调整前端调用与状态管理。

## Capabilities

### New Capabilities

- `chat-message-pagination`: 聊天消息的递进式分页加载与历史回放能力,覆盖"首屏拉取最近 k 条""上滑加载更老""边界态""滚动位置保持""缓存兜底"等行为契约。

### Modified Capabilities

- 无(现有 `keyboard-avoid-header`、`screen-rotation-lock` 等 spec 仅约束独立模块,不涉及消息加载契约,故不需要 delta spec)。

## Impact

- **代码**:
  - `entry/src/main/ets/viewmodel/ChatViewModel.ts` — `loadHistoryPage` 已基本可用,需补齐参数(可选 `before` / `after` 方向)与对 `cursor.previous` / `cursor.next` 的统一解析。
  - `entry/src/main/ets/pages/ChatPage.ets` — `loadHistory`、`loadPreviousMessages`、`build()` 中的 `Scroll` 组件:接入首次拉取分支、上滑 `onReachStart` 触发的分页追加、滚动位置保持(记录插入前的 `currentOffset`,插入后 `scrollTo` 回去)、"已到最早消息"提示渲染。
  - `entry/src/main/ets/core/OpenCodeCore.ts` — `cacheMessages` / `getCachedMessages` 现有 API 保持不变,作为兜底来源。
- **依赖**:不引入新依赖,继续使用 `@ohos.net.http` 调用 opencode web。
- **API 契约**:opencode web 的 `GET /session/{id}/message` 需支持 `limit`、`order`、`cursor` 三个 query 参数(已支持,见 `OpenCodeApiClient.getMessages` 与 `loadHistoryPage` 的 URL 构造)。
- **UI 体验**:首屏速度提升、内存占用降低;长会话滚动期间不再因一次性插入大量节点而卡顿。
