知识点：
1. 什么是 CRLF 注入

- CRLF = 回车换行符（`\r\n`，URL 编码 `%0d%0a`）。
- HTTP 协议中，头与头之间用 CRLF 分隔，头与正文之间用 CRLF CRLF 分隔。
- 如果用户输入被反射到 HTTP **响应头**中，且服务端没有过滤 CRLF 字符，攻击者可以通过插入换行符注入额外的 HTTP 头。

```
正常搜索词 = test
→ 响应头：Set-Cookie: search=test

注入搜索词 = test%0d%0aSet-Cookie: csrf=fake
→ 响应头：
  Set-Cookie: search=test
  Set-Cookie: csrf=fake    ← 注入的新头
```

2. CRLF 注入 vs 反射型 XSS

| | CRLF 注入 | 反射型 XSS |
|---|---|---|
| 注入位置 | HTTP **响应头** | HTTP **响应体**（HTML 正文） |
| 注入符号 | `%0d%0a`（换行符） | `<script>`（HTML 标签） |
| 执行方式 | 伪造 HTTP 头（写 Cookie、跳转等） | 在页面中执行 JavaScript |
| 判断标准 | 输入出现在 `<html>` 之前 → CRLF | 输入出现在 `<html>` 之内 → XSS |

3. CRLF 注入能做什么

| 注入的头 | 效果 |
|---------|------|
| `Set-Cookie: csrf=fake` | 在目标域名下写入 Cookie |
| `Location: https://evil.com` | 强制跳转到恶意网站 |
| `Content-Type: text/html` | 改变响应内容类型 |
| 任意 HTTP 头 | 伪造服务端响应行为 |

4. 本靶场中的利用场景

- 靶场搜索功能把搜索词反射到 `Set-Cookie` 响应头中。
- 通过 CRLF 注入 `Set-Cookie: csrf=自定义值`，可以在 victim 浏览器的靶场域名下写入任意 Cookie。
- 配合 CSRF 表单：表单 Body 中放同一个值 → 两处相等 → 绕过双提交 Cookie 校验。

5. 为什么 Exploit Server 不能直接写 Cookie

- Cookie 有域名隔离机制。
- Exploit Server 的 `Set-Cookie` 只写在自己域名下。
- 浏览器访问靶场域名时不会携带其他域名的 Cookie。
- 必须通过 CRLF 注入让靶场**自己**的响应头写入 Cookie，才能写在靶场域名下。

6. 原理链路图

```
攻击者构造请求：/?search=test%0d%0aSet-Cookie: csrf=fake
    │
    ▼
服务端把搜索词原样放进响应头（未过滤 CRLF）
    │
    ▼
响应头变成：
  Set-Cookie: search=test
  Set-Cookie: csrf=fake    ← 注入的
    │
    ▼
浏览器收到响应，执行两条 Set-Cookie
    │
    ▼
Cookie csrf=fake 写入靶场域名下
```

7. 关键概念：你控制请求，服务端写响应

| 角色 | 做了什么 |
|------|---------|
| 攻击者 | 在请求 URL 中放了带 %0d%0a 的搜索词 |
| 服务端 | 把搜索词原样放进响应头，没有过滤换行符 |
| 浏览器 | 收到响应头，遇到换行就认为是新的 Set-Cookie，存了 Cookie |

- 攻击者没有直接写响应头，而是让服务端帮忙把内容写进响应头。
- 漏洞根源：服务端未过滤用户输入中的 CRLF 字符。

8. URL 编码对照

| 字符 | 编码 | 用途 |
|------|------|------|
| CR（回车 \r） | %0d | HTTP 头换行符的第一部分 |
| LF（换行 \n） | %0a | HTTP 头换行符的第二部分 |
| 空格 | %20 | 头名称和值之间的空格 |
| 分号 ; | %3b | Cookie 属性分隔符（SameSite 等） |

通关过程：
1：在靶场搜索框输入 test，抓包查看响应头
2：确认搜索词被反射到 `Set-Cookie: search=test` 响应头中
3：在浏览器中访问 `https://靶场域名/?search=test%0d%0aSet-Cookie:%20csrf=mytest123`
4：F12 → Application → Cookies → 确认靶场域名下出现 csrf=mytest123
5：说明 CRLF 注入成功，可以在靶场域名下写入任意 Cookie
6：配合 CSRF 靶场使用：img 加载 CRLF URL → 写 Cookie → onerror 提交表单

易错点：
- 以为 CRLF 注入是改请求头 → 是改**响应头**，通过请求中的值让服务端帮忙写。
- 以为 CRLF 注入就是 XSS → CRLF 篡改响应头，XSS 篡改响应体，完全不同。
- img src 用了 Exploit Server 域名 → Cookie 写在错误域名下，必须用靶场域名。
- URL 中 %0d%0a 被浏览器解码后丢失 → 确认用 URL 编码形式。
- 搜索词中有分号但没编码 → 分号需要用 %3b 编码。