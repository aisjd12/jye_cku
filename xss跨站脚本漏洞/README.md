# XSS 跨站脚本漏洞专题

来源：PortSwigger Web Security Academy — Cross-site scripting 分类靶场。

## 靶场索引

| 靶场 | 类型 | 源 / 汇入 | 笔记 |
|------|------|----------|------|
| [[xss跨站脚本漏洞/DOM xss 在sack中使用源代码 document.write location.search]] | DOM 型 | `location.search` → `document.write` | 服务端无感知，前端直接写 DOM |
| [[xss跨站脚本漏洞/DOM xss 在sack中使用源代码 innerHTML location.search]] | DOM 型 | `location.search` → `innerHTML` | innerHTML 不执行 script，需事件属性 |
| [[xss跨站脚本漏洞/jQuery 选择器收集器中使用哈希变更事件的 DOM XSS]] | DOM 型 | `location.hash` → `$(选择器)` | hashchange 自动触发，无需点击 |
| [[xss跨站脚本漏洞/jQuery 锚属性汇入中的 DOM XSS使用源代码]] | DOM 型 | `location.search` → `$('a').attr('href')` | `javascript:` 伪协议点击即执行 |
| [[xss跨站脚本漏洞/将xss储存在HTML上下文中，且未编码]] | 存储型 | 评论区存库 → 全站渲染 | 一次注入、所有人中招 |
| [[xss跨站脚本漏洞/将xss映射到html上下文中，且没有编码]] | 反射型 | `/?search=` → 服务端原样反射 | 需诱导点击恶意 URL |

## XSS 三类型速记

| 类型 | 存储位置 | 触发条件 | 危害 |
|------|---------|---------|------|
| 反射型 | 只存在于本次响应 | 点击恶意 URL | 临时，但可钓鱼 |
| 存储型 | 服务器数据库 | 任何人打开页面 | 持久，可打管理员 |
| DOM 型 | 客户端 JS 运行 | 打开 URL/加载页面 | 绕过服务端过滤/WAF |

## 通用方法论

1. **找注入点**：搜索框、评论、昵称、URL 参数、hash、referrer，逐个测试。
2. **判断上下文**：输入落在 HTML 正文 / 属性值 / JS 字符串 / URL，选对应 payload。
3. **选 payload**（skill 快速表）：
   - HTML 正文：`<svg onload=alert(1)>` / `<img src=1 onerror=alert(1)>`
   - 属性值：`" autofocus onfocus=alert(1)//`
   - JS 字符串：`'-alert(1)-'`
   - href：`javascript:alert(1)`
4. **看响应与源码**：是否编码？过滤哪些字符？有没有 WAF？据此换绕过（大小写、编码、截断、事件属性变体）。
5. **升级利用**：弹窗验证 → 窃 Cookie / CSRF 改资料 / 打管理员 / 键盘记录。

## 防御红线

- **输出编码优先**：上下文相关编码（HTML/属性/JS/URL），别只做输入过滤。
- **禁用危险 sink**：`document.write`、`innerHTML`、`$()`、`eval` 处理不可信数据一律换 `textContent`/安全 DOM API。
- **协议白名单**：`href`/`src` 只允许 `http(s)`，拒 `javascript:`/`data:`。
- **CSP 兜底**：`script-src` 加 nonce / 只允许可信源。
- **HttpOnly Cookie** + 关键操作二次验证，降低被窃后危害。
