知识点：
1. 漏洞核心：Token 绑定到非会话 Cookie，而非 session

- 请求中有三组关键数据：

| 位置 | 名称 | 作用 |
|------|------|------|
| Cookie | csrfKey | Token 的"钥匙" |
| Cookie | session | 用户会话 |
| Body | csrf | CSRF Token |

- csrf Token 绑定的是 csrfKey，**不是** session。
- csrfKey 和 csrf 必须来自同一账号、同一次请求。
- 但 csrfKey 不绑定 session → 攻击者的 csrfKey 可以配合 victim 的 session。

```
csrfKey = 攻击者的 ✓
csrf    = 攻击者的 ✓
session = victim 的 ✓
→ 服务端校验：csrfKey 和 csrf 匹配 ✓，不校验 session 绑定 ✓
```

2. 为什么不能直接用 Exploit Server 写 Cookie

- Exploit Server 域名 ≠ 靶场域名。
- 在 Exploit Server 的 Head 中写 Set-Cookie，Cookie 只属于 Exploit Server 域名。
- 浏览器访问靶场时不会携带其他域名的 Cookie。
- 必须让 Cookie 写在**靶场自己的域名下**。

3. CRLF 注入写 Cookie

- 靶场搜索功能把搜索词反射到响应头的 Set-Cookie 中：
  - 请求 `/?search=test` → 响应头 `Set-Cookie: search=test`
- 如果搜索词中插入 CRLF（换行符），可以注入额外的 HTTP 头：

```
/?search=test%0d%0aSet-Cookie:%20csrfKey=你的值
```

- 靶场返回：

```
Set-Cookie: search=test
Set-Cookie: csrfKey=你的值    ← 注入的
```

- 这个请求发给靶场域名，Cookie 写在靶场域名下 ✓

4. CRLF 注入 vs 反射型 XSS

| | CRLF 注入 | 反射型 XSS |
|---|---|---|
| 注入位置 | HTTP 响应头 | HTTP 响应体（HTML） |
| 注入符号 | %0d%0a（换行） | `<script>`（HTML 标签） |
| 效果 | 伪造 HTTP 头（写 Cookie） | 在页面中执行 JS |

5. img onerror 的时序设计

```html
<img src="靶场搜索接口+CRLF注入" onerror="document.forms[0].submit()">
```

流程：

```
页面加载
→ img 请求靶场搜索接口（带 CRLF 注入）
→ 靶场返回 Set-Cookie 写入 csrfKey
→ img 加载失败（搜索结果不是图片）
→ 触发 onerror
→ onerror 执行 document.forms[0].submit()
→ 表单提交时携带 csrfKey + victim session
→ 表单参数包含 csrf + 新邮箱
→ 服务端校验通过 → victim 邮箱被修改
```

- 不能用 `<script>submit()</script>` 直接提交：Cookie 还没写入，csrf 和 csrfKey 对不上。
- img onerror 保证了"先写 Cookie → 再提交表单"的时序。

6. URL 编码对照

| 字符 | 编码 |
|------|------|
| CR（回车） | %0d |
| LF（换行） | %0a |
| 空格 | %20 |
| 分号 ; | %3b |

7. 防御方案

- csrfKey 绑定到 session：校验 csrfKey 是否属于当前会话。
- 禁止搜索词反射到 Set-Cookie 头：对搜索词做 URL 编码或移除 CRLF。
- 使用 SameSite=Strict 的 session Cookie。

通关过程：
1：登录 A 账号，抓修改邮箱请求，记录 csrfKey（Cookie 中）和 csrf（Body 中）两个值
2：登录 B 账号，抓修改邮箱请求，保持 Cookie 不动，把 csrfKey 和 csrf 都替换成 A 的 → 成功 → 确认 Token 绑定 csrfKey 不绑定 session
3：验证 CRLF 注入：在浏览器中访问 `https://靶场域名/?search=test%0d%0aSet-Cookie:%20csrfKey=mytest123`
4：F12 → Application → Cookies → 确认靶场域名下出现了 csrfKey=mytest123
5：在 Exploit Server 构造 PoC：
6：Head 填 `Content-Type: text/html; charset=utf-8`
7：Body 填表单（action 指向靶场接口，method POST）+ img 标签（src 指向靶场搜索接口带 CRLF 注入，onerror 触发表单提交）
8：csrf 值和 csrfKey 值都填 A 账号的
9：email 填测试邮箱（不是自己的）
10：Store → Deliver exploit to victim → victim 邮箱被修改 → 靶场解决

易错点：
- csrfKey 和 csrf 不来自同一次请求 → 对不上 → 失败。
- img src 用了 Exploit Server 域名而不是靶场域名 → Cookie 写在了错误域名下。
- action 或 src 中带了 Markdown 链接格式 `[https://...](https://...)` → 浏览器无法解析 URL。
- HTML 标签前带了反斜杠 `\<form>` → 解析异常。
- img 放在 form 里面 → onerror 时 form 可能还没完全解析。
- img src 中 csrfKey 值为空（只写了 `%3b` 没写值）→ Cookie 写了空值。
- 靶场重置后 csrfKey 和 csrf 重新生成 → 用旧值会失败。
- 以为 CRLF 注入就是 XSS → CRLF 篡改响应头，XSS 篡改响应体，完全不同。