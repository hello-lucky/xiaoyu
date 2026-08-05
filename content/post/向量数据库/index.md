---
title: "从 pgvecto.rs 到 VectorChord：新版 PostgreSQL 向量数据库原理与 RAG 实战"
description: "深入理解 VectorChord、K-means 向量分区、RaBitQ 量化剪枝，以及一条完整的 RAG 查询链路。"
slug: vectorchord-postgresql-vector-database
date: 
image:
categories:
    - AI
tags:
    - PostgreSQL
    - VectorChord
    - 向量数据库
    - RAG
weight: 1
---

## 一、VectorChord 是什么

在介绍 VectorChord 之前，先澄清一个容易产生误解的说法：**VectorChord 严格来说不是一套独立运行的向量数据库，而是 PostgreSQL 的向量检索扩展**。

它由 pgvecto.rs 的原团队 TensorChord 开发，是 pgvecto.rs 的继任者。旧版 pgvecto.rs 自己定义向量类型和索引；VectorChord 则直接建立在 PostgreSQL 与 pgvector 之上：

| 组件 | 主要职责 |
|---|---|
| PostgreSQL | 表、事务、JOIN、过滤、权限、备份和复制 |
| pgvector | `vector`、`halfvec` 等向量类型及距离运算符 |
| VectorChord | 面向大规模数据的向量索引、量化、剪枝和重排 |

因此，业务应用不需要维护“业务数据库 + 独立向量数据库”两套系统。用户、文档、权限、元数据和 Embedding 可以保存在同一个 PostgreSQL 数据库中，并通过一条 SQL 同时完成业务过滤和语义检索。

VectorChord 的主要索引是 `vchordrq`。它通过 K-means 对向量空间进行分区，再利用 RaBitQ 进行快速距离估计和候选剪枝，最后对少量候选进行更精确的重排。新版还提供磁盘图索引 `vchordg`，不过官方目前仍将其标记为 preview。

整体结构可以概括为：

```text
业务应用
   ├── 调用 Embedding 模型，把文本转换成向量
   └── 连接 PostgreSQL
            ├── 普通关系数据：用户、文档、权限、JSON
            ├── pgvector：保存 vector/halfvec
            └── VectorChord：vchordrq/vchordg 向量索引
```

## 二、快速使用 VectorChord

### 1. 启用扩展

VectorChord 依赖 pgvector，可以使用下面的 SQL 同时安装依赖：

```sql
CREATE EXTENSION IF NOT EXISTS vchord CASCADE;
```

其中，pgvector 提供向量数据类型；VectorChord 提供新的索引访问方法。

### 2. 创建文档向量表

下面是一张适合知识库或 RAG 系统的文档分块表：

```sql
CREATE TABLE document_chunks (
    id          bigserial PRIMARY KEY,
    tenant_id   bigint NOT NULL,
    document_id bigint NOT NULL,
    content     text NOT NULL,
    metadata    jsonb,
    embedding   vector(1536) NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);
```

`vector(1536)` 表示每个向量有 1536 个维度。这个维度必须与所用 Embedding 模型的输出一致。

需要注意：PostgreSQL 和 VectorChord 不会自动把文本转换成向量。应用必须先调用 Embedding 模型，再把得到的向量写入数据库。

```text
原始文本
   ↓
Embedding 模型
   ↓
[0.018, -0.032, 0.107, ...]
   ↓
写入 document_chunks.embedding
```

入库和查询必须使用同一个 Embedding 模型以及相同的文本预处理方式。即使两个模型输出的维度相同，它们的向量空间也通常不兼容。

### 3. 创建 vchordrq 索引

如果使用余弦距离，可以创建：

```sql
CREATE INDEX document_chunks_embedding_idx
ON document_chunks
USING vchordrq (embedding vector_cosine_ops);
```

常见距离类型如下：

| 距离 | 索引运算符类 | SQL 运算符 |
|---|---|---|
| 欧氏距离 | `vector_l2_ops` | `<->` |
| 负内积 | `vector_ip_ops` | `<#>` |
| 余弦距离 | `vector_cosine_ops` | `<=>` |

