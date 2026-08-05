---
title: "Proxifier 代理原理、准备工作与使用教程"
description: "从工作原理到分流规则，介绍使用 Proxifier 前需要准备什么，以及如何配置 SOCKS5 代理、DNS 和直连规则。"
slug: proxifier-proxy-guide
date: 
image:
categories:
    - stu
tags:
    - Proxifier
    - 代理
    - SOCKS5
    - 网络工具
weight: 1
---

Proxifier 是一款强制代理工具。它能够让原本不支持代理设置的程序，通过 SOCKS 或 HTTPS 代理访问网络。

需要注意的是，Proxifier 本身不提供代理线路。它更像一个流量调度器：拦截程序发起的网络连接，根据预先配置的规则，决定让连接通过代理、直接访问、进入代理链，或者直接阻止。

本文介绍 Proxifier 的基本原理、开始使用前需要准备的内容，以及 Windows 环境下的常见配置方法。

## 一、Proxifier 的工作原理

浏览器等程序通常可以主动读取系统代理设置，但一些命令行工具、旧软件或专用客户端不会使用系统代理。Proxifier 可以在这些程序建立网络连接时介入，并按照规则重新安排连接路径。

其基本流程如下：

```text
应用程序发起连接
        ↓
Proxifier 捕获连接
        ↓
从上到下匹配 Proxification Rules
        ↓
Proxy / Chain / Direct / Block
        ↓
代理服务器或目标服务器
```

每条规则可以根据以下条件匹配：

- 应用程序，例如 `abc.exe`；
- 目标域名或 IP，例如 `*.example.com`；
- 目标端口，例如 `80;443`；
- 应用、目标地址和端口的组合。

规则支持四种常用动作：

| 动作 | 作用 |
| --- | --- |
| Proxy | 通过指定代理服务器连接 |
| Chain | 通过代理链连接 |
| Direct | 不经过代理，直接连接目标 |
| Block | 阻止连接 |

Proxifier 的规则按照从上到下的顺序匹配，连接命中第一条符合条件的规则后便不再继续匹配，因此规则顺序非常重要。

### Proxifier 与系统代理、VPN 的区别

系统代理主要对主动读取代理设置的软件生效；Proxifier 则可以根据进程强制转发连接，因此更适合精确控制某个程序。

Proxifier 也不等同于完整 VPN。它主要适用于 TCP 连接，而游戏、语音通话、QUIC/HTTP3 等功能可能依赖 UDP。如果目标是代理所有协议和系统流量，代理客户端的 TUN 模式或 VPN 通常更合适。

## 二、开始前需要准备什么

### 1. 一个可用的代理入口

这是最重要的准备工作。至少需要知道以下信息：

```text
服务器地址：127.0.0.1、远程 IP 或域名
端口：例如 1080、7891
协议：SOCKS5、SOCKS4 或 HTTPS
账号密码：代理要求认证时需要
```

代理入口可能来自：

- Clash 或 Mihomo 提供的本地 SOCKS/Mixed 端口；
- v2rayN 提供的本地 SOCKS 端口；
- SSH 动态端口转发；
- 公司或服务商提供的远程 SOCKS5/HTTPS 代理。

一般推荐优先选择 SOCKS5，兼容性相对较好。

> 订阅链接以及 VLESS、VMess、Trojan、Shadowsocks 节点通常不能直接填入 Proxifier。应先让 Clash、Mihomo 或 v2rayN 等客户端连接节点，再将客户端开放的本地 SOCKS5 端口交给 Proxifier。

### 2. 确认本地代理端口

在代理客户端中查看端口配置，常见形式如下：

```text
HTTP Port: 7890
SOCKS Port: 7891
Mixed Port: 7890
```

例如客户端显示 SOCKS 端口为 `7891`，则 Proxifier 中可以填写：

```text
Address: 127.0.0.1
Port: 7891
Protocol: SOCKS5
```

### 3. 明确代理范围

配置前应先决定哪些流量需要经过代理：

- 仅代理某个程序；
- 仅代理指定域名或端口；
- 除局域网外全部代理；
- 代理 Proxifier 能处理的全部 TCP 连接。

仅代理指定程序通常最稳定，也最不容易影响系统中的其他软件。

同时应找到程序真正联网的 `.exe` 文件。部分软件会先运行启动器，再由启动器创建其他进程，实际联网的可能不是启动器本身。可以通过任务管理器或 Proxifier 的连接日志确认。

### 4. 避免重复代理

如果目标程序内部已经设置了 HTTP 或 SOCKS 代理，通常应将其恢复为直接连接，再交给 Proxifier 统一处理，否则可能形成双重代理：

```text
应用内部代理 → Proxifier 再次代理 → 连接异常
```

代理客户端本身仍需保持运行，因为 Proxifier 需要连接它提供的本地端口。

### 5. 准备直连例外

建议始终让本机回环地址直接连接：

```text
localhost
127.0.0.1
::1
```

如果还需要访问路由器、NAS、打印机或公司内网，可以将以下私有地址段设为直连：

