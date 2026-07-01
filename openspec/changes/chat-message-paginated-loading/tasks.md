## 1. ChatViewModel 适配

- [ ] 1.1 在 `ChatViewModel` 新增常量 `DEFAULT_PAGE_SIZE = 20`,并为 `loadHistoryPage` 增加可选参数 `direction: 'older' | 'newer' = 'older'`(内部映射到 `order`)。
- [ ] 1.2 校验 `loadHistoryPage` 的响应解析:同时兼容 `Array` 与 `{ items, cursor }` 两种返回结构,并在 `cursor` 缺失时降级为 `{}`(已实现,确认即可)。
- [ ] 1.3 抽出 `buildPreviousCursor(items, lastItem)` 辅助方法,从已知消息尾部推断 `cursor.previous`,供缓存兜底路径使用。

## 2. ChatPage 状态扩展

- [ ] 2.1 在 `ChatPage` 增加 `@State historyCursor: string = ''`、`@State isLoadingHistory: boolean = false`、`@State reachedEnd: boolean = false`、`@State loadError: 'offline' | 'empty' | null = null`。
- [ ] 2.2 引入 `PAGINATION_ENABLED` 常量(默认 `true`)与 `@StorageLink('paginationEnabled')`,在 AboutPage 中提供开关。
- [ ] 2.3 把现有 `historyCursor` 状态从`保留但初始化为空,改为由 `loadHistory` 的 `latest` 分支写入 `page.cursor.previous || ''`。

## 3. 首次加载逻辑

- [ ] 3.1 `loadHistory` 在 `PAGINATION_ENABLED` 下走 `loadHistoryPage(... limit=20, order='desc')`,仅用返回值初始化 `messages`、`historyCursor`、`reachedEnd`、`loadError`。
- [ ] 3.2 当 `page.items.length === 0` 时,把 `reachedEnd = true` 并把 `loadError` 置为 `null`(空会话,不视为错误)。
- [ ] 3.3 失败分支:有缓存则 `loadError='offline'` + 渲染缓存;无缓存则 `loadError='empty'` + `messages=[]`。

## 4. 上滑加载更老消息

- [ ] 4.1 在 `Scroll` 上保留 `onReachStart`,绑定 `loadPreviousMessages`,并加 `onScrollFrameBegin` 兜底,当 `yOffset <= 4` 时也允许触发。
- [ ] 4.2 `loadPreviousMessages` 在请求**前**记录 `this.scroller.currentOffset().yOffset` 为 `prevY`,在 `isLoadingHistory = true` 后再发请求,避免并发。
- [ ] 4.3 请求成功后,把新消息去重后插入到 `messages` 头部,并 `setTimeout(() => this.scroller.scrollTo({ xOffset: 0, yOffset: prevY }), 0)` 还原视口。
- [ ] 4.4 当响应 `cursor.previous` 为空时,设置 `historyCursor=''`、`reachedEnd=true`、不再发起后续请求;若 `items.length === 0` 也按到达边界处理。

## 5. 边界与错误态 UI

- [ ] 5.1 在 `Scroll` 内部 Column 顶部增加"加载历史消息..."(`isLoadingHistory` 为 true 时显示)与"已到最早消息"(`reachedEnd && !isLoadingHistory` 时显示)两个互斥提示,样式与现有"没有更多消息"保持一致。
- [ ] 5.2 增加"加载失败,点击重试"提示,绑定到 `loadError === 'offline' || (loadHistory 失败且未到尽头)` 状态,点击重新调用 `loadPreviousMessages`。
- [ ] 5.3 顶部错误条与"已到最早消息"互斥,二者在同一渲染分支里用 `if/else if`。

## 6. 增量刷新与现有 sendMessage 协同

- [ ] 6.1 校验 `sendMessage` 中 `beforeCursor` 拉增量的路径仍工作:在 `PAGINATION_ENABLED` 下,`lastMsgCursor` 已被 `loadHistory` 初始化,无需改动。
- [ ] 6.2 当增量请求超时/失败时,降级为直接把 `response` 追加为最新消息,避免"发完消息看不到回复"。
- [ ] 6.3 校验 SSE `message.updated` / `text.delta` 事件在分页场景下仍正确更新 `streamingMessageId`,不在 `loadPreviousMessages` 完成后被回退。

## 7. 验证

- [ ] 7.1 新建一个会话跑 50 轮,杀掉应用冷启动进入,确认首屏只拉 20 条、视口在底部、上滑到顶触发分页后滚动不跳变。
- [ ] 7.2 制造断网:首屏失败应显示"无法连接,已显示本地缓存"或"暂无历史消息"中的一种。
- [ ] 7.3 在 AboutPage 关闭分页开关,确认回到旧版整段拉取行为。
- [ ] 7.4 跑现有 ArkTS 编译(`hvigorw assembleHap` 或项目内 build 脚本),确保 0 error / 0 warning(尤其留意 `arkts-no-untyped-obj-literals`)。
