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

## 四、后端分析



## 五、数据库分析

## 六、一次flow运行

## 七、认证系统

## 八、配置文件

## 九、Docker Compose

## 十、observability-监控相关



## 十一、Langfuse-LLM观测

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



## 十二、Graphiti-知识图谱相关功能-配合Neo4j-向量数据库



