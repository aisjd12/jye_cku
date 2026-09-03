# CSRF 跨站请求伪造专题

来源：PortSwigger Web Security Academy — Cross-site request forgery 分类靶场。

## 靶场索引

| 靶场 | Token 防护 | 绕过方式 | 难度 |
|------|----------|---------|------|
| [[csrf漏洞/无防御的csrf漏洞]] | 无 | 直接构造跨站表单 | ★ |
| [[csrf漏洞/CSRF，令牌验证依赖于请求方法]] | POST 有 Token | 改用 GET 绕过校验 | ★★ |
| [[csrf漏洞/CSRF：令牌校验取决于令牌是否存在]] | 有 Token | 删除 Token 参数 | ★★ |
| [[csrf漏洞/CSRF，令牌不绑定用户会话]] | 有 Token | 用 A 的 Token + B 的 Cookie | ★★★ |
| [[csrf漏洞/CSRF，令牌绑定在非会话Cookie上]] | Token 绑 csrfKey | CRLF 注入写 csrfKey + img onerror | ★★★★ |
| [[csrf漏洞/CSRF，令牌在Cookie中重复]] | 双提交 Cookie | CRLF 注入写任意 csrf + 两处相等 | ★★★★ |
| [[csrf漏洞/SameSite 通过方法覆盖 Lax 旁路]] | SameSite=Lax | GET 顶层导航 + _method=POST 方法覆盖 | ★★★★ |

## CSRF 核心原理速记

```
受害者登录 → 浏览器保存 session Cookie
→ 受害者访问攻击者页面
→ 页面表单向目标网站发起请求
→ 浏览器自动携带 Cookie
→ 服务端认为是合法用户操作 → 执行
```

- 攻击者**看不到**受害者的 Cookie，只是让浏览器**自动带**。
- 攻击者控制：请求地址、方法、参数。
- 浏览器自动提供：session Cookie。

## Token 防护绕过速查

| 防护类型 | 漏洞特征 | 绕过方式 |
|---------|---------|---------|
| 无防御 | 无 Token | 直接构造表单 |
| 方法相关 | GET 不校验 Token | 改用 GET 提交 |
| 存在性检查 | 删参数跳过校验 | 删除 Token 参数 |
| 不绑会话 | Token 不检查归属 | 用别人的 Token |
| 绑非会话 Cookie | csrfKey 不绑 session | CRLF 写真实 csrfKey |
| 双提交 Cookie | 只比较两处相等 | CRLF 写任意 csrf + Body 同值 |
| SameSite=Lax | 跨站 POST 不带 Cookie | GET 顶导 + 方法覆盖（_method=POST） |

## Exploit Server 三字段

| 字段 | 作用 | 常见填法 |
|------|------|---------|
| File | 页面路径 | `/` |
| Head | HTTP 响应头 | `Content-Type: text/html; charset=utf-8` |
| Body | HTML 页面内容 | CSRF 表单 + 脚本 |

## img onerror 模式

第5、6关需要先写 Cookie 再提交表单，用 img onerror 保证时序：

```html
<img src="靶场搜索接口+CRLF注入" onerror="document.forms[0].submit()">
```

```
页面加载 → img 请求靶场 → CRLF 写 Cookie → img 失败 → onerror 提交表单
```

- 不能用 `<script>submit()</script>`：Cookie 还没写入。
- img 放在 form 外面。
- img src 必须用靶场域名（不是 Exploit Server 域名）。

## 通用方法论

1. **抓状态修改请求**：找到修改邮箱/密码/权限的 POST 请求。
2. **检查 Token**：有没有 csrf Token？Token 在哪里（Body/Cookie/Header）？
3. **测试 Token 校验逻辑**：
   - 删 Token → 还行吗？
   - 改 Token 值 → 还行吗？
   - 换别人的 Token → 还行吗？
   - 换请求方法 → 还行吗？
   - 两处 Token 是否只比较相等性？
4. **构造 PoC**：根据绕过方式构造 HTML 表单，放到 Exploit Server。
5. **投递验证**：Store → Deliver exploit to victim → 检查 victim 是否中招。

## SameSite 速记

`
SameSite=Lax 时跨站三道隔离：
  ✅ GET 顶层导航带 Cookie（<a> / location / form GET submit）
  ❌ 跨站 POST 表单不带 Cookie
  ❌ img / fetch / XHR 不带 Cookie
`

绕过思路：**用带 Cookie 的触发方式（GET 顶导）+ 方法覆盖（_method=POST）骗服务端**。

## 防御红线

- **使用 CSRF Token**：绑定到当前会话，校验存在性和有效性。
- **SameSite Cookie**：`SameSite=Lax` 或 `Strict`。
- **校验 Origin/Referer**：拒绝跨站请求。
- **状态修改只允许 POST/PUT/DELETE**：不接受 GET。
- **Token 一次性使用**：用后即焚，防重放。
- **过滤 CRLF**：防止用户输入注入 HTTP 响应头。

## 踩坑总结（7 关通用）

| 坑 | 原因 | 解决 | 出现关卡 |
|----|------|------|---------|
| action/src 带 `[url](url)` | 从聊天窗口复制 URL，Markdown 格式被带入 | 手动输入纯 URL | 第1~6关反复出现 |
| HTML 标签前有 `\` | 从聊天复制时反斜杠被带入 | 手动输入或检查 | 第1~6关反复出现 |
| email 用 `&#64;` 实体编码 | 从某处复制时被编码 | 直接写明文 `@` | 第1关 |
| Burp PoC 缺参数 | 生成器只出模板，不自动补全 | 手动加 hidden input | 第1、2关 |
| GET 参数放 Body | GET 不读 Body | 参数移到 URL `?` 后面 | 第2关 |
| csrfKey 值为空 | 只写了 `%3b` 没填值 | 填实际 csrfKey | 第5关 |
| img src 用 Exploit Server 域名 | Cookie 写在错误域名下 | 用靶场域名 | 第5、6关 |
| Token 过期 | 拿到后隔太久才用 | 快速完成 Store + Deliver | 第4关 |
| 同时改 Cookie 和 Token | 等于还是用 A 的会话 | 只改 Token，Cookie 不动 | 第4关 |
| 用 script 直接 submit() | Cookie 还没写入 | 用 img onerror 保证时序 | 第5、6关 |
| 用真实 Token 而非任意值 | 双提交 Cookie 只需相等 | 随便定一个值 | 第6关 |
| 靶场重置后值失效 | Token/Key 重新生成 | 重新抓包获取新值 | 第5、6关 |