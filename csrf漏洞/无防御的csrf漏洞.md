知识点：
1. 什么是无防御的 CSRF

- 服务端对状态修改请求没有任何 CSRF 防护：无 Token、无 SameSite、无 Origin/Referer 校验。
- 攻击者只需构造一个跨站表单，让受害者浏览器自动携带 Cookie 发请求即可。
- 这是最基础的 CSRF 形态，所有更复杂的变体都是在它之上加防御再找绕过。

2. CSRF 的核心原理

```
受害者登录目标网站 → 浏览器保存 session Cookie
→ 受害者访问攻击者页面
→ 页面表单向目标网站发起 POST 请求
→ 浏览器自动携带目标网站的 Cookie
→ 服务端认为是合法用户操作 → 执行
```

- 攻击者**看不到**受害者的 Cookie，只是让浏览器**自动带**。
- 攻击者控制的是：请求地址、方法、参数。
- 浏览器自动提供的是：session Cookie。

3. Exploit Server 三个字段的作用

| 字段 | 作用 | 本关填什么 |
|------|------|-----------|
| File | 页面访问路径 | `/` |
| Head | HTTP 响应头 | `Content-Type: text/html; charset=utf-8` |
| Body | 页面 HTML 内容 | CSRF 表单 |

4. PoC 结构

```html
<form action="https://靶场域名/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="测试邮箱">
</form>
<script>
  document.forms[0].submit();
</script>
```

- `action`：目标接口地址
- `method`：POST（状态修改用 POST）
- `hidden input`：提交参数
- `submit()`：页面加载后自动提交表单

5. 防御方案

- 使用 CSRF Token：服务端生成随机 Token，要求每个状态修改请求携带。
- 设置 SameSite Cookie：`SameSite=Lax` 或 `Strict`。
- 校验 Origin / Referer 头。
- 敏感操作二次验证（密码、验证码）。

通关过程：
1：登录账号，打开修改邮箱页面，抓到 `POST /my-account/change-email` 请求
2：检查请求中是否有 CSRF Token → 发现完全没有，确认无防御
3：在 Exploit Server 中构造自动提交表单的 HTML 页面
4：`File` 填 `/`，`Head` 填 `Content-Type: text/html; charset=utf-8`
5：`Body` 填写表单 HTML，email 改成不属于自己的测试邮箱
6：Store → View exploit 确认页面正常 → Deliver exploit to victim
7：victim 邮箱被修改 → 靶场解决

易错点：
- 把 `action` 写成 Markdown 链接格式 `[https://...](https://...)`，浏览器无法解析。
- 只在 Burp Repeater 中测试成功 → 靶场需要通过 Exploit Server 投递给 victim 才算通过。
- 以为需要获取 victim 的 Cookie → CSRF 不需要看到 Cookie，浏览器自动携带。
- email 填了自己的邮箱 → 不能和自己已有邮箱相同。
