# 基于 Referer 头的访问控制

> 官方名：Referer-based access control。机翻成"基于干涉的访问控制"是错的，refer = 引荐/来源。

## 知识点：Referer 是什么

- 浏览器**自动携带**的请求头，值是**发起这次请求的页面 URL**（来源页面）
- 例：在 `/admin` 页面点"升级用户"，请求自动带 `Referer: https://xxx/admin`
- 原本用途：来源统计、防盗链 —— **不是为鉴权设计的**
- 与 Cookie 一样属于**客户端可控字段**，任何拿它做权限判断的逻辑都可被绕过

## 通关过程

1. 抓管理员"升级用户"请求，观察请求头里有 `Referer: .../admin`
2. Repeater 里只把 Cookie 换成普通账号 → 请求仍成功 → 说明服务端鉴权不看 Cookie
3. 删掉/篡改 Referer → 请求被拒 → 定位结论：服务端靠 Referer 是否包含 `/admin` 判权限
4. 真实利用：把 CSRF PoC（POST `username=carlos&action=upgrade` 到 `/admin-roles`）托管在 exploit 服务器的 **`/admin` 路径**下，发给受害者 → 受害者浏览器自动带 `Referer: https://exploit-xxx/admin` → 通过校验

> 浏览器不允许网页随意伪造 Referer 的值，但攻击者可以**控制来源页面的 URL**——把恶意页面路径设成 `/admin` 即可"合法"地带出目标 Referer。

## 易错点（我的踩坑）

- 标题机翻成"基于干涉"，误以为是**请求方法**的问题 → 实际是 **Referer 头**
- 换 Cookie 后还能成功，一度以为和请求头无关 → 真相：Repeater 里残留了管理员的 `Referer`
- **变量分离法**：一次只改/删一个字段（Cookie、路径、某个头），请求开始被拒的那个字段就是服务端真正依赖的鉴权依据

## 防御建议

- 权限校验只依赖服务端会话/角色，不依赖 Referer 等任意客户端头
- Referer 可被攻击者间接控制（来源页 URL），也可被 Referrer-Policy 去除 → 不能作为安全边界
- 默认拒绝：管理接口必须显式校验"当前登录用户是否为管理员"
