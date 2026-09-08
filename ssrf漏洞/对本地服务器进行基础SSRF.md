# 对本地服务器进行基础SSRF

> 官方名：Basic SSRF against the local server。核心：利用一个"由服务器代你发请求"的功能，让它访问本机回环地址上的管理接口。

## 知识点：SSRF 是什么

- SSRF = Server-Side Request Forgery（服务端请求伪造）
- 应用里有些功能会让**服务器主动去请求一个用户指定的 URL**，例如库存查询、导入远程文件、webhook
- 攻击者把 URL 改成**内网地址**，服务端就会替你访问它自己或内网里的服务

```text
正常：stockApi=http://stock.weliketoshop.net:8080/...
攻击：stockApi=http://localhost/admin
     ↑ 本来是请求外网库存服务，改成 localhost 后变成访问服务器本机的 admin
```

### localhost / 127.0.0.1 / 0.0.0.0 的含义

| 写法 | 含义 |
|------|------|
| `localhost` | 本机回环域名 |
| `127.0.0.1` | 本机回环 IP |
| `0.0.0.0` | 常见于服务监听地址，不一定是"本机地址"，但很多系统访问它会绕到本机 |

> 管理面板通常不暴露公网，只允许从服务器自身访问。SSRF 的请求源是**服务器自己**，正好符合这个条件。

## 通关过程

1. 打开商品详情页，点"检查库存"，Burp 抓到 POST 请求，body 里有 `stockApi=http://stock.weliketoshop.net:8080/product/stock/check...`
2. 把 `stockApi` 改成 `http%3a%2f%2flocalhost%2fadmin`，重放 → 返回管理页面 HTML
3. 观察管理页面里的"删除用户"链接，路径形如 `/admin/delete?username=carlos`
4. 把 `stockApi` 改成这个完整的 delete URL：
   `http://localhost/admin/delete?username=carlos`
5. **关键坑**：它是嵌套在 `stockApi` 参数值里的第二层 URL，里面的 `?`、`=` 必须同样 URL 编码
6. 最终值：
   `http%3a%2f%2flocalhost%2fadmin%2fdelete%3Fusername%3Dcarlos`
7. 通过 `/product/stock` 发送 POST，删除 carlos 成功

## 易错点（我的踩坑）

1. 以为只加 `stockApi=http%3a%2f%2flocalhost%2fadmin` 就能删除
   - 那只是访问管理页面，还没调用删除接口
2. 第一次删除失败的原因：
   - 内层 URL 的 `?` 和 `=` 没有编码
   - 外层解析参数时会把它们当成**外层语法的一部分**，把内层查询串截断/破坏
3. 嵌套 URL 经验法则：
   - 里面的所有特殊字符都要 `%xx` 编码
   - `:` → `%3a`，`/` → `%2f`，`?` → `%3F`，`=` → `%3D`
4. 请求还是发到外层入口 `/product/stock`，不要直接去请求 `/admin/delete`

## 防御建议

- **禁止回环/内网地址**：localhost、127.0.0.1、0.0.0.0、内网网段、云元数据 IP
- 只允许白名单目标域名，不要让用户随意指定完整 URL
- 优先使用**服务端固定模板**：用户只能传业务参数，不能传协议、主机、路径
- DNS/重定向也要校验，防止 `localhost` 的别名绕过
- 为对外发请求的服务使用**独立网络分区**，与内部管理面板隔离