索引运算符类必须和查询中的距离运算符匹配，否则 PostgreSQL 可能无法使用对应索引。

### 4. 查询相似文档

```sql
SELECT
    id,
    content,
    1 - (embedding <=> :query_vector) AS similarity
FROM document_chunks
WHERE tenant_id = :tenant_id
ORDER BY embedding <=> :query_vector
LIMIT 10;
```

`<=>` 返回的是余弦距离，距离越小越相似；`1 - 距离` 可以转换成更容易理解的余弦相似度。

## 三、K-means：先把搜索范围缩小

### 1. 为什么需要 K-means

假设数据库中保存了 1,000 万条向量。如果每次查询都将查询向量与这 1,000 万条向量逐一比较，结果虽然精确，但计算量非常大。

VectorChord 的第一步不是立刻比较所有向量，而是先利用 K-means 把向量空间切分成多个区域。查询到来后，只搜索最可能包含答案的区域。

可以把它想象成去图书馆找书：

```text
没有分类：
遍历整个图书馆的每一本书

使用 K-means 分类：
先判断问题更接近“计算机”书架
再到计算机书架中查找
```

### 2. K-means 的基本思想

K-means 中的 `K` 表示要划分成多少个簇。算法大致分为四步：

1. 随机或按某种策略选择 K 个初始中心点。
2. 计算每个向量距离哪个中心最近。
3. 把向量分配给最近的中心。
4. 对每个簇中的向量求平均，得到新的中心。

步骤 2～4 会重复多次，直到中心基本不再变化，或者达到最大迭代次数。

例如有六个二维向量：

```text
A(1,1)  B(1,2)  C(2,1)
D(8,8)  E(8,9)  F(9,8)
```

令 `K = 2`，最终大致会形成：

```text
簇 1：A、B、C，中心约为 (1.33, 1.33)
簇 2：D、E、F，中心约为 (8.33, 8.33)
```

如果查询向量是 `(1.5, 1.4)`，它明显更接近簇 1，因此可以优先搜索 A、B、C，而不必先检查 D、E、F。

### 3. K-means 在 VectorChord 中做什么

在 `vchordrq` 中，K-means 不是为了给数据生成业务标签，而是为了构建向量空间分区。每个叶子分区维护一个向量列表，也就是 list。

```text
全部向量
   ├── list 1
   ├── list 2
   ├── list 3
   └── list 4
```

查询时，VectorChord先比较查询向量与各个中心点的距离，再探测距离最近的若干 lists。这样就把搜索范围从“全部向量”缩小成“附近分区中的向量”。

较大的数据集可以显式配置分区：

```sql
CREATE INDEX document_chunks_embedding_idx
ON document_chunks
USING vchordrq (embedding vector_cosine_ops)
WITH (options = $$
residual_quantization = true

[build.internal]
lists = [1000]
spherical_centroids = true
build_threads = 8
$$);
```

参数含义：

- `lists`：向量空间的分区结构。
- `spherical_centroids = true`：使用球面 K-means，通常更适合余弦距离。
- `build_threads`：构建 K-means 分区时使用的线程数。
- `residual_quantization`：量化向量与聚类中心之间的残差，通常有助于改善速度和召回。

查询时可以通过 `probes` 控制检查多少个分区：

```sql
SET vchordrq.probes = '10';
```

一般规律是：

```text
probes 较小 → 检查的分区少，查询更快，但可能漏掉相关结果
probes 较大 → 检查的分区多，召回率更高，但查询更慢
```

K-means 只负责缩小搜索空间，并不能独自解决分区内部仍有大量候选的问题。这就需要下一层的 RaBitQ。

## 四、RaBitQ：低成本估算距离并剪枝

### 1. 为什么需要向量量化

一个 1536 维的 `float32` 向量，仅原始数值就大约需要：

```text
1536 × 4 字节 = 6144 字节，约 6 KB
```

如果有 1 亿条向量，仅原始向量数据就可能达到数百 GB。更重要的是，每次比较都要进行大量浮点运算和内存读取。

向量量化的核心目标是：

> 用更少的比特近似表示原始向量，以更低的存储和计算成本完成候选筛选。

