# SSRF 漏洞专题

来源：PortSwigger Web Security Academy — Server-side request forgery 分类靶场。

## 靶场索引

| 靶场 | 漏洞类型 | 绕过方式 | 难度 |
|------|---------|---------|------|
| [[ssrf漏洞/对本地服务器进行基础SSRF]] | SSRF → 本地管理接口 | `stockApi` 改成 `http://localhost/admin/delete?...`（嵌套 URL 需二次编码） | ★ |
| [[ssrf漏洞/基础SSRF对抗另一个后端系统]] | SSRF → 内网主机管理接口 | stockApi 扫描 192.168.0.0/24 定位隐藏后端 | ★★ |
| [[ssrf漏洞/带外检测的盲SSRF]] | 盲 SSRF + OOB 回连 | 用 Collaborator 域名让服务器主动访问你，产生 DNS/HTTP 交互 | ★★ |
| [[ssrf漏洞/带有黑名单输入滤波器的SSRF]] | SSRF + 黑名单绕过 | 等价 IP、大小写、二次 URL 编码绕过 `localhost` / `admin` 过滤 | ★★★ |
| [[ssrf漏洞/通过开放重定向绕过SSRF过滤]] | SSRF + 开放重定向绕过 | `stockApi` 先指向 `nextProduct`，再通过 302 跳到内网管理接口 | ★★★ |
| [[ssrf漏洞/盲SSRF配合Shellshock漏洞]] | 盲 SSRF + Shellshock 命令执行 | `User-Agent` 带 payload，`Referer`/`stockApi` 指向内网，命令结果经 Collaborator 带出 | ★★★★ |
| [[ssrf漏洞/带有白名单输入滤波器的SSRF]] | SSRF + 白名单绕过 | 利用 `@` / `#` / 双重 URL 编码造成解析差异，让白名单和后端看到不同 host | ★★★★ |

## 核心概念速记

```text
SSRF = Server-Side Request Forgery（服务端请求伪造）
```

- 应用功能会让**服务器主动去请求用户指定的 URL**
- 攻击者把这个 URL 指向**内网 / 本机 / 云元数据**
- 服务器替你访问了一个你本来访问不到的资源

## 为什么 SSRF 危险

1. **内网穿透**：外网不能访问的 admin，服务器可以访问 localhost
2. **绕过防火墙**：服务器所在内网通常是可信区
3. **云元数据读取**：AWS/GCP/Azure 的 169.254.169.254 可拿到凭据
4. **端口/服务探测**：根据响应差异探测内网端口

## 防御红线

- 不允许用户指定完整 URL（协议、主机、端口）
- 出网请求走白名单域名
- 禁止回环地址、内网网段、云元数据 IP
- 处理 DNS 解析后再校验（防 DNS Rebinding）
- 对外发请求服务使用独立网络分区，与内网管理面板隔离