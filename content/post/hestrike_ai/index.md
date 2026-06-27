---
title：hestrike_ai
decription:"项目解析"
categories:
    - AI
tags:
    -
---





## 一、目录

E:\pentest\github\hexstrike-ai-master
├── hexstrike-ai-mcp.json           # MCP 客户端配置模板，用于 Claude、Cursor、Roo Code 等接入
├── hexstrike_mcp.py                # MCP 桥接层，把 HexStrike API 包装成 AI 可调用的 MCP 工具
├── hexstrike_server.py             # 项目主服务端，提供 HTTP API 并实际执行安全工具和系统命令
├── LICENSE                         # MIT 开源许可证
├── README.md                       # 项目说明文档，包含安装、功能、API、使用方式和安全提醒
└── requirements.txt                # Python 依赖列表，安装 Flask、FastMCP、Selenium 等库



## 二、和strike-ai-mcp.json

MCP客户端配置模板，本身不会运行项目，而是告诉MCP客户端“启动hestrike_mcp.py,然后让它连接Hestrike后端服务”

```
AI 客户端
  读取 hexstrike-ai-mcp.json
        |
        v
启动 hexstrike_mcp.py
        |
        v
hexstrike_mcp.py 连接 http://IPADDRESS:8888
        |
        v
hexstrike_server.py 提供真正的工具执行能力
```

mcpservers：MCP配置顶层字段，”这里定义了一组MCP服务“

```json
"mcpServers": { ... }
```

一个客户端可以配置多个MCP服务，这里配置了一个

hestrike-ai

```json
"hestrike-ai:{...}"
```

AI 客户端会把它显示成一个叫 `hexstrike-ai` 的工具服务器

cmmand:表示启动MCP是要执行的命令是python3

```json
"command": "python3"
```

args给command的参数，拼接执行

```
python3 /path/hexstrike_mcp.py --server http://IPADDRESS:8888
```

总结核心作用：

1. 启动 hexstrike_mcp.py
2. 告诉 hexstrike_mcp.py 去连接哪个 hexstrike_server.py

```json
{
  "mcpServers": {
    "hexstrike-ai": {
      "command": "python3",
      "args": [
        "/path/hexstrike_mcp.py",
        "--server",
        "http://IPADDRESS:8888"
      ],
      "description": "HexStrike AI v6.0 - Advanced Cybersecurity Automation Platform. Turn off alwaysAllow if you dont want autonomous execution!",
      "timeout": 300,
      "alwaysAllow": []
    }
  }
}
```



## 三、hestrike_mcp.py--MCP桥接层

```
AI 客户端
Claude / Cursor / Copilot / Roo Code
        |
        | 调用 MCP 工具
        v
hexstrike_mcp.py
把 MCP 工具调用转换成 HTTP 请求
        |
        | 请求 API
        v
hexstrike_server.py
真正执行 nmap、sqlmap、nuclei 等工具
```

![](https://img.xiaoyuwell.top/PicGo/20260627163447944.png)

可以理解为翻译器；
AI 说：我要调用 nmap_scan()
hexstrike_mcp.py 翻译成：POST /api/tools/nmap
hexstrike_server.py 执行：nmap ...



## 四、hestrike_serevr.py--后端

是一个 Flask API 服务端。它接收 HTTP 请求，然后在本机执行安全工具、系统命令、文件操作、AI 参数选择、CTF/漏洞情报工作流等任务。

和hestrike_mcp.py的关系：

```
AI 客户端
  -> hexstrike_mcp.py
    -> HTTP 请求
      -> hexstrike_server.py
        -> 执行 nmap / nuclei / sqlmap / hydra / 文件操作 / 系统命令
```

总结：

```
启动 Flask API
接收 MCP 转发来的请求
检测工具是否存在
执行系统命令
运行安全扫描工具
管理缓存和进程
生成工作流和攻击链
返回扫描结果
```



## 五、总结

核心组件：

```
hexstrike_server.py   # 后端服务，真正执行命令和安全工具
hexstrike_mcp.py      # MCP 桥接层，让 AI 客户端能调用后端能力
```

配置文件：

```
hexstrike-ai-mcp.json # 告诉 AI 客户端如何启动 hexstrike_mcp.py
```

整体链路：

```
用户 / AI 客户端
        |
        v
Claude / Cursor / Roo Code / Copilot
        |
        v
hexstrike_mcp.py
        |
        v
http://127.0.0.1:8888
        |
        v
hexstrike_server.py
        |
        v
nmap / nuclei / sqlmap / gobuster / hydra / 文件操作 / 系统命令
```

流程：

```
1. AI 客户端调用 MCP 工具 nmap_scan -->nmap_scan(target="127.0.0.1", scan_type="-sV", ports="80,443")
2. hexstrike_mcp.py 接收参数
3. hexstrike_mcp.py 组装 JSON   -->nmap -sV -p 80,443 127.0.0.1
4. hexstrike_mcp.py 请求 POST /api/tools/nmap
5. hexstrike_server.py 接收请求
6. hexstrike_server.py 拼接命令
7. EnhancedCommandExecutor 执行命令
8. 收集 stdout / stderr / return_code
9. 返回 JSON 给 MCP
10. MCP 返回结果给 AI
11. AI 总结给用户
```

总结

```
AI 客户端通过 MCP 调用 hexstrike_mcp.py，
hexstrike_mcp.py 把调用转成 HTTP 请求，
hexstrike_server.py 收到请求后执行本机安全工具或系统命令，再把结果一路返回给 AI。
```



## 六、部署--trace

```
Sudo -i
git clone https://github.com/0x4m4/hexstrike-ai.git
```

![](https://img.xiaoyuwell.top/PicGo/20260627180018944.png)

```
cd hexstrike-ai
ls
```

![](https://img.xiaoyuwell.top/PicGo/20260627180047885.png)

打开hexstrike-ai文件，依次执行里面的命令

![](https://img.xiaoyuwell.top/PicGo/20260627180122197.png)

![](https://img.xiaoyuwell.top/PicGo/20260627180135278.png)

![](https://img.xiaoyuwell.top/PicGo/20260627180209222.png)

服务端配置完成，接下来配置客户端

打开trace的设置进行MCP的配置

![](https://img.xiaoyuwell.top/PicGo/20260627180810883.png)

```
{
  "mcpServers": {
    "hexstrike-ai": {
      "command": "python",
      "args": [
        "path\\hexstrike-ai-master\\hexstrike_mcp.py",  --找到hexstrike_mcp.py在window中的路径进行更爱
        "--server",
        "http://10.50.97.167:8888"  
      ]
    }
  }
}
```

