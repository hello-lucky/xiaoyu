---
title: "codex+性价比之王deepseek V4-pro"
description: 
slug: 
date: 
image: 
categories:
    - AI
tags:
    - 
weight: 1
---

#### 1.桌面版codex安装

在这之前需要自行安装号Node.j环境

 OpenAI 已于 2026 年 3 月正式发布 Codex Windows 桌面版，推荐Microsoft Store 安装：winget install OpenAI.Codex或者直接在 Microsoft Store 搜索 "Codex by OpenAI" 安装。

#### 2.核心问题说明

Codex 桌面版使用 OpenAI Responses API（/v1/responses），而 DeepSeek 只提供 Chat Completions API（/v1/chat/completions）。两者协议不兼容，直接填 DeepSeek 的 Key 会报 400/404 错误。

 解决方案：在本机运行一个协议转换代理（CCX + CC-Switch），将 Codex 的请求翻译为 DeepSeek 能理解的格式。

  架构如下：

  Codex 桌面版  →  CC-Switch (本地)  →  CCX (本地 :3000)  →  api.deepseek.com

#### 3.安装cc-switch

  CC-Switch 负责拦截 Codex 的请求并转发到 CCX

这是一位博主的飞书，记载着详细的内容

[‌‍⁠‍‌⁠‌‍‬﻿﻿‬﻿⁠⁠﻿⁠‬‬‌‬‌⁠‬﻿‍‌‍Deepseek for claude code/codex - 飞书云文档](https://my.feishu.cn/wiki/Drh6wGaWsifdIGkvlx3c115OnEe)

#### 4.安装并配置ccx

在 CCX 目录下创建 .env 文件：

  PROXY_ACCESS_KEY=123456（你设置的一个密码）
  PORT=3000
  ENABLE_WEB_UI=true
  APP_UI_LANGUAGE=en

启动 CCX，打开浏览器访问 http://localhost:3000，在 Web UI 中添加渠道：

![](https://img.xiaoyuwell.top/PicGo/20260527150046695.png)

![](https://img.xiaoyuwell.top/PicGo/20260527144753811.png)

![](https://img.xiaoyuwell.top/PicGo/20260527145015264.png)

点击详细配置

![](https://img.xiaoyuwell.top/PicGo/20260527145139392.png)

![](https://img.xiaoyuwell.top/PicGo/20260527145222698.png)

然后添加渠道即可

#### 5.配置 cc-switch

![](https://img.xiaoyuwell.top/PicGo/20260527145457862.png)

![](https://img.xiaoyuwell.top/PicGo/20260527145848681.png)

虽然在cc-switch中对它进行测试是被拒的，但是可以使用

![](https://img.xiaoyuwell.top/PicGo/20260527150527203.png)

这时你只需要重启codex，选择API登录，输入123456，也就是env中的PROXY_ACCESS_KEY。

![](https://img.xiaoyuwell.top/PicGo/20260527150850068.png)

#### 注意

在使用前都需要启动ccx

