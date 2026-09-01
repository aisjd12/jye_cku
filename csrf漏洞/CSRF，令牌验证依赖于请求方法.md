知识点：
1. 漏洞核心：Token 校验逻辑因请求方法而异

- 正常请求：`POST /my-account/change-email`，Body 中带 `csrf` Token。
- 服务端对 POST 请求执行 CSRF Token 校验。
- 但把请求方法改成 `GET` 后，服务端使用了另一条处理逻辑，该逻辑不检查 Token，却仍然执行状态修改操作。

```
POST → 校验 Token ✓
GET  → 跳过 Token 校验 ✗ → 仍然修改邮箱
```

2. GET 与 POST 的参数位置

| 方法 | 参数位置 | 示例 |
|------|---------|------|
| POST | 请求体（Body） | `email=test@qq.com` |
| GET | URL 查询字符串 | `?email=test@qq.com` |

- 改成 GET 后，参数必须从 Body 移到 URL 的 `?` 后面。
- 服务端按 GET 的方式从查询字符串读取参数，不读 Body。
- 如果 GET 请求仍把参数放 Body 中，服务端提示"缺少参数"。

3. HTML 表单默认 GET

- `<form>` 不写 `method` 时，默认使用 `GET`。
- GET 表单的 `<input type="hidden">` 会被浏览器拼接到 URL 查询字符串中。

```html
<!-- 不写 method，默认 GET -->
<form action="https://靶场域名/my-account/change-email">
  <input type="hidden" name="email" value="test@example.com">
</form>
```

浏览器生成：

```
GET /my-account/change-email?email=test%40example.com
```

4. 防御方案

- 对所有请求方法统一执行 Token 校验，不分 GET/POST。
- 状态修改接口只允许 POST/PUT/DELETE，不接受 GET。
- 禁止 GET 请求执行写操作（RESTful 规范）。

通关过程：
1：登录账号，抓到 `POST /my-account/change-email` 请求，Body 中有 `csrf` Token 和 `email` 参数
2：在 Repeater 中把方法改成 GET，直接发送 → 服务端提示缺少 `email` 参数
3：把 `email` 参数从 Body 移到 URL 查询字符串：`GET /my-account/change-email?email=test@qq.com`
4：删除 Body 和多余的 `Content-Type` 头 → 发送 → 邮箱修改成功，说明 GET 不校验 Token
5：在 Exploit Server 构造 GET 表单 PoC
6：`action` 填靶场接口地址，不写 `method`（默认 GET）
7：隐藏字段 `name="email" value="测试邮箱"`
8：加 `document.forms[0].submit()` 自动提交
9：Store → Deliver exploit to victim → victim 邮箱被修改 → 靶场解决

易错点：
- 改成 GET 后参数仍放在 Body → 服务端读不到参数，提示缺少 email。
- URL 中 `?` 被复制粘贴时丢失或变成控制字符 → 返回 404。
- Burp 生成的 PoC 没有自动把 POST Body 参数转成 GET 查询参数 → 需要手动加 hidden input。
- 以为 GET 不会修改数据 → 本关的核心就是 GET 绕过了 Token 校验但仍然执行修改。