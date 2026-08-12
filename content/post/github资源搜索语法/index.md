---
title: "GitHub资源搜索语法"
description: 
slug: 
date: 
image:
categories:
    - others
tags:
    - github搜索语法
weight: 1
---



# 常用仓库资源搜索语法

| 目的         | 语法             | 示例                     |
| ------------ | ---------------- | ------------------------ |
| 指定语言     | `language:`      | `language:python`        |
| Star 数量    | `stars:`         | `stars:>5000`            |
| Fork 数量    | `forks:`         | `forks:>500`             |
| 指定作者     | `user:`          | `user:torvalds`          |
| 指定组织     | `org:`           | `org:microsoft`          |
| 指定仓库     | `repo:`          | `repo:microsoft/vscode`  |
| 搜仓库名     | `in:name`        | `agent in:name`          |
| 搜描述       | `in:description` | `AI in:description`      |
| 搜 README    | `in:readme`      | `RAG in:readme`          |
| 搜 Topic     | `topic:`         | `topic:machine-learning` |
| 指定 License | `license:`       | `license:mit`            |
| 创建日期     | `created:`       | `created:>2025-01-01`    |
| 最近更新     | `pushed:`        | `pushed:>2026-01-01`     |
| 仓库大小 KB  | `size:`          | `size:<10000`            |
| 排除归档项目 | `archived:false` | `archived:false`         |
| 只看公开仓库 | `is:public`      | `is:public`              |

这些限定符可以自由组合；数值条件支持 `>`、`>=`、`<`、`<=` 和 `..` 范围。

比如：

```
# 高质量 Python 爬虫项目
crawler language:python stars:>5000 archived:false

# 高 Star 的 RAG 项目
RAG stars:>3000 language:python

# 最近仍在维护的 LLM 项目
LLM stars:>1000 pushed:>2026-01-01

# 微软的 TypeScript 项目
org:microsoft language:typescript stars:>1000

# README 中提到 MCP 的项目
MCP in:readme stars:>100

# 机器学习 Topic
topic:machine-learning stars:>5000

# MIT 开源许可证
AI agent license:mit stars:>1000
```

### 范围和排除

可以这样写：

```
stars:1000..5000
```

表示 Star 在 1000～5000 之间。

```
stars:>1000 -language:javascript
```

表示超过 1000 Star，但排除 JavaScript 项目。

对于普通 GitHub 搜索，`-限定符` 可以排除条件，关键词还可以使用 `NOT`。

### 常用的“找宝藏项目”模板

```
关键词 language:语言 stars:>1000 pushed:>2025-01-01 archived:false
```