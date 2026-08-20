知识点：
1. `innerHTML` 与 `document.write` 的危险区别

- `innerHTML` 同样会把字符串**按 HTML 解析**插入 DOM，所以 `"><img src=1 onerror=alert(1)>` 会被渲染成真实元素并触发事件。
- **关键差异**：`innerHTML` 不执行 `<script>` 标签（脚本不会被运行），所以必须用**事件属性**（`onerror`/`onload`）或 `<svg>`、`<iframe>` 等元素触发，而不是直接塞 `<script>`。

2. 为什么 location.search 是经典源

- `location.search` 返回 URL 中 `?` 之后的部分，攻击者完全可控，且**不经过服务端**，前端 JS 读取后直接拼接。
- 常见写法：`document.getElementById('x').innerHTML = location.search.slice(1)` 或 `innerHTML += params`。

3. 标准 payload 思路

- 闭合 HTML 上下文（`"><` 或 `</option>` 等），再插入带事件属性的标签：
  - `<img src=1 onerror=alert(1)>`
  - `<svg onload=alert(1)>`
  - `<iframe src="javascript:alert(1)">`

4. 如何防御？

- 不用 `innerHTML` 插入不可信数据；用 `textContent` 或 DOM 节点创建 API。
- 使用框架内置的转义（React/Vue 默认转义）避免危险拼接。
- CSP 兜底：`default-src 'none'`、`script-src 'nonce-xxx'`。

通关过程：
1：打开靶场首页，页面有搜索框，输入内容提交后 URL 变为 `/?search=<输入>`
2：传 payload：`/?search=<img src=1 onerror=alert(1)>`
3：页面弹出 alert 窗口 → 完成
