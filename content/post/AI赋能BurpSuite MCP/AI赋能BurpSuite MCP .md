---
title: "AI赋能BurpSuite MCP "
description: "新版的Burp suite推出了MCP服务，可接入Claude、codex、以及trace的等进行AI赋能"
slug: 
date: 
image: 
categories:
    - AI
tags:
    - 
weight: 1
---







先在拓展商店中下载Burp suite的MCP

![](https://img.xiaoyuwell.top/PicGo/20260529124326087.png)

点击MCP查看配置

![](https://img.xiaoyuwell.top/PicGo/20260529124541980.png)

在拓展中查看是否已经启动

![](https://img.xiaoyuwell.top/PicGo/20260529124707359.png)

### 1.CC Switch

![](https://img.xiaoyuwell.top/PicGo/20260529123637626.png)

配置导向选择sse，配置你的MCP及监听端口

![](https://img.xiaoyuwell.top/PicGo/20260529124404614.png)

在CCS中配置好了启动Claude可检查是否已经连接，如果存在问题也可以让它自行配置

![](https://img.xiaoyuwell.top/PicGo/20260529124815588.png)

以上是接入Claude略显得麻烦，也可以在trace中进行配置可以更直观简单

### 2.Trace

在trace的设置中添加MCP即可

```
{
  "mcpServers": {
    "burp-suite": {
      "url": "http://127.0.0.1:9876"
    }
  }
}
```

![](https://img.xiaoyuwell.top/PicGo/20260529125158796.png)