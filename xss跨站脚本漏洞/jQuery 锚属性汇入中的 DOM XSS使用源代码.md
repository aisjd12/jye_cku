知识点：
1. "源和汇入都在 jQuery"指的是什么？

- 这是一个 **DOM 型 XSS**：源是 `location.search`（URL 查询串），汇入是 jQuery 的属性写入函数 `$('a').attr('href', ...)`。
- 页面 JS 读取 `location.search` 里的参数，写进某个 `<a>` 标签的 `href` 属性，**没有做协议校验**。

2. 为什么 `href` 可控很危险

- `href` 支持 `javascript:` 伪协议。如果把 `href` 设为 `javascript:alert(1)`，用户点击该链接时就会执行脚本。
- 所以无需闭合标签、无需事件属性，只需构造 URL 参数让生成的 `href` 变成 `javascript:alert(1)` 即可。

3. 攻击者怎么"骗"受害者执行

- 构造 URL：`/?<参数>=javascript:alert(1)`（具体参数名需看源码，常见是 `url` 或 `returnUrl`）。
- 受害者打开该 URL 后，页面里那个链接的 `href` 已被污染，一旦点击就弹窗/执行脚本。
- 社交工程配合：把恶意链接发出去，诱导受害者点击页面内的"返回"链接。

4. 如何防御？

- 对 `href`/`src` 等协议相关属性做 **协议白名单**（仅允许 `http:`/`https:`/`mailto:`），拒绝 `javascript:`、`data:`、`vbscript:`。
- 使用 `URL` 对象解析校验后再赋值，或设置 `CSP: script-src` 阻止内联 JS。
- 不直接 `$('a').attr('href', 用户输入)`，先 sanitize。

通关过程：
1：打开靶场首页，查看页面 JS，找到 `location.search` 读取参数并写入 `$('a').attr('href', ...)` 的代码，确认参数名（如 `returnUrl` 或 `url`）
2：构造 URL：`/?returnUrl=javascript:alert(1)`（以实际参数名为准）
3：打开页面后点击被污染的"返回"链接 → 弹出 alert → 完成
