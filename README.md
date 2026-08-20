# jye_cku — 网络安全学习笔记

个人网络安全学习笔记库，托管于 GitHub (`aisjd12/jye_cku.git`)，由 Obsidian + Git 同步维护。

## 目录结构

```
jye_cku/
├── README.md        # 本索引
├── 信息泄露/         # 信息泄露专题（Burp Academy 靶场）
├── xss跨站脚本漏洞/   # XSS 跨站脚本专题（Burp Academy 靶场）
├── 渗透笔记/         # 渗透测试学习笔记
└── 靶场笔记/         # 靶场通关笔记
```

## 📁 渗透笔记

Web 渗透 / 移动端渗透 / 各类攻击技术笔记。

| 笔记 | 主题 |
|------|------|
| [[渗透笔记/1.对象]] | 测试对象分类 |
| [[渗透笔记/2.识别&解密]] | 识别与解密 |
| [[渗透笔记/H5&Vue]] | H5 & Vue 前端 |
| [[渗透笔记/HHTTP]] | HTTP 协议 |
| [[渗透笔记/Flutter]] | Flutter 安全 |
| [[渗透笔记/原生开发]] | 原生开发 |
| [[渗透笔记/封装平台]] | 封装平台 |
| [[渗透笔记/其他协议]] | 其他协议 |
| [[渗透笔记/其他模式]] | 其他测试模式 |
| [[渗透笔记/数据不回显]] | 数据不回显场景 |
| [[渗透笔记/文件下载]] | 文件下载 |
| [[渗透笔记/反弹shell]] | 反弹 shell |
| [[渗透笔记/基础命令]] | 基础命令 |
| [[渗透笔记/检测防护]] | 检测与防护 |
| [[渗透笔记/拓展模式]] | 拓展模式 |
| [[渗透笔记/常规模式：]] | 常规模式 |
| [[渗透笔记/小迪安全]] | 小迪安全课程笔记 |

## 🎯 靶场笔记

实战靶场通关记录。

| 日期 | 靶场 | 笔记 |
|------|------|------|
| 2026-08-17 | Sunrise Harbor 商城 | [[靶场笔记/2026-08-17-Sunrise-Harbor商城靶场]] |

模板见 [[靶场笔记/_模板]]。

## 🔍 信息泄露专题

PortSwigger Web Security Academy — Information disclosure 分类靶场笔记。

| 靶场 | 核心考点 |
|------|---------|
| [[信息泄露/错误信息中的信息泄露]] | 异常堆栈 / 详细报错回显 API Key |
| [[信息泄露/调试页面的信息泄露]] | phpinfo() / 环境变量 / 注释入口 |
| [[信息泄露/通过备份文件泄露的源代码]] | 目录浏览 / .bak 备份 / 硬编码凭据 |
| [[信息泄露/通过信息泄露绕过认证]] | TRACE 回显 / 自定义头伪造 |
| [[信息泄露/版本控制中的信息泄露]] | .git 泄露 / 历史提交挖密码 |

通用方法论与防御见 [[信息泄露/README]]。

## 💉 XSS 跨站脚本漏洞专题

PortSwigger Web Security Academy — Cross-site scripting 分类靶场笔记。

| 靶场 | 类型 | 核心 |
|------|------|------|
| [[xss跨站脚本漏洞/DOM xss 在sack中使用源代码 document.write location.search]] | DOM 型 | `location.search` → `document.write` |
| [[xss跨站脚本漏洞/DOM xss 在sack中使用源代码 innerHTML location.search]] | DOM 型 | `location.search` → `innerHTML` |
| [[xss跨站脚本漏洞/jQuery 选择器收集器中使用哈希变更事件的 DOM XSS]] | DOM 型 | hashchange → jQuery 选择器 sink |
| [[xss跨站脚本漏洞/jQuery 锚属性汇入中的 DOM XSS使用源代码]] | DOM 型 | `href` 可控 → `javascript:` 伪协议 |
| [[xss跨站脚本漏洞/将xss储存在HTML上下文中，且未编码]] | 存储型 | 评论区存库 → 全站渲染 |
| [[xss跨站脚本漏洞/将xss映射到html上下文中，且没有编码]] | 反射型 | `/?search=` 原样反射 |

三类型速记 + 方法论见 [[xss跨站脚本漏洞/README]]。

## 同步方式

本仓库使用 Obsidian 的 Git 插件（或手动 git）推送到 GitHub：

```bash
git add -A
git commit -m "notes: <描述>"
git push origin main
```
