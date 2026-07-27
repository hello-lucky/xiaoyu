---
title: "哥斯拉 WebShell 流量特征分析与蓝队检测"
description: "在隔离靶场中对哥斯拉 PHP XOR/Base64 样本开展环境搭建、Wireshark 抓包、协议解密、代码审计和蓝队检测验证。"
slug: 
date: 
image:
categories:
    - stu
tags:
    - Godzilla
    - WebShell
    - Wireshark
    - Suricata
    - 蓝队
weight: 1
---

## 0. 实验目标与结论摘要

本实验针对哥斯拉（Godzilla）PHP 动态载荷完成以下工作：

1. 搭建 phpStudy、Pikachu、本地 HTTP 代理和 Wireshark 回环抓包环境。
2. 从完整 PCAP 中定位文件上传、资源访问、测试连接、会话初始化、基础信息获取和命令执行。
3. 从 URI、参数、正文、响应边界、报文长度和时序中提取入侵检测特征。
4. 对服务端 PHP 样本进行代码审计，确认参数、密钥、Session 载荷及响应封装方式。
5. 完成请求和响应的 XOR/Base64/GZip 离线还原，解析控制过程。
6. 将实验结果转化为 Wireshark 过滤器、Suricata 规则、YARA 规则和蓝队处置要点。

本次抓包形成的关键证据链是：

```text
帧4971：浏览器向本地代理提交test1.jpg
    ↓
帧5772：代理向Web服务器转发时文件名变为test1.php
    ↓
帧5776：服务器返回HTTP 200及uploads/test1.php
    ↓
TCP流194：初始化载荷 → test → g_close
    ↓
TCP流214：初始化载荷 → test → getBasicsInfo
    ↓
TCP流229：execCommand请求 → Windows网络配置回显
```

需要区分三类证据：

| 类别 | 示例 | 适用范围 |
|---|---|---|
| HTTP 外层可见特征 | URI、`pass123=`、正文长度、固定响应边界 | 无需解密即可检测 |
| 解密后协议语义 | `test`、`g_close`、`getBasicsInfo`、`execCommand` | 用于调查与控制过程还原 |
| 可修改的样本 IOC | 文件名、密码参数、密钥、MD5边界 | 仅适用于相同配置样本 |

## 1. 搭建测试环境

### 1.1 实验拓扑

```text
浏览器 / 哥斯拉客户端
          │
          ├── HTTP代理：127.0.0.1:8080
          │
          └── Wireshark：Adapter for loopback traffic capture
                              │
                              ▼
                    Web服务：127.0.0.1:80
                              │
                              ▼
              phpStudy + Pikachu 本地PHP靶场
```

网站、客户端和抓包程序位于同一台 Windows 主机，通信不会经过 WLAN 网卡，因此 Wireshark 应选择 Npcap 提供的：

```text
Adapter for loopback traffic capture
```

抓包开始时不设置过窄的捕获过滤器，先保留完整回环流量，再通过显示过滤器定位目标，避免遗漏代理转发、TCP 重组或非标准端口数据。

### 1.2 软件与用途

| 组件 | 用途 |
|---|---|
| Windows + phpStudy | PHP Web 运行环境 |
| Pikachu | 本地文件上传靶场 |
| Wireshark / TShark | 抓包、TCP 重组、字段提取 |
| 本地 HTTP 代理 | 对比浏览器请求与后端请求 |
| Suricata | 网络检测规则验证 |
| YARA | Web 目录静态文件检测 |
| Sysmon / Windows 日志 | 监控 Web 服务进程派生系统命令 |
| CyberChef / 离线脚本 | 编码识别与解密验证 |

参考项目：