```text
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

这样可以避免全局代理导致本地代理端口或局域网设备无法访问。

### 6. 安装与权限

建议从 [Proxifier 官网](https://www.proxifier.com/) 获取安装程序。Windows Standard Edition 安装时需要管理员权限，并会安装相应的网络组件。

Portable Edition 无需正常安装，但不能处理部分后台服务及没有图形界面的应用。如果需要代理系统服务或命令行程序，通常应使用 Standard Edition。

## 三、配置 Proxifier

### 1. 添加代理服务器

打开 Proxifier，依次进入：

```text
Profile → Proxy Servers → Add
```

填写代理信息：

- `Address`：代理地址，例如 `127.0.0.1`；
- `Port`：SOCKS 或 HTTPS 代理端口；
- `Protocol`：通常选择 `SOCKS Version 5`；
- `Authentication`：远程代理要求认证时填写账号密码。

填写后点击 `Check` 测试代理。如果连接和数据传输测试均成功，说明代理入口基本可用。

### 2. 创建程序分流规则

进入：

```text
Profile → Proxification Rules
```

例如只让 Telegram 通过代理：

```text
Name: Telegram
Applications: telegram.exe
Target Hosts: Any
Target Ports: Any
Action: SOCKS5 127.0.0.1:7891
```

然后将 `Default` 设置为 `Direct`。规则顺序可以是：

```text
1. Localhost  → Direct
2. Telegram   → SOCKS5 127.0.0.1:7891
3. Default    → Direct
```

这样只有 Telegram 会经过代理，其他程序保持直接连接。

### 3. 配置全局代理

如果希望 Proxifier 能处理的连接默认都通过代理，可以将 `Default` 的 Action 改为指定代理：

```text
Localhost → Direct
Default   → SOCKS5 127.0.0.1:7891
```

不建议删除默认的 `Localhost → Direct` 规则，否则本机程序之间的通信可能被错误发送给代理。

### 4. 按域名进行分流

例如只让某个网站通过代理，可以新建规则：

```text
Applications: Any
Target Hosts: example.com; *.example.com
Target Ports: Any
Action: Proxy
```

该规则应放在 `Default` 之前。

### 5. 配置局域网直连

在全局代理规则之前创建一条规则：

```text
Name: LAN Direct
Target Hosts: localhost; 127.0.0.1; 10.0.0.0/8; 172.16.0.0/12; 192.168.0.0/16
Action: Direct
```

随后再让 `Default` 通过代理，即可实现“局域网直连、其他连接走代理”。

## 四、DNS 解析设置

进入：

```text
Profile → Name Resolution
```

启用 `Resolve hostnames through proxy` 后，域名会交给代理端解析，适用于以下情况：

- 本地 DNS 无法正确解析目标域名；
- 不希望域名查询直接发送给本地 DNS；
- 域名在本地和代理所在地的解析结果不同。

不过，代理 DNS 模式可能为域名分配类似 `127.8.*.*` 的临时占位地址，这时按目标 IP 编写的 Proxifier 规则可能无法正常匹配。

通常可以先保留自动检测模式；只有遇到 DNS 解析失败或确实需要代理解析时，再强制启用该功能。

## 五、如何检查代理是否生效

Proxifier 主窗口会显示实时连接信息，包括：

- 发起连接的应用和进程；
- 目标域名、IP 与端口；
- 命中的规则；
- 使用的代理服务器；
- 连接成功、失败或超时状态。

启动目标程序后，观察 `Connections` 和日志区域。如果对应连接显示命中了指定规则并经过代理，说明配置已经生效。

还可以让浏览器通过该规则访问 IP 查询网站，对比代理前后的出口 IP。

## 六、常见问题

### 1. 代理测试成功，但程序仍然无法联网

程序可能主要使用 UDP，而 Proxifier 主要处理 TCP。游戏、语音通话及浏览器 QUIC/HTTP3 都可能遇到这种情况。浏览器可以尝试关闭 QUIC，使其回退到基于 TCP 的连接；如果软件高度依赖 UDP，则应考虑 TUN 模式或 VPN。

### 2. 日志中出现连接循环或大量失败

依次检查：

- 代理客户端自身是否被错误地再次代理；
- `127.0.0.1` 和 `localhost` 是否保持直连；
- 目标应用内部是否又配置了一层代理；
- Proxification Rules 的排列顺序是否正确。

### 3. 浏览器代理可用，但 Proxifier 检查失败

填写的代理可能只是普通 HTTP 代理，并不支持 CONNECT 隧道。普通 HTTP 代理能力有限，建议确认服务端支持 HTTPS CONNECT，或者改用 SOCKS5。

### 4. 开启代理后无法访问内网

将 `localhost`、私有 IP 地址段和公司内部域名加入 `Direct` 规则，并确保直连规则位于全局代理规则之前。

### 5. SOCKS5 是否等于加密连接

不等于。SOCKS5 主要负责流量转发，本身不保证传输内容加密。访问 HTTPS 网站时，网站流量仍由 HTTPS 保护；但本机到远程 SOCKS5 服务器之间的其他数据未必加密。使用远程代理时，应确保代理来源可信，并优先通过安全的加密隧道承载连接。

## 七、最小可用配置清单

正式使用前，可以按照下面的清单检查：

```text
□ 代理客户端已经运行
□ 已确认代理地址、端口和协议
□ 优先使用 SOCKS5
□ Proxifier 的 Check 测试已经通过
□ 已找到需要代理的程序 .exe
□ Localhost 与局域网地址保持直连
□ 目标程序没有配置重复代理
```

一个常见的最小配置如下：

```text
代理客户端：Clash / v2rayN
代理入口：127.0.0.1:7891
代理协议：SOCKS5

Proxifier 规则：
Localhost → Direct
指定程序  → SOCKS5 127.0.0.1:7891
Default   → Direct
```

完成这些设置后，Proxifier 就可以在不影响其他程序的前提下，让指定应用稳定地通过代理访问网络。
