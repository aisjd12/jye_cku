# 带外检测的盲SSRF

> 官方名：Blind SSRF with out-of-band detection。核心：你看不到服务器请求的返回内容，但可以让它访问一个你能收到“访问记录”的外部地址，用带外交互证明 SSRF 存在。

## 知识点：盲 SSRF + OOB 检测

### 盲 SSRF 是什么

- 普通 SSRF：你能看到服务器代你请求后的**响应内容**
- 盲 SSRF：你只能触发服务器去请求，但**看不到返回结果**

```text
你 → 目标服务器 → 内部 URL
                ↓
            响应你看不到
```

### OOB 检测思路

让目标服务器去访问一个**你能收到访问记录**的地址：

```text
你 → 目标服务器 → Collaborator 域名
                        ↓
              Collaborator 后台记录这次访问
```

Burp Collaborator 会给你一个专属域名，例如：

```text
xxxxx.oastify.com
```

目标服务器只要尝试访问它，就会触发 DNS / HTTP / TLS 交互，你在 Collaborator 面板里就能看到。

## 通关过程

1. 找到会触发服务器端请求的功能点（例如库存查询、导入远程文件、webhook 等）
2. 打开 Burp 的 **Collaborator** 标签
3. 点击 **Copy to clipboard**，拿到你的专属域名
4. 把可控参数（或 `Referer` 等）改成这个域名
5. 发送请求
6. 回到 Collaborator，点击 **Poll now**
7. 看到 DNS / HTTP 交互记录，说明目标服务器确实替你发起了这次请求，SSRF 成立

## 易错点（我的踩坑）

### 1. 不需要自己搭 DNS 服务器

- Collaborator 已经帮你做了“可回连地址”
- 你要做的只是把目标服务器引导到这个域名上

### 2. DNS 查询是怎么来的

- 目标服务器要访问 `xxxxx.oastify.com`，必须先做 DNS 解析
- 这个解析请求会被 Collaborator 的权威 DNS 服务器捕获
- 所以你在 Collaborator 里能看到 DNS 交互

```text
目标服务器 → 解析 Collaborator 域名 → DNS 查询被捕获
```

### 3. 为什么 `Referer` 头会被服务器解析

- `Referer` 确实是浏览器自动带上的“来源页面”
- 但它本质上只是一个**客户端可控的 HTTP 头**
- 服务端代码完全可以把它当成普通输入来处理

在这类漏洞里，应用把 `Referer` 的值当成了要请求的 URL（或影响了要请求的地址）：

```text
你控制 Referer: http://xxxxx.oastify.com
        ↓
服务端读取这个头
        ↓
把它作为请求目标
```

所以不要把 HTTP 头当成“只读元数据”，它同样是**输入**。

## 防御建议

- 不允许服务器端功能直接使用用户可控的 URL 或 Header 值作为请求目标
- 对出站请求做白名单校验
- 禁止访问回环地址、内网段、云元数据地址
- 对 `Referer` 这类 Header 只做统计/日志用途，不要作为业务逻辑输入