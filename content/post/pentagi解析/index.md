---
title: pentagi
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



# Pentagi解析

## 一、简介

pentagi是一个AI自动化渗透测试平台：用户 在网页上创建一个测试任务，让多个agent分工思考、搜索、执行命令，并把结果保存、展示、生成报告。

渗透测试流程：

1.确定目标范围。

2.收集目标信息。

3.扫描端口、服务、网站目录。

4查找漏洞。

5.尝试利用漏洞。

6.记录证据。

7.编写报告。

PentAGI 想把这些步骤自动化。它会调用大模型，比如 OpenAI、Anthropic、Gemini、Ollama 等，让 AI 扮演不同角色：

Researcher：负责搜索、收集信息。

Developer：负责分析、规划攻击路径或写辅助脚本。

Executor：负责执行工具命令，比如 `nmap`、`sqlmap` 等。

Reporter：负责总结结果，生成报告。

Assistant：用户交互式助手，可以帮你查看、解释、继续任务。

是一个完整的系统：有网页、有后端、有数据库、有Docker沙箱、有日志、有实时跟新、有报告。

## 二、Linux部署

1.创建工作目录

```
mkdir pentagi && cd pentagi
```

2.下载配置文件：

```
curl -O https://raw.githubusercontent.com/vxcontrol/pentagi/master/.env.example
curl -O https://raw.githubusercontent.com/vxcontrol/pentagi/master/docker-compose.yml
```

3.**配置环境**：复制 `.env.example` 为 `.env`，填入 API 密钥与各项设置。

```
cp .env.example .env
# 编辑 .env
nano .env
```

4.拉取镜像、启动

```
docker compose up -d
```

5.访问web页面：

```
https://localhost:8443
```

初始账号密码：

```
admin@pentagi.com/admin
```

## 二、项目运行

1.流程

```
用户浏览器
  ↓
React 前端
  ↓
Gin REST / GraphQL API
  ↓
Flow / Task / Subtask Controller
  ↓
FlowProvider 多智能体调度层
  ↓
ToolsExecutor 工具执行层
  ↓
Docker 沙箱 / 搜索引擎 / 向量库 / 数据库 / LLM Provider
```

打开网页

--->网页请求后端接口

---->后端保存用户、任务、日志、报告到数据库

--->后端调用大模型，让AI思考下一步

--->后端通过Docker创建环境，工具在隔离环境中跑

--->工具输出回到后端--->后端把结果实时推送到前端

--->前端展示进度、日志、终端输出、报告

3.图关系

