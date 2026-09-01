知识点：
1. 漏洞核心：Token 存在但不绑定用户会话

- 服务端校验 csrf Token：检查 Token 是否存在、格式是否正确。
- 但**不检查 Token 是否属于当前登录用户**。
- 因此 A 账号的 Token 可以配合 B 账号的 Cookie 使用。

```
Token 来源：A 账号（攻击者）
Cookie 来源：B 账号（受害者）
结果：B 账号邮箱被修改
```

2. 正常 vs 漏洞对比

| 场景 | Token 来源 | Cookie 来源 | 结果 |
|------|-----------|-----------|------|
| 正常 | 自己的 | 自己的 | 成功 |
| 漏洞利用 | A 的 | B 的 | 成功（漏洞） |
| 安全实现 | A 的 | B 的 | 拒绝 |

3. Token 过期 vs Token 绑定

- 过期：Token 有时间限制，超时失效，这是正常行为。
- 绑定会话：Token 必须属于当前登录用户，这是本关缺失的校验。
- 测试时需在时间窗口内快速完成：拿到 A 的 Token → 替换到 B 的请求 → 发送。

4. Exploit Server PoC 结构

```html
<form action="https://靶场域名/my-account/change-email" method="POST">
  <input type="hidden" name="csrf" value="攻击者账号的Token">
  <input type="hidden" name="email" value="测试邮箱">
</form>
<script>
  document.forms[0].submit();
</script>
```

- csrf 值来自攻击者账号
- Cookie 由 victim 浏览器自动携带
- email 不能是攻击者自己的邮箱

5. 防御方案

- Token 绑定到当前会话/用户：服务端校验 Token 是否属于当前 session。
- 使用服务端存储的 Token：登录时生成，存入 session，请求时比对。
- Token 一次性使用：用后即焚。

通关过程：
1：登录 A 账号（攻击者），抓修改邮箱请求，记录 csrf 值
2：登录 B 账号（受害者，用另一个浏览器或无痕窗口），抓修改邮箱请求
3：在 B 的请求中，保持 Cookie 不动，把 csrf 值替换成 A 的
4：email 改成测试值 → 发送 → B 的邮箱被修改成功 → 确认 Token 不绑定会话
5：在 Exploit Server 构造 PoC：表单中 csrf 填 A 的值，email 填测试邮箱
6：Store → Deliver exploit to victim → victim 邮箱被修改 → 靶场解决

易错点：
- 同时改了 Cookie 和 Token → 等于还是在用 A 的会话，不构成漏洞验证。
- Token 拿到后隔太久才用 → 过期失效。
- 让 B 在浏览器中正常提交表单后再去抓旧请求 → 可能生成新 Token，旧 Token 失效。
- 在 Repeater 中测试成功但没投递给 victim → 靶场需要通过 Exploit Server 验证。
- 以为 Token 过期就是绑定会话 → 过期和绑定是两个不同的概念。