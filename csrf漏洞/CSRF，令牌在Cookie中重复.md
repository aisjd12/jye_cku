知识点：
1. 漏洞核心：双提交 Cookie 模式的缺陷

- 请求中有两处 csrf：
  - Cookie 中的 csrf
  - Body 中的 csrf
- 服务端校验逻辑：只检查两处值是否相等，不校验值本身是否有效。

```
Cookie 中 csrf = X
Body   中 csrf = X
→ 两处相等 → 通过 ✓
```

- 这就是"双提交 Cookie"（Double Submit Cookie）模式的缺陷实现。
- 正确实现应该校验 Token 是否绑定会话，而不是只比较两处是否相等。

2. 为什么任意值都行

- 因为服务端只比较相等性，不看有效性。
- 攻击者可以自己定一个值（比如 fake 或 test123），同时写入 Cookie 和 Body。

| 正常请求 | 漏洞利用 |
|---------|---------|
| Cookie csrf = 服务端生成的值 | Cookie csrf = 攻击者定的任意值 |
| Body csrf = 同一个值 | Body csrf = 同一个值 |
| 两处相等 → 通过 | 两处相等 → 通过 |

3. 和上一关的区别

| | 令牌绑定在非会话 Cookie 上 | 令牌在 Cookie 中重复 |
|---|---|---|
| Cookie 中的参数 | csrfKey（必须是真实有效的） | csrf（任意值） |
| Body 中的参数 | csrf（必须是真实有效的） | csrf（同一个任意值） |
| 服务端校验 | csrfKey 和 csrf 是否匹配 | 两处 csrf 是否相等 |
| 攻击者需要 | 从自己账号拿真实 Token | 随便定一个值 |

- 上一关：值必须正确，只是不绑定 session。
- 这一关：值不需要正确，只要两边相等。

4. 利用方式：CRLF 注入写 Cookie

- 需要让 victim 浏览器的靶场域名下有一个你知道值的 csrf Cookie。
- 用靶场搜索功能的 CRLF 注入，注入 `Set-Cookie: csrf=你定的值`。
- 表单 Body 中放同一个值的 csrf。
- 表单提交时：Cookie 带 csrf + Body 带 csrf → 两边相等 → 通过。

5. PoC 结构

```html
<form action="靶场接口" method="POST">
  <input type="hidden" name="email" value="测试邮箱">
  <input type="hidden" name="csrf" value="你定的值">
</form>
<img src="靶场域名/?search=test%0d%0aSet-Cookie:%20csrf=你定的值%3b%20SameSite=None" onerror="document.forms[0].submit()">
```

- img 先请求搜索接口 → CRLF 注入写入 csrf Cookie
- img 加载失败 → onerror 触发表单提交
- 表单提交时两处 csrf 相等 → 服务端通过

6. 防御方案

- Token 绑定到会话：校验 csrf 是否属于当前 session，不能只比较两处相等。
- 使用服务端存储的 Token：登录时生成存入 session，请求时比对。
- 禁止搜索词反射到 Set-Cookie 头：过滤 CRLF 字符。

通关过程：
1：登录账号，抓修改邮箱请求，发现 Cookie 中有 csrf，Body 中也有 csrf，两处值相同
2：在 Repeater 中把两处 csrf 改成同一个随机值 → 发送 → 成功，确认服务端只校验相等性
3：在浏览器中验证 CRLF 注入：访问 `https://靶场域名/?search=test%0d%0aSet-Cookie:%20csrf=mytest123`
4：F12 → Application → Cookies → 确认靶场域名下出现 csrf=mytest123
5：在 Exploit Server 构造 PoC：
6：表单 action 指向靶场接口，method POST
7：表单中 csrf 填自定义值（如 fake），email 填测试邮箱
8：img src 指向靶场搜索接口 + CRLF 注入 `Set-Cookie: csrf=fake`
9：img onerror 触发 `document.forms[0].submit()`
10：Store → Deliver exploit to victim → victim 邮箱被修改 → 靶场解决

易错点：
- 以为需要拿真实的 csrf Token → 不需要，任意值只要两边相等就行。
- img src 用了 Exploit Server 域名 → Cookie 写在错误域名下，必须用靶场域名。
- action 或 src 中带了 Markdown 链接格式 `[url](url)` → 浏览器无法解析。
- HTML 标签前有 `\` 反斜杠 → 解析异常。
- img 放在 form 里面 → onerror 时 form 可能未完全解析。
- 用 script 直接 submit() → Cookie 还没写入，两处 csrf 对不上。