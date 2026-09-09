# 带有白名单输入滤波器的SSRF

> 官方名：SSRF with whitelist-based input filter。核心：白名单不是简单判断“字符串是否包含允许域名”，而是要正确解析 URL 的 `userinfo`、`host`、`fragment` 等结构。这个靶场利用 URL 解析差异，让白名单看到一个“允许”的地址，后端实际请求另一个地址。

## 知识点：URL 结构

一个标准 URL 可以拆成：

```text
scheme://[userinfo@]host[:port]/path[?query][#fragment]
```

例如：

```text
http://user@localhost/admin#tag
```

对应关系是：

| 部分 | 值 | 含义 |
|------|------|------|
| scheme | `http` | 协议 |
| userinfo | `user` | 用户信息，放在 `@` 前面 |
| host | `localhost` | 真正请求的主机 |
| port | 空 / `80` | 端口 |
| path | `/admin` | 路径 |
| fragment | `tag` | 片段，通常不会发给服务器 |

---

## 知识点：`@` 的作用

`@` 是 URL 里 **userinfo 和 host 的分界符**。

```text
http://allowed.com@localhost/admin
```

解析结果是：

```text
userinfo = allowed.com
host     = localhost
path     = /admin
```

也就是说：

```text
@ 前面的部分：用户信息
@ 后面的部分：真正要请求的主机
```

如果白名单逻辑只是简单地判断：

```text
URL 是否以 http://allowed.com 开头
```

它就可能被 `@` 前面的 `allowed.com` 骗过，但后端实际请求的是 `localhost`。

---

## 知识点：`#` 的作用

`#` 是 **fragment 分隔符**。

```text
http://localhost/admin#xxx
```

解析结果是：

```text
host     = localhost
path     = /admin
fragment = xxx
```

关键点：

```text
# 后面的内容通常不会作为请求路径发给服务器
```

所以它可以用来“截断”某些只看前半部分的逻辑，或者让不同解析器对 URL 的理解产生差异。

---

## 知识点：双重 URL 编码

你这次用到的核心是：

```text
#  ->  %23  ->  %2523
```

也就是说：

```text
原始字符：#
一次编码：%23
二次编码：%2523
```

如果不同解析层的解码次数不一样，就会出现：

```text
白名单层看到的字符
        ≠
后端实际看到的字符
```

这就是 **parser differential**。

---

## 通关思路

1. 找到可控的服务器端请求参数，例如 `stockApi`
2. 直接改成：

   ```text
   http://localhost/admin
   ```

   会被白名单拦截

3. 观察白名单允许的域名，例如：

   ```text
   stock.weliketoshop.net
   ```

4. 构造一个利用 `@` 和 `#` 的 URL，让白名单看到允许域名，但后端实际请求 `localhost`
5. 需要时对 `#` 做二次 URL 编码，利用不同解析层的解码差异
6. 最终让服务器实际请求：

   ```text
   http://localhost/admin
   ```

7. 再把路径改成删除用户的接口，完成靶场

---

## 我踩过的坑

### 1. 一开始以为单纯 URL 编码就能绕过

单纯把字符编码成 `%xx`，白名单和后端可能都做同样处理，结果还是一样。

这关真正的关键是：

```text
不是字符变形
而是结构解析差异
```

要让白名单和后端对同一个 URL 的理解不一致。

---

### 2. 把 `@` 理解成“用户名和内容的分隔”

更准确地说：

```text
@ = userinfo 和 host 的分隔符
```

例如：

```text
http://allowed.com@localhost/admin
```

- `allowed.com` 是 userinfo
- `localhost` 才是 host

---

### 3. 把 `#` 理解成“忽略后面的内容”

`#` 确实表示 fragment，但它不是普通注释符。

它的准确含义是：

```text
# 后面的部分是 fragment，通常不发给服务器
```

所以可以用来让某些只解析前半部分的逻辑产生误判。

---

## 防御建议

- 不要用简单字符串匹配做白名单
- 必须用标准 URL 解析器提取 host，再和白名单比较
- 解析前先规范化 URL，处理重复编码
- 校验最终请求的 host，而不是原始字符串的前缀
- 只允许固定协议、固定域名、固定端口
- 禁止访问内网地址、回环地址、云元数据地址
- 出网请求走独立代理，并做统一校验
