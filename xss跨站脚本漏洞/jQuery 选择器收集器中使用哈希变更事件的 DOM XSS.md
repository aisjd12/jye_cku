知识点：
1. 什么是"哈希变更事件"（hashchange）？

- `hashchange` 是浏览器在 URL **锚点（# 后面部分）变化**时触发的事件，常用于单页应用做"伪路由"。
- 攻击者可以构造 `#<payload>` 的 URL 发给受害者，受害者页面加载后**无需点击任何东西**，只要 hash 变化就会自动执行 handler。

2. jQuery 选择器当"汇入"的危险

- jQuery 的选择器参数（`$()` 函数）如果由攻击者控制，存在特殊规则：**当参数以 `#` 开头且包含 HTML 字符串时，jQuery 会把它当作 HTML 解析**。
- 于是 `location.hash` 作为源 → `$(location.hash)` 作为汇入，就能把 hash 里的标签直接解析成 DOM。

3. 为什么用 iframe 触发

- 经典利用：页面有一个 `window.onhashchange` → `$(location.hash)` 的 handler。
- 攻击者构造 `<iframe src="目标URL#<img src=x onerror=alert(1)>">`，嵌入恶意网页。当 iframe 加载、hash 变化时，handler 执行 `$(...)` 解析 hash → 事件属性触发脚本。

4. 如何防御？

- 禁止把不可信数据传入 `$()`、`$()` 选择器只接受硬编码选择器。
- 用 `.text()` 而不是 `.html()` 渲染数据。
- 校验 hash 内容白名单，拒绝含 `<`、`>` 的输入。

通关过程：
1：打开靶场首页，页面有"购物车"功能，触发一个 `location.hash` 相关 handler
2：在 URL 后加锚点 payload：`#<img src=x onerror=alert(1)>`，页面直接弹出 alert → 完成
3：（若需演示利用链）在攻击者服务器/本地放一个页面，用 iframe 加载：
   `<iframe src="https://目标靶场/#<img src=x onerror=alert(1)>"></iframe>`
   受害者打开该页面即被触发
