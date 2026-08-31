知识点：
1. 什么是存储型 XSS？

- **本质**：攻击者提交的 payload 被服务器保存，之后其他用户访问相关页面时，payload 会从数据库中取出并进入页面。
- **本题场景**：评论表单中的 Website 字段会被保存，并作为作者名称链接的 `href` 属性值输出。
- **触发方式**：评论提交成功后，访问文章并点击对应的作者名称链接。

2. `href` 属性的作用

- `href` 用于指定超链接的目标地址：

```html
<a href="https://example.com">作者名称</a>
```

- `href="..."` 内部的值是链接目标。
- `>` 和 `</a>` 之间的内容是链接显示文字。
- 两者位置不同，只有输入进入 `href` 属性时，才可以测试 URI scheme。

3. 为什么双引号被 HTML 编码仍然可以利用

- 传统 payload：`"><script>alert(1)</script>` 依赖闭合属性并插入新标签。
- 如果 `<`、`>` 和双引号被 HTML 编码，标签注入和属性闭合都会失败。
- 但 `javascript:` 是一种 URI scheme，不依赖插入新的 HTML 标签：

```html
<a href="javascript:alert(1)">作者名称</a>
```

- 点击链接时，浏览器会执行 `javascript:` 后面的 JavaScript。
- 因此本题的关键是 `href` 目标可控，但应用没有禁止危险的 `javascript:` scheme。

4. Name 和 Website 字段的区别

- **Name 字段**通常作为链接显示文字：

```html
<a href="固定地址">javascript:alert(1)</a>
```

此时 payload 只是文本，不会执行。

- **Website 字段**通常作为 `href` 属性值：

```html
<a href="javascript:alert(1)">你的名字</a>
```

此时点击作者名称会触发 JavaScript。

5. 如何判断输入位置？

- 先在 Website 字段提交随机字符串，例如 `abc123xyz`。
- 在文章页面查看 HTML 源码或 Burp 响应原文。
- 如果看到：

```html
<a href="abc123xyz">作者名称</a>
```

说明输入进入了 `href` 属性。

- 如果看到：

```html
<a href="固定地址">abc123xyz</a>
```

说明输入在链接文字中，不是 `href` 属性。

6. 如何防御？

- 对 `href`、`src` 等 URL 属性执行协议白名单校验，只允许 `https`、`http` 等业务需要的协议。
- 明确拒绝 `javascript:`、`data:` 等危险 scheme。
- 对 URL 属性进行上下文相关编码，不能只依赖 `<`、`>` 或双引号过滤。
- 使用安全模板引擎和框架默认转义功能。
- 配置 CSP，限制脚本来源并减少 inline script 的影响。
- 对评论、昵称、Website 等存储型输入进行服务端校验和输出编码。

通关过程：
1：打开靶场文章页面，找到评论表单
2：在 Name 字段填写普通名称，在 Website 字段先填写 `abc123xyz`
3：提交评论并查看文章页面源码，确认 `abc123xyz` 位于 `<a href="...">` 的属性值中
4：将 Website 字段替换为：`javascript:alert(document.domain)`
5：重新提交评论并返回文章页面
6：点击该评论对应的作者名称链接，弹出 `alert(document.domain)` → 完成

易错点：

- 把 payload 填入 Name 字段时，它通常只会出现在 `<a>` 标签文字中，不会执行。
- 看到 `>javascript:alert(1)</a>` 说明 payload 在文本节点中；看到 `href="javascript:alert(1)"` 才说明进入了链接属性。
- 应查看 Burp 的原始响应或正确的作者链接节点，页面中可能同时存在文章链接、作者链接等多个 `<a>` 元素。