![](https://img.xiaoyuwell.top/PicGo/20260615121226920.png)

4.各agent之间的关系

```
Primary Agent：总调度,理解当前子任务目标--->进行任务委派给对应的agent--->接受agent结果--->综合结果--->继续下一步--->最后通过done工具提交结果
  ├── 调用 Searcher：查资料、漏洞、文档、PoC
  ├── 调用 Pentester：执行安全测试和验证
  ├── 调用 Coder：写脚本、改代码、生成工具
  ├── 调用 Installer：安装工具、配置环境
  ├── 调用 Memorist：查历史经验和长期记忆
  └── 调用 Adviser：遇到复杂问题时请求策略建议

Searcher Agent--信息收集，查互联网、漏洞信息、CVE、exploit、技术文档--->结果交付其他agent
  ├── 可调用搜索引擎
  ├── 可调用 Browser
  ├── 可查 Memory
  └── 可存储搜索答案

Pentester Agent--渗透测试：侦察、安全测试、漏洞验证、执行工具--->通过hack_result返回结果
  ├── 可调用 Searcher
  ├── 可调用 Coder
  ├── 可调用 Installer
  ├── 可调用 Memorist
  ├── 可调用 Adviser
  └── 可调用 Terminal/File/Browser

Coder Agent--脚本开发：写脚本、payload，编写自动化检测脚本，使用终端工具运行代码
  ├── 可调用 Searcher
  ├── 可调用 Installer
  ├── 可调用 Memorist
  ├── 可调用 Adviser
  └── 可调用 Terminal/File
  
Enricher Agent--上下文增强：在 Adviser 给建议前补充背景、查记忆
  └──为Adviser Agent整理材料

Adviser Agent--策略
  └── 通常先由 Enricher 补充上下文，再给建议

Memorist Agent--长期记忆
  ├── 查历史任务
  ├── 查当前执行上下文
  ├── 查向量库
  └── 查 Graphiti 知识图谱

Generator Agent--子任务生成：总任务拆分--->子任务
  └── 初始拆解 Task 为 Subtasks

Refiner Agent--子任务调度：删除不必要、新增后续、修改已有子任务、调整顺序、根据执行结果更新计划
  └── 根据执行结果动态修改 Subtasks

Reporter Agent--结果总结：汇总任务执行过程、生成.md报告、结果保存到数据库、前端展示
  └── 最后生成任务报告

Reflector Agent--纠错：agent工具调度不正确、输出不合预期
  └── 纠正 Agent 不调用工具、快超限、格式不合规等问题

Summarizer Agent--上下文压缩：压缩长工具输出、总结历史对话、保留关键上下文
  └── 压缩长上下文和长工具输出
  
 Assistant Agent--面向用户交互
  └──可调用agent团队的交互式助手
```

5.一次完整任务执行

```
对某个授权目标做 Web 安全测试
```

系统内部运行

![](https://img.xiaoyuwell.top/PicGo/20260615121331764.png)

## 三、前端分析

基于React+Typecript+Vite 的AI工作台

负责登录、创建flow、查看agent执行过程、实时终端输出、查看任务进度、文件资源、知识库、LLM配置、API监控、报告输出等

核心入口

```
frontend/src/main.tsx
frontend/src/app.tsx
frontend/package.json
```

技术栈

构建工具：Vite--前端项目启动器和打包器

```
pnpm run dev

启动本地开发服务器
让浏览器能访问前端页面
监听代码变化，自动刷新页面
把 TypeScript、React、CSS 等内容转换成浏览器能运行的代码
生产环境时把代码压缩打包

frontend/vite.config.ts
Vite = 前端项目的发动机 + 打包工具
```

前端框架：React--前端框架

```
传统网页可能是直接写 HTML：
   <button>登录</button>

React 则把页面拆成一个个“组件”：
    function LoginButton() {
        return <button>登录</button>;
    }
    
React组件   
login-form.tsx
flow.tsx
dashboard.tsx
button.tsx
dialog.tsx
......
页面可以拆成小组件
组件可以复用
状态变化后页面会自动更新
适合复杂后台系统
比如 Flow 页面中，任务状态、终端日志、AI 消息不断变化，React 可以根据数据变化自动重新渲染页面。
React = 用组件搭页面的工具
```

语言：TypeScript--带类型的JavaScript

```
普通 JavaScript：
function add(a, b) {
  return a + b;
}

TypeScript：
function add(a: number, b: number): number {
  return a + b;
}
使用不同类型约束变量
TypeScript = 更安全、更适合大型项目的 JavaScript
```

路由：react-router-dom--前端页面跳转

```
后端路由：决定接口怎么响应
前端路由：决定当前 URL 显示哪个页面组件
```

GraphQL：Apollo Client--API查询，前端使用 GraphQL 的客户端工具

```
发送 GraphQL 请求
缓存查询结果
生成 React Hook
管理 loading/error/data 状态
处理实时订阅数据
```

实时通信：GraphQL WebSocket subscriptions

```
WebSocket：
连接建立后不断开
后端有新消息可以主动推给前端
前端不用一直问
--->从而实现实时更新
GraphQL WebSocket subscriptions = 后端主动推送实时数据给前端
```

EST 请求：Axios--常用的 HTTP 请求库

```
例如：
api.get('/info')
api.post('/auth/login', credentials)
api.put('/user/password', data)

Axios 被封装在：src/lib/axios.ts

封装后，项目里的其他地方不用每次都重复写请求配置。
它统一处理：
基础路径 /api/v1
请求超时时间
携带 cookie
错误处理
401 未登录跳转登录页
403 无权限处理

Axios = REST API 请求工具
```

UI 基础组件：Radix UI + 类 shadcn/ui 封装

```
Radix UI 是一套“无样式或少样式”的高质量基础交互组件。
比如：
Dialog 弹窗
DropdownMenu 下拉菜单
Tabs 标签页
Tooltip 提示
Popover 浮层
Select 选择框
Switch 开关
Checkbox 复选框
Accordion 折叠面板

Radix 的优势是它已经处理好了很多复杂交互：
键盘可访问性
焦点管理
弹窗关闭逻辑
ARIA 属性
屏幕阅读器支持

但是 Radix 默认不负责漂亮样式。
所以项目又用了类似 shadcn/ui 的封装方式：把 Radix 组件包一层，加上 Tailwind 样式，变成项目自己的 UI 组件。
例如：
src/components/ui/button.tsx
src/components/ui/dialog.tsx
src/components/ui/dropdown-menu.tsx
src/components/ui/tabs.tsx

Radix UI = 可靠的交互骨架
类 shadcn/ui 封装 = 项目自己的漂亮组件库
```

样式：Tailwind CSS v4--一种用 class 写样式的工具。

```
传统 CSS：
.button {
  padding: 8px 16px;
  background: blue;
  color: white;
}

Tailwind：
<button className="px-4 py-2 bg-blue-500 text-white">
  登录
</button>

在：src/styles/index.css
里面定义了：
字体
亮色主题
暗色主题
颜色变量
圆角
阴影
滚动条
Markdown 样式
编辑器样式
Toast 样式

Tailwind CSS = 快速写界面样式的工具
```

表单：react-hook-form + zod

```
react-hook-form 负责管理表单状态：
输入框的值
错误信息
是否正在提交
是否被修改过
提交事件

zod 负责校验规则：
什么是必填的
修改密码是前后不一样
两次新密码一样

react-hook-form = 表单状态管理
zod = 表单规则校验

react-hook-form + zod = 表单输入和校验的组合方案
```

图表：recharts--React 生态里的图表库。

```
可画：
折线图
柱状图
饼图
面积图
统计趋势图
Tooltip
Legend

Dashboard 或分析页面会用它展示数据：
Token 使用量
工具调用次数
Flow 数量
模型使用统计
Provider 使用情况
任务执行统计

recharts = 用 React 写统计图表
```

Markdown：react-markdown、marked、tiptap

```
AI 回复
任务报告
Prompt
知识库内容
说明文档
Flow 报告

相关库：
```text
react-markdown
marked
tiptap

react-markdown：
把 Markdown 文本渲染成 React 页面
marked：
把 Markdown 解析成 HTML 或中间结果
tiptap：
富文本/Markdown 编辑器，用来编辑内容

react-markdown = 显示 Markdown
marked = 解析 Markdown
tiptap = 编辑 Markdown/富文本
```

终端：xterm--浏览器里的终端组件，“网页里的命令行窗口”

```
后端会运行工具、命令、容器任务，前端需要展示类似终端的输出，例如：
nmap 扫描结果
shell 命令输出
工具执行日志
容器终端内容

src/components/shared/terminal/
src/features/flows/terminal/

xterm = 网页里的专业终端显示器
```

测试：vitest + testing-library

```
Vitest 是测试运行器
负责：
找到测试文件
运行测试
统计通过/失败
生成覆盖率

testing-library 是 React 组件测试工具：
页面上是否出现按钮
用户点击按钮后是否出现弹窗
输入错误邮箱是否显示错误提示

Vitest = 跑测试
testing-library = 测 React 组件行为
```

简记：

```
Vite：启动和打包项目
React：写页面
TypeScript：让代码更安全
react-router-dom：页面跳转
Apollo Client：请求 GraphQL 数据
GraphQL subscriptions：实时更新
Axios：请求 REST 接口
Radix UI/shadcn：基础 UI 组件
Tailwind CSS：写样式
react-hook-form + zod：表单和校验
recharts：画图表
Markdown 工具：显示/编辑 Markdown
xterm：网页终端
Vitest/testing-library：测试代码
```

组合：

```
用户打开/flows/123

1. react-router-dom 判断当前路由是 /flows/:flowId
2. React 渲染 Flow 页面
3. Apollo Client 发送 GraphQL 请求，获取 Flow 详情
4. Apollo Client 建立 WebSocket 订阅
5. 后端有任务更新、终端日志、AI 消息时主动推送
6. React 收到新数据后重新渲染页面
7. Tailwind CSS 控制界面样式
8. Radix/shadcn 组件提供按钮、弹窗、菜单、标签页
9. xterm 显示终端输出
10. react-markdown 显示 AI 回复或报告内容
11. recharts 显示统计图表
12. Axios 处理部分 REST 操作，比如文件上传或登录

用户浏览器
   |
   | React 页面
   |
   |-- react-router-dom 控制显示哪个页面
   |
   |-- Apollo Client 请求 GraphQL 数据
   |       |
   |       |-- HTTP 查询数据
   |       |
   |       |-- WebSocket 接收实时更新
   |
   |-- Axios 请求 REST API
   |
   |-- Tailwind + Radix 显示界面
   |
   |-- xterm 显示终端
   |
   |-- markdown 工具显示报告/AI 内容
```

目录：

```
frontend/
├── public/
│   ├── favicon/
│   │   └── 浏览器图标、PWA manifest
│   └── fonts/
│       └── Inter、Roboto Mono、NotoSans 等字体文件
│
├── scripts/
│   ├── generate-ssl.ts
│   │   └── 开发环境生成 HTTPS 证书
│   └── lib.ts
│       └── 构建脚本辅助函数，例如获取 git hash
│
├── src/
│   ├── main.tsx
│   │   └── React 应用真正入口
│   ├── app.tsx
│   │   └── 全局 Provider、路由、页面懒加载配置
│   │
│   ├── pages/
│   │   └── 页面级组件，一个文件基本对应一个路由
│   │
│   ├── features/
│   │   └── 业务功能模块，比 pages 更细，放具体业务组件和 Hook
│   │
│   ├── components/
│   │   └── 可复用组件，包括布局、UI 基础组件、图标、共享组件
│   │
│   ├── providers/
│   │   └── React Context，全局或局部状态管理
│   │
│   ├── graphql/
│   │   └── GraphQL 自动生成类型和 Hook，不建议手改
│   │
│   ├── hooks/
│   │   └── 通用 React Hook
│   │
│   ├── lib/
│   │   └── 工具函数、请求封装、报告生成、路由标题等
│   │
│   ├── models/
│   │   └── TypeScript 数据模型
│   │
│   ├── schemas/
│   │   └── zod 表单校验 schema
│   │
│   ├── styles/
│   │   └── 全局 Tailwind、主题变量、字体、滚动条、编辑器样式
│   │
│   └── types/
│       └── TypeScript 类型补充声明
│
├── types/
│   └── Vite、Vitest、Tiptap 类型声明
│
├── package.json
│   └── 依赖和命令
├── vite.config.ts
│   └── Vite 构建、代理、HTTPS、代码分包配置
├── graphql-codegen.ts
│   └── GraphQL 类型生成配置
├── graphql-schema.graphql
│   └── 前端使用的 GraphQL 操作定义来源
├── index.html
│   └── HTML 模板
├── vitest.config.ts
│   └── 测试配置
└── eslint.config.mjs
    └── 代码规范配置
    
src/pages/“页面入口”
├── login.tsx
│   └── 登录页
├── oauth-result.tsx
│   └── OAuth 登录回调结果页
├── dashboard/
│   ├── dashboard.tsx
│   ├── dashboard-overview.tsx
│   └── dashboard-analytics.tsx
│
├── flows/
│   ├── flows.tsx
│   │   └── Flow 列表
│   ├── new-flow.tsx
│   │   └── 新建 Flow
│   ├── flow.tsx
│   │   └── 单个 Flow 的主工作台
│   └── flow-report.tsx
│       └── Flow 报告页面
│
├── resources/
│   └── resources.tsx
│       └── 用户资源文件管理
│
├── knowledges/
│   ├── knowledges.tsx
│   └── knowledge.tsx
│
├── templates/
│   ├── templates.tsx
│   └── template.tsx
│
└── settings/
    ├── settings-providers.tsx
    ├── settings-provider.tsx
    ├── settings-prompts.tsx
    ├── settings-prompt.tsx
    └── settings-api-tokens.tsx
    
src/features/业务模块。页面通常会调用这里的组件。
├── authentication/
│   ├── login-form.tsx
│   └── password-change-form.tsx
│
├── flows/
│   ├── flow-form.tsx
│   ├── flow-tabs.tsx
│   ├── flow-central-tabs.tsx
│   ├── agents/
│   ├── dashboard/
│   ├── files/
│   ├── messages/
│   ├── screenshots/
│   ├── tasks/
│   ├── terminal/
│   ├── tools/
│   └── vector-stores/
│
├── resources/
│   └── 资源文件搜索、上传、移动、复制、删除、新建目录等逻辑
│
├── knowledges/
│   └── 知识库表单、详情页布局、导航
│
└── templates/
    └── Flow 模板相关导航和详情逻辑

src/components/更通用的组件
├── ui/
│   └── 基础 UI 组件，例如 Button、Input、Dialog、Table、Tabs、Tooltip
│
├── layouts/
│   ├── main-layout.tsx
│   ├── main-sidebar.tsx
│   ├── settings-layout.tsx
│   └── flows-layout.tsx
│
├── routes/
│   ├── protected-route.tsx
│   └── public-route.tsx
│
├── icons/
│   └── OpenAI、Anthropic、Gemini、Ollama、Qwen 等 provider 图标
│
├── shared/
│   ├── markdown.tsx
│   ├── markdown-editor.tsx
│   ├── monaco-terminal.tsx
│   ├── document-title.tsx
│   ├── confirmation-dialog.tsx
│   ├── file-manager/
│   ├── terminal/
│   ├── detail-navigation/
│   ├── inline-edit/
│   ├── overwrite/
│   └── unsaved-changes/
│
└── dashboard/
    └── 图表卡片、指标卡片、图表 tooltip
    
src/providers/是 React 里的“全局/局部状态仓库”
├── user-provider.tsx
│   └── 登录状态、登出、OAuth、刷新用户信息
├── theme-provider.tsx
│   └── 明暗主题
├── flow-provider.tsx
│   └── 单个 Flow 的数据、Assistant、实时订阅
├── flows-provider.tsx
│   └── Flow 列表、新建、删除、完成等
├── sidebar-flows-provider.tsx
│   └── 侧边栏 Flow 数据
├── providers-provider.tsx
│   └── LLM Provider 配置
├── resources-provider.tsx
│   └── 用户资源文件状态
├── templates-provider.tsx
│   └── Flow 模板状态
├── knowledges-provider.tsx
│   └── 知识库状态
├── favorites-provider.tsx
│   └── 收藏 Flow
└── system-settings-provider.tsx
    └── 系统配置
```

知识弥补

```
1. HTML / CSS / 浏览器基础
2. JavaScript 基础
3. TypeScript 基础
4. React 组件和 Hook
5. React Router
6. Axios 和 REST API
7. GraphQL 基础
8. Apollo Client
9. WebSocket / Subscription
10. react-hook-form + zod
11. Tailwind CSS
12. Radix UI / shadcn 风格组件
13. Vite / pnpm / ESLint / Prettier
14. Vitest 测试
```

```
第一轮：看懂项目怎么启动
  package.json
  index.html
  src/main.tsx
  src/app.tsx
  
第二轮：看懂登录
  user-provider.tsx
  protected-route.tsx
  public-route.tsx
  login.tsx
  login-form.tsx

第三轮：看懂请求
  lib/axios.ts
  lib/apollo.ts
  graphql/types.ts 只看 Hook 名字，不深入

第四轮：看懂 Flow 主流程
  pages/flows/flows.tsx
  pages/flows/new-flow.tsx
  pages/flows/flow.tsx
  providers/flow-provider.tsx

第五轮：看懂 Flow 子功能
  features/flows/tasks
  features/flows/messages
  features/flows/terminal
  features/flows/files
  features/flows/agents

第六轮：看懂设置和资源管理
  pages/settings
  pages/resources
  features/resources

第七轮：看懂通用组件
  components/shared
  components/ui

第八轮：看懂测试和工程化
  *.test.ts
  vitest.config.ts
  eslint.config.mjs
```

## 四、后端分析--\backend

### 1.后端--自动化渗透测试调度中心

用户在前端创建一个 Flow，也就是一次渗透测试任务。后端会做这些事：

1. 验证用户身份。
2. 创建 Flow 记录。
3. 选择 LLM Provider，比如 OpenAI、Anthropic、Gemini、Ollama 等。
4. 创建 Docker 容器作为安全执行环境。
5. 让 AI Agent 拆解任务、搜索信息、写代码、执行命令、分析结果。
6. 把执行过程写入数据库。
7. 通过 GraphQL Subscription 实时推送进度给前端。

```
pentagi-2.1.0\backend\cmd\pentagi\main.go
```

```
读取配置
  ↓
初始化日志、观测系统
  ↓
连接 PostgreSQL
  ↓
执行数据库迁移
  ↓
初始化 Docker Client
  ↓
初始化 LLM Providers
  ↓
恢复未完成的 Flow
  ↓
创建 Gin Router
  ↓
启动 HTTP/HTTPS 服务
```

### 2.backend 顶层目录

```
backend/
├── cmd/
├── docs/
├── fern/
├── gqlgen/
├── migrations/
├── pkg/
├── sqlc/
├── go.mod
└── go.sum
```

1. cmd/--main.go

	```
	backend/cmd/
	├── pentagi/      主后端服务入口
	├── installer/    安装向导
	├── ctester/      Docker/容器测试工具
	├── ftester/      LLM function calling 测试工具
	└── etester/      embedding/向量搜索测试工具
	```

2. docs/后端文档目录，包含很多设计说明和使用说明，帮助理解配置、数据库、Docker 执行和 Flow 运行逻辑
3. fern/和 API 文档或 SDK 生成有关
4. gqlgen/GraphQL 代码生成配置

```
GraphQL schema 在：
backend/pkg/graph/schema.graphqls
```

5. migrations/数据库迁移

```
backend/migrations/
├── migrations.go
└── sql/.sql 文件，用来创建和修改数据库表、枚举、索引、权限等
```

6. sqlc/SQLC 查询定义

`sqlc` 是一个 Go 工具，它根据 SQL 文件生成类型安全的 Go 数据库访问代码

```
backend/sqlc/
├── sqlc.yml
└── models/
```

​    7.pkg/

```
backend/pkg/
├── config/        配置读取
├── server/         HTTP REST / GraphQL API 层
├── controller/     Flow、Task、Agent 调度核心
├── providers/      大模型 Provider 接入
├── tools/          Agent 可调用工具
├── docker/         Docker 容器封装
├── database/       数据库访问层
├── graph/          GraphQL 层
├── templates/      Prompt 模板
├── csum/           上下文压缩总结
├── observability/  日志、Tracing、Langfuse、OpenTelemetry
├── graphiti/       知识图谱客户端
├── flowfiles/      Flow 文件管理
├── resources/      用户资源文件管理
├── terminal/       终端输出处理
├── queue/          队列
├── schema/         JSON Schema 工具
├── system/         系统平台工具
├── cast/           消息链解析/转换
└── version/        版本信息
```





## 五、数据库分析

1.Docker 数据库配置：pentagi-2.1.0/docker-compose.yml

2.数据库连接配置：pentagi-2.1.0/backend/pkg/config/config.go:21

3.后端启动时怎么连接数据库和跑迁移：pentagi-2.1.0/backend/cmd/pentagi/main.go:80

4.最初始的数据库表：pentagi-2.1.0/backend/migrations/sql/20241026_115120_initial_state.sql

5.SQLC 配置：pentagi-2.1.0/backend/sqlc/sqlc.yml

6.具体查询：
backend/sqlc/models/flows.sql
pentagi-2.1.0/backend/sqlc/models/users.sql
pentagi-2.1.0/backend/sqlc/models/knowledge.sql

......



## 六、认证系统

```
backend/pkg/server/
├── router.go                 # 注册认证路由、中间件、Cookie session
├── middleware.go             # 本地用户限制、禁缓存中间件
├── auth/
│   ├── auth_middleware.go    # 核心认证中间件：Cookie / Bearer Token
│   ├── session.go            # Cookie 和 JWT 签名密钥生成
│   ├── api_token_jwt.go      # API Token JWT 签发和校验
│   ├── api_token_id.go       # 生成 10 位 token_id
│   ├── api_token_cache.go    # API Token 状态/权限缓存
│   ├── users_cache.go        # 用户状态/哈希缓存
│   └── permissions.go        # 权限检查工具
├── services/
│   ├── auth.go               # 登录、登出、OAuth、/info
│   ├── users.go              # 用户管理、改密码
│   └── api_tokens.go         # REST API Token 管理
├── oauth/
│   ├── client.go             # OAuth 抽象接口
│   ├── google.go             # Google 登录
│   └── github.go             # GitHub 登录
└── models/
    ├── users.go              # 用户、登录、改密模型
    ├── api_tokens.go         # API Token 模型
    └── init.go               # 校验器，包括密码复杂度

frontend/src/
├── providers/user-provider.tsx                 # 前端登录状态管理
├── features/authentication/login-form.tsx      # 登录表单
├── features/authentication/password-change-form.tsx
├── pages/login.tsx
├── pages/oauth-result.tsx
├── pages/settings/settings-api-tokens.tsx
├── components/routes/protected-route.tsx
├── components/routes/public-route.tsx
├── lib/axios.ts                                # REST 请求，自动带 Cookie
└── lib/apollo.ts                               # GraphQL / WebSocket 客户端
```

![](https://img.xiaoyuwell.top/PicGo/20260630234959386.png)

## 七、配置文件

```
pentagi-2.1.0/
├── .env
│   └── 本机真实环境配置文件，存放 API Key、数据库密码、端口等敏感配置，不要上传 GitHub。
│
├── .env.example
│   └── 环境变量模板，第一次部署时通常复制为 .env 再修改。
│
├── docker-compose.yml
│   └── 核心 Docker 启动配置，启动 PentAGI 主服务、PostgreSQL/pgvector、scraper 等。
│
├── docker-compose-observability.yml
│   └── 可选监控配置，启动 Grafana、Loki、Jaeger、VictoriaMetrics、OpenTelemetry。
│
├── docker-compose-langfuse.yml
│   └── 可选 LLM 观测平台配置，用于查看大模型调用、trace、成本等。
│
├── docker-compose-graphiti.yml
│   └── 可选知识图谱配置，启动 Neo4j + Graphiti，给 Agent 提供知识记忆能力。
│
├── Dockerfile
│   └── 构建生产镜像：先构建前端，再构建 Go 后端，最后打包运行环境。
│
├── backend/
│   └── Go 后端主目录，负责 API、认证、数据库、AI Agent、工具执行等。
│
├── frontend/
│   └── React + TypeScript 前端主目录，负责浏览器页面和用户操作界面。
│
├── observability/
│   └── 监控系统配置目录，包含 Grafana 面板、Loki 日志、Jaeger 链路追踪等。
│
├── examples/
│   └── 示例配置目录，主要是不同 LLM Provider 的 YAML 示例配置。
│
├── scripts/
│   └── 脚本目录，例如容器入口脚本、构建辅助脚本。
│
├── build/
│   └── 构建相关文件目录。
│
├── licenses/
│   └── 第三方依赖许可证信息。
│
├── README.md
│   └── 项目说明文档，介绍功能、安装、运行方式。
│
├── AGENTS.md
│   └── 给 AI 编码助手看的项目工作规则。
│
└── DEPLOY_LINUX.md
    └── Linux 部署说明。
```

## 八、Docker Compose

```
E:\pentest\github\pentagi-2.1.0
├── docker-compose.yml
│   主 Docker Compose 文件，启动 PentAGI 核心服务
├── docker-compose-langfuse.yml
│   可选 Langfuse LLM 追踪分析栈
├── docker-compose-observability.yml
│   可选 Grafana / OTEL / Loki / Jaeger 监控栈
├── docker-compose-graphiti.yml
│   可选 Graphiti / Neo4j 知识图谱栈
├── Dockerfile
│   构建 PentAGI 主镜像
├── .env.example
│   环境变量模板
├── backend
│   Go 后端，REST、GraphQL、数据库、agent、工具调用逻辑
├── frontend
│   React + TypeScript 前端
├── observability
│   Grafana、OpenTelemetry、Loki、Jaeger、ClickHouse 配置
├── examples
│   Provider 配置、提示词、报告、指南示例
├── scripts
│   镜像入口脚本等辅助脚本
├── build
│   构建相关资源
├── licenses
│   许可证信息
└── backend/cmd/installer/files/links
    安装器里引用的 Compose 文件链接模板
```



## 九、observability-监控相关

```
observability/
├── clickhouse/
│   └── prometheus.xml
│       # ClickHouse 的 Prometheus 指标暴露配置
│       # 让 ClickHouse 在 9363 端口提供 /metrics
│       # OpenTelemetry Collector 会抓这里的指标
│
├── grafana/
│   ├── config/
│   │   ├── grafana.ini
│   │   │   # Grafana 基础配置
│   │   │   # 默认账号：admin
│   │   │   # 默认密码：admin
│   │   │   # 默认首页指向 dashboards/home.json
│   │   │
│   │   └── provisioning/
│   │       ├── dashboards/
│   │       │   └── dashboard.yml
│   │       │       # 自动加载 Grafana dashboard JSON 文件
│   │       │       # dashboards 目录下的看板会自动出现在 Grafana
│   │       │
│   │       └── datasources/
│   │           └── datasource.yml
│   │               # Grafana 数据源配置
│   │               # VictoriaMetrics：查指标
│   │               # Jaeger：查链路追踪
│   │               # Loki：查日志
│   │               # 还配置了 trace 和 log 的互相跳转
│   │
│   └── dashboards/
│       ├── home.json
│       │   # Grafana 首页
│       │   # 展示可用 dashboard 列表
│       │
│       ├── components/
│       │   ├── pentagi_service.json
│       │   │   # PentAGI 后端服务看板
│       │   │   # 看 CPU、内存、Go goroutine、GC、堆内存等
│       │   │
│       │   └── victoriametrics.json
│       │       # VictoriaMetrics 自身运行状态看板
│       │       # 看指标数据库的存储、查询、写入、资源消耗
│       │
│       └── server/
│           ├── docker_containers.json
│           │   # Docker 容器资源看板
│           │   # 看每个容器 CPU、内存、网络、IO
│           │
│           ├── docker_engine.json
│           │   # Docker Engine 看板
│           │   # 看 Docker daemon 的运行指标
│           │
│           └── node_exporter_full.json
│               # 宿主机完整系统看板
│               # 看 CPU、内存、磁盘、网络、进程、文件系统等
│
├── jaeger/
│   ├── config.yml
│   │   # Jaeger 主配置
│   │   # 负责接收、查询、展示 trace
│   │   # Web UI 默认容器内端口是 16686
│   │
│   ├── plugin-config.yml
│   │   # Jaeger ClickHouse 存储插件配置
│   │   # 告诉 Jaeger 把 trace 数据写入 ClickHouse
│   │   # 默认连接 clickstore:9000
│   │
│   ├── sampling_strategies.json
│   │   # Jaeger 采样策略配置
│   │   # 控制哪些 trace 被采集
│   │
│   └── bin/
│       ├── jaeger-clickhouse-linux-amd64
│       ├── jaeger-clickhouse-linux-arm64
│       └── SOURCE.md
│           # Jaeger 的 ClickHouse 存储插件二进制
│           # 根据 CPU 架构自动选择 amd64 或 arm64
│
├── loki/
│   └── config.yml
│       # Loki 日志系统配置
│       # 使用本地文件系统存储日志 chunks 和 rules
│       # Grafana 通过 Loki 查询日志
│
└── otel/
    └── config.yml
        # OpenTelemetry Collector 核心配置
        # 接收 PentAGI 后端发来的 logs / metrics / traces
        # 同时抓 node-exporter、cadvisor、ClickHouse、Jaeger、Loki、pgexporter 等指标
        # traces 发送到 Jaeger
        # logs 发送到 Loki
        # metrics 发送到 VictoriaMetrics
```



## 十、Langfuse-LLM观测

pentagi会让多agent进行自动渗透测试，langfuse负责记录这些agent做了什么、调用的模型、输入输出、用了多少token、有无报错、工具执行结果

```
docker-compose-langfuse.yml：启动 Langfuse 自身服务。
docker-compose.yml)：PentAGI 主服务接收 Langfuse 环境变量。
backend/pkg/observability/lfclient.go：后端创建 Langfuse 客户端。
backend/pkg/observability/langfuse：项目自写的 Langfuse SDK 封装。
backend/docs/langfuse.md：Langfuse 集成说明文档。
.env.example：Langfuse 环境变量示例。
```

整体架构分两部分：

```
1.Langfuse服务端
由docker-compose-langfuse.yml 启动，包括：
langfuse-web：网页界面，默认映射到 http://localhost:4000
langfuse-worker：后台处理队列
langfuse-postgres：保存项目、用户、配置等数据
langfuse-clickhouse：保存大量追踪分析数据
langfuse-redis：队列和缓存
langfuse-minio：对象存储，类似本地 S3

2.Pentagi后端上报端：
Go 后端启动时读取：
LANGFUSE_BASE_URL
LANGFUSE_PROJECT_ID
LANGFUSE_PUBLIC_KEY
LANGFUSE_SECRET_KEY
然后通过 backend/pkg/observability/lfclient.go 创建客户端，把 AI 执行过程发送到 Langfuse。
```

![](https://img.xiaoyuwell.top/PicGo/20260616215036829.png)

记录

```
Trace：一整个大流程，如一个flow完整执行
Span：一个有开始和结束的步骤，比如“准备 flow worker”。
Generation：一次 LLM 调用，比如调用 OpenAI、Anthropic、Qwen 等模型。
Event：一个瞬时事件，比如工具调用失败、限流重试。
Score：评分，比如搜索结果质量、工具执行结果。
Retriever：向量库检索，比如从 pgvector 搜索记忆。
Tool：工具调用，比如搜索、终端、浏览器、代码工具等。
Embedding：文本向量化调用。
Agent / Chain / Evaluator / Guardrail：为更复杂的 Agent 流程预留的观测类型。
```

**启动入口**
后端主程序在backend/cmd/pentagi/main.go中初始化 Langfuse

相关目录

```
E:/pentest/github/pentagi-2.1.0
├── docker-compose-langfuse.yml
│   └── Langfuse 服务端部署文件，定义 web、worker、postgres、clickhouse、redis、minio
├── docker-compose.yml
│   └── PentAGI 主服务，注入 LANGFUSE_* 环境变量
├── .env.example
│   └── Langfuse 配置模板
├── backend/docs/langfuse.md
│   └── Langfuse 集成开发文档
└── backend/pkg/observability/
    ├── lfclient.go
    │   └── 创建 Langfuse client 和 observer
    ├── obs.go
    │   └── 全局观测入口，把 Langfuse 和 OpenTelemetry 统一起来
    └── langfuse/
        ├── client.go
        │   └── Langfuse API 客户端封装
        ├── observer.go
        │   └── 批量队列发送机制
        ├── observation.go
        │   └── Trace、Span、Event、Generation 等统一入口
        ├── generation.go
        │   └── LLM 调用记录
        ├── span.go / event.go / score.go
        │   └── 步骤、事件、评分记录
        ├── retriever.go / embedding.go / tool.go
        │   └── 检索、向量化、工具调用记录
        ├── converter.go
        │   └── 把 LangChainGo 消息转换成 Langfuse 更好展示的格式
        └── api/
            └── 自动生成的 Langfuse OpenAPI 客户端代码
```



## 十一、Graphiti-知识图谱相关功能-配合Neo4j-向量数据库

整体架构

```
PentAGI Go 后端
  |
  | 1. 调用 Graphiti API：写入/查询图谱
  v
Graphiti 服务
  |
  | 2. 用 OpenAI-compatible LLM 做实体/关系抽取
  v
Neo4j
  |
  | 3. 存实体、关系、时间上下文
  v
知识图谱查询结果返回给 agent

PentAGI Go 后端
  |
  | 调用 Embedding Provider 生成向量
  v
PostgreSQL + pgvector
  |
  | 存 document + embedding + cmetadata
  v
memory / guide / answer / code 语义检索
```

目录

```
docker-compose-graphiti.yml
  Graphiti + Neo4j 服务定义。

backend/pkg/config/config.go
  GRAPHITI_ENABLED / GRAPHITI_TIMEOUT / GRAPHITI_URL 配置。

backend/pkg/graphiti/client.go
  PentAGI 对 graphiti-go-client 的封装。

backend/pkg/providers/providers.go
  启动时初始化 Graphiti client；失败则降级，不阻塞主服务。

backend/pkg/providers/performer.go
  agent 响应和工具执行自动写入 Graphiti 的核心逻辑。

backend/pkg/templates/graphiti/
  写入 Graphiti 前的文本模板。

backend/pkg/tools/graphiti_search.go
  agent 查询 Graphiti 的工具实现。

backend/pkg/tools/registry.go
  graphiti_search 工具 schema 和描述。

backend/migrations/sql/20260501_120000_knowledge.sql
  pgvector 知识库表结构和清理逻辑。

backend/pkg/database/knowledge/
  pgvector 知识库 CRUD / 语义搜索逻辑。

frontend/src/providers/knowledges-provider.tsx
  前端知识库页面的数据层，走 GraphQL 查询 pgvector。
```

两套独立存储-互补使用

```
Graphiti + Neo4j：实体--关系，知识图谱存储
PostgreSQL + pgvector：向量数据库存储

Graphiti 负责“发生过什么、东西之间有什么关系”
pgvector 负责“有没有相似的知识、文档、代码、指南”
```

### 1.Graphiti + Neo4j：知识图谱存储

重点是“实体”和“关系”

实体：

```
IP 地址
域名
端口
服务
漏洞
工具
命令
技术手法
agent 响应
工具执行记录
```

关系：

```
某 IP 开放某端口
某端口运行某服务
某工具发现某目标
某漏洞影响某服务
某技术利用某漏洞
某次工具执行成功或失败
```

Graphiti ：pentagi-2.1.0/backend/pkg/graphiti/client.go，封装 Graphiti 客户端

```
知识图谱服务，PentAGI 后端不会直接操作 Neo4j，而是调用 Graphiti API

接收 PentAGI 发来的文本
调用 LLM 抽取实体和关系
把实体和关系写入 Neo4j
对外提供图谱查询 API

存：按照flow分组写入
1.agent 的回答--函数storeAgentResponseToGraphiti
pentagi-2.1.0/backend/pkg/providers/performer.go
模板：pentagi-2.1.0/backend/pkg/templates/graphiti/agent_response.tmpl
2.工具执行记录--函数storeToolExecutionToGraphiti
模板：pentagi-2.1.0/backend/pkg/templates/graphiti/tool_execution.tmpl

查：pentagi-2.1.0/backend/pkg/tools/graphiti_search.go
```

Neo4j ：真正保存图数据的数据库

```
保存节点
保存边
保存节点之间的关系
支持图遍历
支持按关系查上下文
```

关系：

```
PentAGI 后端 -> Graphiti API -> Neo4j
```

### 2.PostgreSQL + pgvector：向量数据库存储

核心逻辑：

```
文本 -> embedding 向量 -> 存进 PostgreSQL
查询文本 -> embedding 向量 -> 和库里的向量做相似度比较
```

embedding：**把一段文字转换成一串数字向量**，让数据库可以计算“语义相似度”

```
文本 A: nmap discovered SSH and HTTP services
文本 B: target has open port 22 and web service
转：
A -> [0.12, -0.08, 0.44, ...]
B -> [0.10, -0.07, 0.41, ...]

计算这两串数字的距离，距离越近，说明语义越相似
```

pgvector：pentagi-2.1.0/backend/migrations/sql/20260501_120000_knowledge.sql

pgvector 的知识库 API：pentagi-2.1.0/backend/pkg/database/knowledge/knowledge.go

```
创建知识文档
更新知识文档
删除知识文档
列出知识文档
语义搜索知识文档
```

流程：

```
用户创建 Flow
  |
  v
PentAGI agent 开始执行
  |
  |-- 查询 Graphiti ------------------> Graphiti API --> Neo4j
  |                                     返回事件/实体/关系
  |
  |-- 查询 pgvector ------------------> Embedding --> PostgreSQL pgvector
  |                                     返回相似文本
  |
  |-- 执行 terminal/browser/search
  |
  |-- 工具执行记录 -------------------> Graphiti --> Neo4j
  |                                     保存图谱事件和关系
  |
  |-- 工具结果文本 -------------------> Embedding --> pgvector
  |                                     保存 memory 文本块
  |
  |-- agent 主动 store_guide/code/answer
                                        保存可复用知识到 pgvector
```

