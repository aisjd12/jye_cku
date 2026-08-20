知识点：
1. 什么是反射型（Reflected）XSS？

- **本质**：服务端把 URL 参数（或表单数据）直接拼进 HTML 响应**返回给浏览器**，payload 被"原样反射"回页面。由于服务端**未做任何编码**，输入被当作 HTML 解析执行。
- **触发条件**：受害者必须点击攻击者构造的恶意 URL（`/?search=<payload>`），因此常搭配钓鱼链接/短链投放。

2. 反射型 XSS 的完整利用链

- 攻击者构造恶意 URL → 投递给受害者。
- 受害者点击 → 请求发出 → 服务端反射 payload → 浏览器执行脚本。
- 脚本把 Cookie（非 HttpOnly）发到攻击者服务器，或直接篡改页面诱导转账/登录钓鱼。

3. 为什么"没有编码"是漏洞根因

- 服务端应把用户输入当**纯文本**输出（HTML 实体编码 `<`→`&lt;` 等），但这里直接拼接字符串。
- 拼进 HTML 上下文时，`<svg onload=alert(1)>`、`<img src=1 onerror=alert(1)>` 都会被解析成真实元素。

4. 如何防御？

- 输出编码（context-aware encoding）：HTML 上下文用 HTML 实体转义。
- 输入校验白名单 + 限制长度、字符集。
- CSP：`script-src 'nonce-xxx'` 阻止内联脚本。
- 用框架默认转义（React/Vue/Angular）替代字符串拼接 innerHTML。

通关过程：
1：打开靶场首页，页面有搜索框，输入任意内容后提交
2：URL 变为 `/?search=<输入>`，且输入内容**未编码直接反射**回页面
3：构造 payload：`/?search=<script>alert(1)</script>`
4：页面弹出 alert → 完成