### 2. RaBitQ 的直观理解

RaBitQ 是一种面向近似最近邻搜索的量化方法。它不会在第一阶段完整计算所有候选向量的精确距离，而是把高精度向量转换为紧凑的低比特表示。

可以把原始向量与量化向量类比为：

```text
原始向量：高清原图
量化向量：压缩缩略图
```

用缩略图可以快速判断两张图片是否明显不同；只有看起来相似的图片，才有必要加载高清原图继续比较。

VectorChord 的检索过程可以简化成：

```text
1,000 万条原始向量
       ↓ K-means 分区
10 万条附近候选
       ↓ RaBitQ 快速估距和下界剪枝
数百条高质量候选
       ↓ 使用更高精度距离重排
Top 10
```

### 3. 距离估计与下界

普通量化算法往往只能给出“估计距离”。RaBitQ 的关键价值之一，是还可以给出距离下界。

假设当前 Top 10 结果中最差的距离是 `0.25`，某个候选通过 RaBitQ 得到的距离下界已经是 `0.40`。即使再进行精确计算，它也不可能进入当前 Top 10，因此可以直接跳过。

```text
当前 Top-K 门槛：0.25

候选 A 的距离下界：0.10 → 可能入选，继续精算
候选 B 的距离下界：0.40 → 不可能入选，直接剪枝
```

这就是“剪枝”：在不必完整计算每个候选精确距离的情况下，提前淘汰不可能成为答案的向量。

VectorChord 中的 `vchordrq.epsilon` 控制距离下界估计的保守程度：

```sql
SET vchordrq.epsilon = 1.9;
```

通常：

```text
epsilon 较高 → 判断更保守，精算候选更多，召回率更高但速度更慢
epsilon 较低 → 剪枝更激进，速度更快但可能降低召回率
```

### 4. rabitq8 和 rabitq4

VectorChord 1.1.0 开始支持直接将量化结果作为列类型保存：

- `rabitq8`：大致每维 8 bit。
- `rabitq4`：大致每维 4 bit。

示例：

```sql
CREATE TABLE compressed_items (
    id        bigserial PRIMARY KEY,
    embedding rabitq8(1536)
);

INSERT INTO compressed_items (embedding)
VALUES (quantize_to_rabitq8(:embedding::vector));

CREATE INDEX compressed_items_embedding_idx
ON compressed_items
USING vchordrq (embedding rabitq8_cosine_ops);
```

查询向量也需要量化：

```sql
SELECT id
FROM compressed_items
ORDER BY embedding
    <=> quantize_to_rabitq8(:query_vector::vector)
LIMIT 100;
```

不同类型的取舍如下：

| 类型 | 存储空间 | 精度 | 适用场景 |
|---|---:|---:|---|
| `vector` | 最大 | 最高 | 默认选择、需要高精度 |
| `halfvec` | 较小 | 较高 | 常见空间优化 |
| `rabitq8` | 很小 | 有量化损失 | 超大数据、批量召回 |
| `rabitq4` | 最小 | 损失更明显 | 存储极度受限 |

量化是有损的。把 `rabitq8` 或 `rabitq4` 反量化，也无法完整恢复原始向量。重要业务最好保留原始向量，或者保留原始文本和重新生成 Embedding 的能力。

## 五、RAG 查询：从用户问题到最终答案

### 1. RAG 是什么

RAG 的全称是 Retrieval-Augmented Generation，即“检索增强生成”。

大语言模型本身可能：

- 不知道企业内部资料；
- 知识已经过时；
- 对精确条款产生幻觉；
- 无法判断当前用户是否有权查看某份文档。

RAG 先从可信知识库中检索相关资料，再把资料和问题一起交给大语言模型。模型不是单纯依赖记忆，而是基于检索结果生成答案。

```text
用户问题
   ↓
检索知识库
   ↓
找到相关资料
   ↓
资料 + 问题交给大语言模型
   ↓
生成有依据的答案
```

### 2. 知识入库阶段

假设知识库中有一份退款规则文档：

```text
退款申请需要在订单完成后七天内提交。
```

