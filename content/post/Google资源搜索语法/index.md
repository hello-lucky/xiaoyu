---
title: "Google资源搜索语法"
description: 
slug: 
date: 
image:
categories:
    - others
tags:
    - Google搜索语法
weight: 1

---





# 简记

```
"精确关键词" + site:网站 + filetype:文件类型 + -排除词 + OR + after:/before:
```

| 语法        | 作用                 | 示例                       |
| ----------- | -------------------- | -------------------------- |
| `"关键词"`  | 精确匹配完整短语     | `"machine learning"`       |
| `site:`     | 限定网站或域名       | `人工智能 site:edu.cn`     |
| `filetype:` | 限定文件类型         | `深度学习 filetype:pdf`    |
| `-关键词`   | 排除关键词           | `Python 教程 -广告`        |
| `OR`        | 搜索 A 或 B          | `Python OR Java`           |
| `intitle:`  | 关键词必须出现在标题 | `intitle:深度学习`         |
| `inurl:`    | 关键词必须出现在网址 | `inurl:research AI`        |
| `before:`   | 指定日期之前         | `OpenAI before:2024-01-01` |
| `after:`    | 指定日期之后         | `LLM after:2025-01-01`     |
| `..`        | 数字范围             | `笔记本 5000..8000`        |