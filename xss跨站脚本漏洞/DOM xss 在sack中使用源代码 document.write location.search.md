知识点：
1. 什么是 DOM 型 XSS？

- **本质**：前端 JavaScript 直接读取 URL/请求中的参数作为"源（Source）"，再用 `document.write`、`innerHTML` 等"汇入（Sink）"把它写进 DOM。整个过程发生在**客户端浏览器**，服务端 HTML 完全正常，因此传统 WAF 和服务端过滤都拦不到。
- **与反射型/存储型区别**：反射型和存储型是"服务端把输入拼进 HTML"；DOM 型是"JS 在浏览器里把输入拼进页面"。

2. 这条链路的两个关键概念

- **源（Source）**：攻击者能控制的数据入口，如 `location.search`（URL 查询串）、`location.hash`（URL 锚点）、`document.referrer`。
- **汇入（Sink）**：把数据写进页面导致脚本执行的危险函数，如 `document.write`、`innerHTML`、`eval`、`jQuery.html()`。

3. 为什么 `document.write` 是危险汇入？

- `document.write` 会把参数内容**作为原始 HTML 解析**，不是当作文本。传入 `<img src=1 onerror=alert(1)>` 就会直接插入元素并触发 `onerror`。
- 若页面在搜索框里用 `document.write('搜索: ' + searchTerm)`，那么 `searchTerm` 完全可控 → 直接注入标签。

4. 攻击者在无防护场景的标准玩法

- 构造恶意 URL 让受害者点击（配合钓鱼/诱骗）。
- 恶意脚本窃取 Cookie（需非 HttpOnly）、劫持会话、改页面内容、盗取表单输入。

5. 如何防御？

- 不使用 `document.write`/`innerHTML` 拼接不可信数据，改用 `textContent`、`innerText` 或安全 DOM API。
- 对源数据做白名单校验（如仅允许字母数字）。
- 部署 CSP（Content-Security-Policy）作为纵深防御。

通关过程：
1：打开靶场首页，页面有搜索框，输入任意内容提交
2：提交后 URL 变为 `/?search=<输入内容>`，发现查询串参数 `search`
3：用 Burp 或直接改 URL 传 payload：`/?search="><svg onload=alert(1)>`
4：页面刷新后弹出 alert 窗口 → 完成（即可提交答题）