完整入库流程如下：

1. 解析 PDF、网页或 Word 文档。
2. 将长文档切分成多个 chunk。
3. 为每个 chunk 生成 Embedding。
4. 把文本、元数据和向量一起写入 PostgreSQL。
5. 使用 VectorChord 建立向量索引。

```text
PDF/网页/Word
      ↓
文本解析
      ↓
切分文档 chunks
      ↓
Embedding 模型
      ↓
文本 + 元数据 + 向量
      ↓
PostgreSQL + VectorChord
```

写入示例：

```sql
INSERT INTO document_chunks (
    tenant_id,
    document_id,
    content,
    metadata,
    embedding
)
VALUES (
    1001,
    2001,
    '退款申请需要在订单完成后七天内提交。',
    '{"category":"refund","language":"zh"}',
    :embedding
);
```

### 3. 用户查询阶段

用户提出问题：

```text
买完东西多久还能申请退款？
```

应用使用与入库阶段相同的 Embedding 模型，把问题转换成查询向量：

```text
“买完东西多久还能申请退款？”
              ↓
       Embedding 模型
              ↓
       query_vector
```

然后执行向量检索：

```sql
SELECT
    dc.id,
    dc.content,
    dc.metadata,
    1 - (dc.embedding <=> :query_vector) AS similarity
FROM document_chunks dc
WHERE dc.tenant_id = :tenant_id
ORDER BY dc.embedding <=> :query_vector
LIMIT 8;
```

如果还需要权限控制，可以直接利用 PostgreSQL 的关系查询：

```sql
SELECT
    dc.id,
    dc.content,
    d.title,
    1 - (dc.embedding <=> :query_vector) AS similarity
FROM document_chunks dc
JOIN documents d
    ON d.id = dc.document_id
JOIN document_permissions p
    ON p.document_id = d.id
WHERE dc.tenant_id = :tenant_id
  AND p.user_id = :user_id
  AND d.status = 'published'
ORDER BY dc.embedding <=> :query_vector
LIMIT 8;
```

这条 SQL 同时完成了：

- 租户隔离；
- 用户权限过滤；
- 文档状态过滤；
- 表关联；
- 向量相似度排序；
- Top-K 召回。

### 4. 构造提示词并生成答案

假设检索到：

```text
退款申请需要在订单完成后七天内提交。
```

应用把它和用户问题组合成提示词：

```text
请仅根据以下资料回答问题。如果资料不足，请明确说明。

资料：
退款申请需要在订单完成后七天内提交。

用户问题：
买完东西多久还能申请退款？
```

大语言模型生成：

```text
订单完成后的七天内可以提交退款申请。
```

整个 RAG 链路中各组件的职责是：

| 组件 | 职责 |
|---|---|
| Embedding 模型 | 把文本和问题映射到同一个向量空间 |
| VectorChord | 快速找到语义相近的候选文档 |
| PostgreSQL | 保存数据，执行权限、租户、状态和时间过滤 |
| 大语言模型 | 根据检索内容组织最终答案 |

VectorChord 不负责生成 Embedding，也不会直接生成最终答案。

### 5. RAG 的常见优化

#### 文档切分

chunk 过大时，一个向量会混入多个主题；chunk 过小时，又可能缺少完整上下文。需要用真实问题测试 chunk 大小与重叠长度。

#### 混合检索

向量检索擅长理解语义，但对产品编号、错误码和人名等精确关键词不一定理想。可以结合 PostgreSQL 全文搜索或 VectorChord-bm25，再使用 RRF 等方式融合排序。

#### 重排

向量索引先召回几十条或几百条候选，再使用 Cross-Encoder、ColBERT 或其他 reranker 精排，通常能提升最终相关性。

#### 相似度阈值

不要直接认定“相似度大于 0.8 就一定相关”。不同 Embedding 模型和数据集的分数分布不同，阈值必须通过标注数据评估。

#### 评估

至少应同时测量：

- Recall@K：正确答案是否被召回；
- MRR/NDCG：正确答案的排名是否足够靠前；
- P50/P95/P99 延迟；
- 每秒查询数；
- 索引大小与构建时间；
- 加入权限和业务过滤后的实际表现。