- [Godzilla 官方项目](https://github.com/BeichenDream/Godzilla)
- [Suricata HTTP 规则关键字](https://docs.suricata.io/en/latest/rules/http-keywords.html)
- [Zeek 文档](https://docs.zeek.org/en/current/)
- [CyberChef](https://gchq.github.io/CyberChef/)
- [NSA WebShell 检测与缓解项目](https://github.com/nsacyber/Mitigating-Web-Shells)

### 1.3 样本配置

靶场只能解析 PHP，因此实验选择 PHP 动态载荷：

```text
有效载荷：PhpDynamicPayload
加密器：PHP_XOR_BASE64
POST参数：pass123
Session键名：payload
XOR密钥：32250170a0dca92d
```

![](https://img.xiaoyuwell.top/PicGo/20260719221121622.png)

服务端文件扩展名必须与 Web 容器能力匹配。例如将 JSP 文件放入只支持 PHP 的站点时，可能显示源码或被作为静态文件下载，这不代表 JSP 已被执行。蓝队也可以利用“脚本扩展名与实际容器不匹配”发现配置错误或源码泄露。

### 1.4 抓包证据

| 项目 | 数值 |
|---|---|
| 文件 | `gsl.pcapng` |
| 抓包时间 | 2026-07-22 16:19:19 ～ 16:23:58 |
| 时长 | 279.678120 秒 |
| 数据包数量 | 约 12,000 |
| 文件大小 | 9,422,236 字节 |
| PCAP SHA-256 | `D9585F0770551D64E9049068D06360594D6A77DCE5F992C13038C66B0D09138D` |

为保证实验可复核，还应同时保存：

- 样本文件副本及 SHA-256；
- Web 访问日志和错误日志；
- 测试前后的 Web 目录文件清单；
- 操作时间记录；
- 管理客户端版本、载荷和加密器名称。

## 2. Wireshark 流量分析

### 2.1 工作流程

#### 第一步：确认 HTTP 服务与代理端口

```wireshark
http && (tcp.port == 80 || tcp.port == 8080)
```

如果 HTTP 运行在其他端口，应使用“分析 → 解码为 → HTTP”。不能仅因为 Protocol 列显示 TCP，就判断没有 HTTP 流量。

#### 第二步：流量过滤

```wireshark
http.request.url contains "claetcheck.php"
```



![](https://img.xiaoyuwell.top/PicGo/20260722185555053.png)

可添加以下列：

```text
frame.number
frame.time
tcp.stream
tcp.srcport
tcp.dstport
http.request.method
http.request.uri
http.content_length
http.response.code
```

#### 第三步：定位文件上传

```wireshark
http.request.method == "POST" &&
http.request.uri contains "clientcheck.php"
```

![](https://img.xiaoyuwell.top/PicGo/20260722185809149.png)

展开：

```text
MIME Multipart Media Encapsulation
└── Content-Disposition
```

记录 `filename`、multipart 字段名、声明的 `Content-Type`、文件真实开头、请求帧、响应帧以及代理前后的字段变化。

#### 第四步：定位上传后的资源访问

```wireshark
http.request.method == "GET" && http.request.uri contains "/uploads/test1.php"
```

GET 返回 `200` 且正文为空，只能证明资源可访问或被 PHP 引擎处理，不能单独证明已经发生控制行为。

#### 第五步：定位持续控制通信

```wireshark
http.request.method == "POST" && http.request.uri contains "/uploads/test1.php"
```

结合已知样本参数：

```wireshark
http.request.method == "POST" && http.request.uri contains "/uploads/test1.php" && tcp contains "pass123="
```

![](https://img.xiaoyuwell.top/PicGo/20260722200750398.png)

重点观察：

- 是否持续访问同一低访问量脚本；
- 参数名是否固定；
- 参数值是否为 Base64、URL 编码或高熵数据；
- 初始化请求与后续控制请求的长度差异；
- 响应是否具有稳定的首尾边界；
- 是否缺少正常页面的 Referer、Cookie 或浏览器访问链。

#### 第六步：关联请求与响应

选择请求帧，在 HTTP 协议树中查看：

```text
Response in frame
```

例如定位帧 10714 的响应：

```wireshark
http.request_in == 10714
```

由响应返回请求：

```wireshark
http.response_in == 10740
```

#### 第七步：追踪 TCP 流并关联主机证据

单独查看目标流：

```wireshark
tcp.stream in {194,214,229}
```

网络证据最终需要与以下主机证据闭环：

```text
Web访问日志中的URI与时间
Web目录中新建或修改的脚本
样本文件哈希及YARA命中
php-cgi/httpd/nginx派生cmd或powershell
命令输出与HTTP响应的时间和长度
```

### 2.2 上传阶段

#### 2.2.1 浏览器到代理

| 项目 | 内容 |
|---|---|
| 请求帧 | 4971 |
| 响应帧 | 5778 |
| TCP 流 | 105 |
| 方向 | `127.0.0.1:19693 → 127.0.0.1:8080` |
| URI | `http://localhost/pikachu-master/vul/unsafeupload/clientcheck.php` |
| Content-Length | 1170 |
| 文件名 | `test1.jpg` |
| 声明类型 | `image/jpeg` |
| 文件数据 | 876 字节 |

![](https://img.xiaoyuwell.top/PicGo/20260722214507699.png)

![](https://img.xiaoyuwell.top/PicGo/20260722214538051.png)

#### 2.2.2 代理到 Web 服务器

| 项目 | 内容 |
|---|---|
| 请求帧 | 5772 |
| 响应帧 | 5776 |
| TCP 流 | 131 |
| 方向 | `127.0.0.1:60648 → 127.0.0.1:80` |
| URI | `/pikachu-master/vul/unsafeupload/clientcheck.php` |
| Content-Length | 1170 |
| 文件名 | `test1.php` |
| 声明类型 | `image/jpeg` |
| 文件数据 | 876 字节 |

![](https://img.xiaoyuwell.top/PicGo/20260722214617973.png)

![](https://img.xiaoyuwell.top/PicGo/20260722214635884.png)

两个请求使用相同 multipart boundary，文件数据长度和内容一致，但文件名从 `test1.jpg` 变为 `test1.php`。帧 5776 返回 HTTP 200，HTML 中出现：

```text
uploads/test1.php
```

从帧 5772 提取的文件：

```text
大小：876字节
SHA-256：B4F29F6E4384C05E2958BD379EE3C44FEE071822EAA28245D6D9ED78309C0D56
声明类型：image/jpeg
真实开头：<?php
```

![](https://img.xiaoyuwell.top/PicGo/20260722190100502.png)

该证据说明服务端没有对最终文件名、真实内容和上传目录执行权限进行联合控制。

### 2.3 上传后的访问

| 请求帧 | 响应帧 | TCP流 | 方向 | 结果 |
|---:|---:|---:|---|---|
| 6408 | 6498 | 106 | 浏览器→代理→浏览器 | HTTP 200，正文长度0 |
| 6494 | 6496 | 152 | 代理→Web服务器→代理 | HTTP 200，正文长度0 |

过滤：

```wireshark
frame.number in {6408,6494,6496,6498}
```

空响应与样本源码一致：普通 GET 没有携带 `pass123` 参数，不会进入处理分支。

### 2.4 测试连接

测试连接位于 TCP 流 194：

```wireshark
tcp.stream == 194
```

| 请求帧 | 响应帧 | 请求体长度 | HTTP结果 | 解密结果 |
|---:|---:|---:|---|---|
| 8699 | 8701 | 52538 | 200，响应长度0 | 初始化动态载荷 |
| 8705 | 8707 | 40 | 200，响应体64字节 | `methodName=test` |
| 8759 | 8761 | 44 | 200，响应体64字节 | `methodName=g_close` |

行为顺序：

```text
大型动态载荷初始化
→ test测试
→ g_close关闭测试会话
```

首次请求只把动态载荷写入 PHP Session，不直接输出数据，因此帧 8701 的 `Content-Length` 为 0。

### 2.5 正式会话初始化

正式会话位于 TCP 流 214：

```wireshark
tcp.stream == 214
```

| 请求帧 | 响应帧 | 请求体长度 | HTTP结果 | 解密结果 |
|---:|---:|---:|---|---|
| 9847 | 9849 | 52538 | 200，响应长度0 | 初始化动态载荷 |
| 9853 | 9855 | 40 | 200，响应体64字节 | `methodName=test` |
| 9871 | 9873 | 52 | 200，响应体1208字节 | `methodName=getBasicsInfo` |

行为顺序：

```text
大型动态载荷初始化
→ test验证
→ getBasicsInfo获取基础信息
```

测试连接和正式会话都出现约 52 KB 的初始化请求，但最后一个方法不同：测试连接以 `g_close` 结束，正式会话继续执行 `getBasicsInfo`。

### 2.6 命令执行

命令执行位于 TCP 流 229：

```wireshark
tcp.stream == 229
```

| 项目 | 内容 |
|---|---|
| 请求帧 | 10714 |
| 响应帧 | 10740 |
| 请求时间 | 2026-07-22 16:23:21.244529 |
| 请求体长度 | 218 字节 |
| 响应编码体长度 | 1612 字节 |
| 解密后响应长度 | 5811 字节 |
| HTTP状态 | 200 OK |
| 请求至响应时间 | 约112.7毫秒 |
| 解密方法字段 | `methodName=execCommand` |

![](https://img.xiaoyuwell.top/PicGo/20260723115008140.png)

帧 10714 解密后确认执行的是 Windows 网络配置查询，帧 10740 解密后包含：

```text
Windows IP 配置
主机名 DESKTOP-V6FOL6Q
Realtek PCIe GbE Family Controller
VMware Network Adapter VMnet1
VMware Network Adapter VMnet8
IPv4地址、DHCP服务器及其他适配器信息
```

![](https://img.xiaoyuwell.top/PicGo/20260722201255202.png)

原始 HTTP 正文中不能直接搜索命令明文，因为命令参数经过 GZip、循环 XOR、Base64 和 URL 编码。网络检测应先识别通信外层，协议解密用于后续调查确认。

## 3. 定义流量特征字符串

### 3.1 样本专用 IOC

```text
上传URI：/pikachu-master/vul/unsafeupload/clientcheck.php
控制URI：/pikachu-master/vul/unsafeupload/uploads/test1.php
POST参数：pass123=
响应前缀：dc3c56c78ad0d757
响应后缀：107d6cf7f72a8d20
文件SHA-256：B4F29F6E4384C05E2958BD379EE3C44FEE071822EAA28245D6D9ED78309C0D56
```

这些值来自本次样本配置。修改文件名、路径、密码或密钥后，IOC 会随之变化，不能将其视为所有哥斯拉样本的通用特征。

### 3.2 上传行为特征

高置信度组合：

```text
filename="test1.php"
AND Content-Type: image/jpeg
AND 文件内容以 <?php 开头
```

静态代码字符串：

```text
@session_start()
@set_time_limit(0)
@error_reporting(0)
base64_decode($_POST
eval($payload)
getBasicsInfo
@run($data)
```

单独匹配 `image/jpeg`、Base64 或 `<?php` 容易产生误报，应将文件名、MIME、真实内容和代码语义组合。

### 3.3 通信外层特征

```text
POST /pikachu-master/vul/unsafeupload/uploads/test1.php
Content-Type: application/x-www-form-urlencoded
请求体以 pass123= 开始
参数值呈Base64/URL编码形式
```

所有有效响应均采用相同边界：

```text
dc3c56c78ad0d757
+ Base64编码数据
+ 107d6cf7f72a8d20
```

该边界在帧 8707、8761、9855、9873 和 10740 中重复出现，是本实验样本的强网络指纹。

### 3.4 通用行为特征

比固定字符串更稳定的行为包括：

```text
上传目录内的脚本文件持续收到POST
首次初始化请求明显大于后续请求
短时间出现多个40～218字节的小型控制请求
请求和响应为高熵或编码数据
响应在单个会话配置中具有稳定首尾结构
同一低访问量URI连续承担信息收集和命令操作
Web服务进程派生系统命令解释器
```

推荐的告警评分：

| 特征 | 建议权重 |
|---|---:|
| 上传目录内脚本收到POST | 高 |
| 脚本伪装成图片上传 | 高 |
| Web服务进程派生命令解释器 | 极高 |
| 固定响应边界同时匹配 | 高 |
| 单纯Base64或高熵 | 低 |
| 单纯`application/x-www-form-urlencoded` | 低 |

## 4. 检测应用

### 4.1 Wireshark 过滤器

定位上传：

```wireshark
http.request.method == "POST" &&
http.request.uri contains "clientcheck.php"
```

定位全部目标 POST：

```wireshark
http.request.method == "POST" && http.request.uri contains "/uploads/test1.php"
```

结合参数：

```wireshark
http.request.method == "POST" && http.request.uri contains "/uploads/test1.php" && tcp contains "pass123="
```

定位固定响应：

```wireshark
http.file_data contains "dc3c56c78ad0d757" &&
http.file_data contains "107d6cf7f72a8d20"
```

定位伪装上传：

```wireshark
http.request.method == "POST" && tcp contains "filename=" && tcp contains "test1.php" && tcp contains "Content-Type: image/jpeg"
```

### 4.2 Suricata 实验规则

以下规则用于本实验 PCAP 验证。投入生产前应调整 `$HOME_NET`、URI、SID、正文检查上限和误报白名单。

检测伪装成 JPEG 的 PHP 上传：

```suricata
alert http any any -> $HOME_NET any (
    msg:"LAB PHP file disguised as JPEG upload";
    flow:established,to_server;
    http.method; content:"POST";
    http.uri; content:"/unsafeupload/clientcheck.php";
    http.request_body;
    content:"filename=\"test1.php\""; nocase;
    content:"Content-Type: image/jpeg"; nocase;
    content:"<?php";
    classtype:web-application-attack;
    sid:1000001; rev:1;
)
```

检测上传目录中的可疑 PHP POST：

```suricata
alert http any any -> $HOME_NET any (
    msg:"LAB suspicious POST to PHP file in upload directory";
    flow:established,to_server;
    http.method; content:"POST";
    http.uri; content:"/uploads/test1.php"; endswith;
    http.request_body; content:"pass123="; startswith;
    classtype:web-application-attack;
    sid:1000002; rev:1;
)
```

检测固定响应边界：

```suricata
alert http $HOME_NET any -> any any (
    msg:"LAB Godzilla fixed framed response";
    flow:established,to_client;
    http.response_body;
    content:"dc3c56c78ad0d757"; startswith;
    content:"107d6cf7f72a8d20"; endswith;
    classtype:trojan-activity;
    sid:1000003; rev:1;
)
```

`http.request_body` 和 `http.response_body` 的可检查大小受 Suricata 配置中的 `request-body-limit` 与 `response-body-limit` 影响。规则未告警时，应先确认正文是否被完整检查。

### 4.3 YARA 静态检测

```yara
rule Lab_PHP_Godzilla_Like_WebShell
{
    meta:
        description = "Detects the audited PHP XOR/Base64 session webshell"
        scope = "authorized lab"

    strings:
        $php = "<?php" ascii
        $s1  = "@session_start" ascii
        $s2  = "base64_decode($_POST" ascii
        $s3  = "eval($payload)" ascii
        $s4  = "getBasicsInfo" ascii
        $s5  = "@run($data)" ascii

    condition:
        uint16(0) == 0x3f3c and 4 of ($s*)
}
```

生产环境应使用“已知良好文件基线 + 新增/修改文件检测 + YARA”组合，而不是只扫描固定文件名。

## 5. 加密流量解密

### 5.1 密钥获取

蓝队获取密钥的推荐顺序：

1. 隔离并复制可疑服务端脚本，进行静态代码审计。
2. 检查客户端配置、测试记录或合法授权材料。
3. 根据公开实现确认算法、字符编码、压缩及响应封装顺序。
4. 使用已知明文验证推导结果，避免无边界暴力破解。

本次样本中：

```php
$pass='pass123';
$payloadName='payload';
$key='32250170a0dca92d';
```

三个变量分别表示：

| 变量 | 作用 |
|---|---|
| `$pass` | HTTP POST 参数名，即 `$_POST['pass123']` |
| `$payloadName` | PHP Session 中缓存动态载荷的键名 |
| `$key` | 请求和响应使用的16字符循环XOR密钥 |

密钥值为密码 MD5 结果的前 16 个十六进制字符：

```text
md5("pass123")
= 32250170a0dca92d53ec9624f336ca24

key
= 32250170a0dca92d
```

响应边界来自：

```text
md5(pass + key)
= dc3c56c78ad0d757107d6cf7f72a8d20
```

所以：

```text
前16字符：dc3c56c78ad0d757
后16字符：107d6cf7f72a8d20
```

![](https://img.xiaoyuwell.top/PicGo/20260722191302936.png)

### 5.2 代码审计得到的协议流程

服务端逻辑可抽象为：

```text
读取POST[pass123]
→ Base64解码
→ 使用16字符密钥循环XOR
→ 首次请求把动态载荷存入Session
→ 后续请求从Session恢复并加载载荷
→ run(data)处理控制参数
→ 结果循环XOR
→ Base64编码
→ 添加固定MD5前后边界
```

循环索引：

```text
key[(i + 1) & 15]
```

XOR 可逆，因此请求和响应使用相同函数处理。

### 5.3 离线解密代码

处理 Wireshark“追踪 HTTP 流”导出的请求值或响应正文。

```python
import base64
import gzip
import hashlib
from urllib.parse import unquote_plus

PASSWORD = "pass123"
KEY = hashlib.md5(PASSWORD.encode()).hexdigest()[:16]


def xor_cycle(data: bytes, key: str = KEY) -> bytes:
    key_bytes = key.encode()
    return bytes(
        byte ^ key_bytes[(index + 1) & 15]
        for index, byte in enumerate(data)
    )


def decode_request(form_body: str) -> bytes:
    """Decode pass123=<URL/Base64/XOR/GZip data>."""
    _, encoded = form_body.split("=", 1)
    encrypted = base64.b64decode(unquote_plus(encoded))
    plaintext = xor_cycle(encrypted)
    if plaintext.startswith(b"\x1f\x8b"):
        plaintext = gzip.decompress(plaintext)
    return plaintext


def decode_response(body: str) -> bytes:
    """Verify response markers, then decode Base64/XOR/GZip data."""
    marker = hashlib.md5((PASSWORD + KEY).encode()).hexdigest()
    prefix, suffix = marker[:16], marker[16:]

    if not body.startswith(prefix) or not body.endswith(suffix):
        raise ValueError("response marker mismatch")

    encrypted = base64.b64decode(body[16:-16])
    plaintext = xor_cycle(encrypted)
    if plaintext.startswith(b"\x1f\x8b"):
        plaintext = gzip.decompress(plaintext)
    return plaintext
```

正确处理顺序：

```text
请求：URL解码 → Base64解码 → XOR → 可选GZip解压 → 字段解析
响应：校验边界 → 去除边界 → Base64解码 → XOR → 可选GZip解压
```

如果顺序错误，常见表现是 Base64 格式错误、解密结果无固定字段或 GZip magic 不正确。

### 5.4 控制过程还原

| 阶段 | TCP流 | 请求帧 | 解密结果 |
|---|---:|---:|---|
| 测试连接 | 194 | 8705 | `methodName=test` |
| 关闭测试 | 194 | 8759 | `methodName=g_close` |
| 会话测试 | 214 | 9853 | `methodName=test` |
| 基础信息 | 214 | 9871 | `methodName=getBasicsInfo` |
| 命令执行 | 229 | 10714 | `methodName=execCommand` |

帧 10740 经边界剥离、Base64 解码、XOR 和 GZip 解压后为 5811 字节，内容与实验操作产生的 Windows 网络配置回显一致。

## 6. 代码审计

### 6.1 上传功能审计

帧 4971 与 5772 表明，同一个 876 字节文件在代理前名为 `test1.jpg`，代理后名为 `test1.php`，服务端仍接受 `Content-Type: image/jpeg`。由此可以判断：

```text
安全判断主要位于客户端
服务端未对最终文件名执行等价校验
服务端信任客户端提交的MIME
没有验证真实文件内容
上传目录允许PHP脚本执行
```

修复建议：

1. 所有上传校验必须在服务端完成。
2. 使用允许列表限制规范化后的最终扩展名。
3. 使用 `finfo`、图像解析和重新编码验证真实图片内容。
4. 由服务端生成随机文件名。
5. 将上传目录放在 Web 根目录之外。
6. 如必须公开访问，关闭上传目录中的脚本执行权限。
7. 记录原始文件名、最终文件名、哈希、用户、源地址和处理结果。

### 6.2 服务端样本审计

关键数据流：

```text
外部输入 $_POST[$pass]
→ base64_decode
→ encode循环XOR
→ Session动态载荷
→ eval($payload)
→ run($data)
```

危险点：

| 代码行为 | 风险 |
|---|---|
| `@error_reporting(0)` | 隐藏运行异常，降低可见性 |
| `@set_time_limit(0)` | 支持长时间执行 |
| `$_POST[$pass]` | 从固定POST参数接收控制数据 |
| `base64_decode` + XOR | 混淆通信内容 |
| `$_SESSION[$payloadName]` | 在Session中缓存动态载荷 |
| `eval($payload)` | 直接执行动态PHP代码 |
| `@run($data)` | 将控制操作交给动态载荷 |
| 固定MD5边界 | 帮助客户端定位正文，同时形成检测指纹 |

代码层面的核心恶意语义不是 Base64 或 XOR，而是“不可信输入经过还原后进入动态执行，并能够调用系统能力”。静态检测应结合数据流、危险函数和持久会话，避免对正常编码业务产生大量误报。

## 7. 蓝队操作点

### 7.1 网络侧

- 在反向代理、WAF 或 TLS 终止点保留 HTTP 请求元数据。
- 对上传目录中的脚本 POST 建立高优先级告警。
- 统计每个 URI 的客户端数、User-Agent 数和 Referer 分布，寻找低频孤立端点。
- 将正文熵和长度序列作为评分项，不单独定性。
- 使用 Suricata 检查上传正文、固定参数和响应边界。
- 使用 Zeek 或 SIEM 聚合“同一 URI 短时间连续 POST”。
- 将请求帧、响应帧、TCP流、时间、URI和正文长度写入调查记录。

### 7.2 主机侧

重点监控以下进程链：

```text
php-cgi.exe / httpd.exe / nginx.exe → cmd.exe
php-cgi.exe / httpd.exe / nginx.exe → powershell.exe
```

即使执行的是普通网络信息查询，只要父进程是 Web 服务进程，也应作为高风险行为调查。

文件侧：

- 对 Web 根目录和上传目录建立已知良好哈希基线；
- 实时监控新增或修改的 `.php` 文件；
- 检查图片、压缩包和临时文件中的脚本内容；
- 使用 YARA 检测动态执行、编码和 Session 缓存组合；
- 保存创建时间、修改时间、所有者、ACL 和哈希；
- 将 PCAP 时间与 Web 日志、Sysmon Event ID 1 和文件事件关联。

### 7.3 应急响应

发现高置信度 WebShell 后：

1. 隔离受影响服务器或限制入口访问。
2. 保存 PCAP、Web 日志、进程、网络连接、内存和可疑文件副本。
3. 计算证据哈希并建立时间线。
4. 搜索同目录、同哈希、同 URI 和相同代码片段。
5. 排查 Web 服务账号权限、计划任务、启动项和横向连接。
6. 修补上传功能或其他初始入口。
7. 从可信镜像恢复，不能把删除单个脚本视为处置完成。
8. 轮换应用密钥、数据库凭据和服务账号凭据。
9. 将 IOC 和行为特征回灌至 NDR、WAF、SIEM、EDR 与文件完整性监控。

## 8. 总结

本实验从文件上传、HTTP 通信、协议解密、代码语义和主机行为五个视角还原了哥斯拉 PHP XOR/Base64 样本的控制过程：

```text
帧4971 → 5772：文件名从test1.jpg变为test1.php
TCP流194：初始化载荷 → test → g_close
TCP流214：初始化载荷 → test → getBasicsInfo
TCP流229：execCommand → Windows网络配置回显
```

样本专用检测可以使用：

```text
URI=/uploads/test1.php
+ POST参数pass123=
+ 响应前缀dc3c56c78ad0d757
+ 响应后缀107d6cf7f72a8d20
```

更稳定的检测思路是：

```text
脚本伪装上传
+ 上传目录可执行
+ 异常脚本持续POST
+ 大初始化请求后出现多个小请求
+ Web服务进程派生系统命令
```

网络规则用于快速发现，代码审计用于确认恶意语义，协议解密用于还原控制过程，主机遥测用于证明执行，文件基线和应急处置用于最终闭环。