## 六、K-means、RaBitQ 和 RAG 的关系

三者处于不同层次，不能混为一谈：

```text
RAG：应用层流程
  └── 需要从知识库检索相关文档
        └── VectorChord 执行向量检索
              ├── K-means：找到最可能相关的向量分区
              └── RaBitQ：在分区内快速估距、剪枝和筛选候选
```

一句话概括：

- **K-means 解决“去哪里找”**；
- **RaBitQ 解决“如何低成本比较和淘汰候选”**；
- **RAG 解决“如何用检索结果帮助大语言模型回答问题”**。

## 七、与 pgvector、pgvecto.rs 的区别

| 项目 | pgvector | pgvecto.rs | VectorChord |
|---|---|---|---|
| 定位 | PostgreSQL 向量基础扩展 | 旧版高性能扩展 | pgvecto.rs 继任者 |
| 向量类型 | `vector`、`halfvec` 等 | 自己定义类型 | 依赖并兼容 pgvector |
| 主要索引 | HNSW、IVFFlat | HNSW、IVF | `vchordrq`、`vchordg` |
| 大规模磁盘检索 | 一般 | 有限 | 重点优化 |
| 低比特类型 | 基础类型为主 | 自有量化类型 | `rabitq8`、`rabitq4` |
| 新项目建议 | 中小规模优先评估 | 不建议新用 | 大规模场景重点评估 |

简单选型建议：

- 几万到几百万向量、优先考虑成熟和简单：先评估 pgvector。
- 数千万到亿级向量、内存有限：重点评估 VectorChord。
- 已经使用 pgvecto.rs：规划转换向量列并重建 VectorChord 索引。
- 需要自动跨机器分片：还应对比 Milvus、Qdrant、Weaviate 等独立向量数据库。

## 八、生产环境注意事项

1. VectorChord 是 PostgreSQL 服务端扩展，托管数据库必须允许安装。
2. PostgreSQL、pgvector 和 VectorChord 的版本需要相互兼容。
3. 向量索引属于近似搜索，不能只测试速度，还要测试召回率。
4. 使用 `EXPLAIN (ANALYZE, BUFFERS)` 确认查询是否命中 `vchordrq` 索引。
5. 大量实时写入会增加索引维护成本，应测试写入与查询并发。
6. 量化是有损操作，重要数据应保留原始文本或原始向量。
7. Embedding 模型升级通常意味着重新生成向量并重建索引。
8. 升级前应测试备份恢复、复制、故障切换和 `REINDEX` 时间。
9. VectorChord 使用 AGPLv3/Elastic License 2.0 双许可证，商业使用前应审查许可证要求。

## 九、总结

VectorChord 的价值不只是“让 PostgreSQL 能存向量”，而是让 PostgreSQL 在保留事务、JOIN、权限和成熟运维能力的同时，具备面向大规模数据的语义检索能力。

它的核心查询链路可以总结为：

```text
查询文本
   ↓ Embedding
查询向量
   ↓ K-means 定位附近分区
候选 lists
   ↓ RaBitQ 快速估距与剪枝
少量高质量候选
   ↓ 精排
Top-K 文档
   ↓ 与用户问题一起交给大语言模型
RAG 最终答案
```

对于需要同时处理业务数据、访问权限和语义检索的 RAG 系统，PostgreSQL + pgvector + VectorChord 是一套非常有吸引力的架构。

## 参考资料

- [VectorChord 官方仓库](https://github.com/tensorchord/VectorChord)
- [VectorChord 官方文档](https://docs.vectorchord.ai/)
- [VectorChord vchordrq 索引文档](https://docs.vectorchord.ai/vectorchord/usage/indexing.html)
- [VectorChord Graph Index 文档](https://docs.vectorchord.ai/vectorchord/usage/graph-index.html)
- [VectorChord Quantization Types 文档](https://docs.vectorchord.ai/vectorchord/usage/quantization-types.html)
- [pgvecto.rs 迁移到 VectorChord](https://docs.vectorchord.ai/vectorchord/admin/migration.html)
