---
title: "向量数据库产品横向对比"
date: 2022-05-12T14:05:43+08:00
draft: false
tags: ["dbms", "ann"]
---

最近对向量数据库做了一些调研，这篇文章是结果的汇总，涵盖目前能找到的大部分具备向量检索能力的开源和商业产品。信息来源主要是各个产品公开的用户文档，不确定的地方标注了“?”。

## 数据模型

| 数据库     | 数据模型    | 多向量字段 | 向量类型         | 标量类型                                                                                             | Schema-Free | 主键        |
| ---------- | ----------- | ---------- | ---------------- | ---------------------------------------------------------------------------------------------------- | ----------- | ----------- |
| Milvus     | 表 + 向量   |            | float(4B)/binary | int/float/bool                                                                                       |             | uint64      |
| vearch     | 表 + 向量   | Y          | float            | string/integer/long/float/double                                                                     |             | string      |
| 阿里云 ADB | 表 + 向量   | Y?         | float            | 参考 MySQL/PG 类型                                                                                   |             |             |
| pgvector   | 表 + 向量   | Y          | float(4B)        | 参考 PG 类型                                                                                         |             |             |
| ES         | 文档 + 向量 | Y          | float            | 参考 ES 类型                                                                                         | Y           | string      |
| Pinecone   | 文档 + 向量 |            | float            | 标量字段保存在 Json                                                                                  | Y           | string      |
| qdrant     | 文档 + 向量 |            | float            | 标量字段保存在 Json                                                                                  | Y           | uint64/UUID |
| vespa      | 文档 + 向量 | Y          | float/int        | [支持类型较多，参考文档](https://docs.vespa.ai/en/reference/schema-reference.html#field-types)       |             | string      |
| weaviate   | 文档 + 向量 |            | float            | [支持类型较多，参考文档](https://weaviate.io/developers/weaviate/current/data-schema/datatypes.html) | Y[^1]       | UUID        |
| vald       | 向量        |            | float            | N/A                                                                                                  | N/A         | string      |

[^1]: Weaviate 支持 auto-schema，可以不用手动创建 schema。

## 索引类型

**向量相似性度量**

| 数据库         | 余弦 | L2(欧式) | 内积 | Jaccard | Hamming | L1(曼哈顿) | Poincare | Lorentz | 其他  |
| -------------- | ---- | -------- | ---- | ------- | ------- | ---------- | -------- | ------- | ----- |
| Milvus         | Y    | Y        | Y    | Y       | Y       |            |          |         | Y[^2] |
| vearch         |      | Y        | Y    |         |         |            |          |         |       |
| 阿里云 ADB[^3] |      | Y        | Y    |         | Y       |            |          |         |       |
| pgvector       | Y    | Y        | Y    |         |         |            |          |         |       |
| ES             | Y    | Y        | Y    |         |         |            |          |         |       |
| Pinecone       | Y    | Y        | Y    |         |         |            |          |         |       |
| qdrant         | Y    | Y        | Y    |         |         |            |          |         |       |
| vespa          | Y    |          | Y    |         | Y       |            |          |         |       |
| weaviate       | Y    |          |      |         |         |            |          |         | Y[^4] |
| vald[^5]       | Y    | Y        | Y    | Y       | Y       | Y          | Y        | Y       |       |

[^2]: Milvus 还支持 Tanimoto、SuperStructure、Subsctructure 三种相似性度量，都是用于衡量化学物质的相似性。
[^3]: ADB-PostgresQL 文档说只有 L2 距离支持索引查询，内积和汉明距离只支持暴力搜索。
[^4]: vespa 支持 Geodegree 相似性度量，也就是两个经纬度坐标之间的距离，这里是球面距离，而非平面上的欧式距离。
[^5]: vald 支持的相似性度量需要在服务配置中指定，不能动态变更。

**索引算法**

| 数据库         | FLAT | IVF | SQ8 | PQ  | HNSW | RHNSW | ANNOY | NSG   |
| -------------- | ---- | --- | --- | --- | ---- | ----- | ----- | ----- |
| Milvus         | Y    | Y   | Y   | Y   | Y    | Y     | Y     | ?[^6] |
| vearch         | Y    | Y   |     | Y   | Y    |       |       |       |
| 阿里云 ADB[^7] |      |     |     | Y   | Y    |       |       |       |
| pgvector       |      | Y   |     |     |      |       |       |       |
| ES             |      |     |     |     | Y    |       |       |       |
| Pinecone[^8]   |      |     |     |     |      |       |       |       |
| qdrant         |      |     |     |     | Y    |       |       |       |
| vespa          |      |     |     |     | Y    |       |       |       |
| weaviate       |      |     |     |     | Y    |       |
| vald[^9]       |      |     |     |     |      |       |       |       |

[^6]: 这篇[知乎文章](https://zhuanlan.zhihu.com/p/105594786)上介绍说支持 NSG，但文档里没找到。
[^7]: 从文档来看，ADB 似乎是把 PQ 和 HNSW 两种算法结合在一起来使用了。
[^8]: Pinecone 文档没有给出使用的索引算法。
[^9]: vald 底层使用雅虎开源的 NGT 库，使用的算法为自己发表的[ONNG、PANNG、ANNGT、ANNG 等](https://github.com/yahoojapan/NGT#publications)。

## 查询

| 数据库     | brute-force[^10] | 混合检索 | 多向量查询 | sort | aggregation |
| ---------- | ---------------- | -------- | ---------- | ---- | ----------- |
| Milvus     | Y                | Y        | ?[^11]     |      |             |
| vearch     | Y                | Y        | Y[^12]     | Y    |             |
| 阿里云 ADB | Y                | Y        | ?          | Y    | Y           |
| pgvector   | ?                | Y        | ?          | Y    | Y           |
| ES[^13]    | Y                |          |            |      |             |
| Pinecone   |                  | Y        |            |      |             |
| qdrant     |                  | Y        |            |      |             |
| vespa      |                  | Y        |            |      |             |
| weaviate   |                  | Y        |            |      |             |
| vald       |                  |          |            |      |             |

[^10]: FLAT 索引等同于暴力搜索。其他索引算法都是有精度损失的，暴力搜索虽然慢，但是可以保证精度 100%。
[^11]: 论文里说支持，但是文档里没有。根据官网文档，Collection 只支持一个向量字段，不可能做多向量查询。根据[这个 issue](https://github.com/milvus-io/milvus/issues/12680)，多向量字段应该是新版本禁掉了。
[^12]: 多个向量字段同时查询，结果是各自查询结果的交集。
[^13]: 从文档看，knn search api 是不支持跟其他字段混合检索的，但是在普通查询中可以用 dense_vector 字段给文档打分，采用的是 brute force 检索方式。多向量查询也可以用自定义打分函数的方式实现。

## 其他功能

| 数据库     | GPU | 标量索引 | 特征提取 | 多名字空间 | 分布式 | 托管服务 | 开源 | Time Travel |
| ---------- | --- | -------- | -------- | ---------- | ------ | -------- | ---- | ----------- |
| Milvus     | Y   | Y[^14]   |          | Y          | Y      |          | Y    | Y           |
| vearch     | Y   | Y        |          | Y          | Y      |          | Y    |             |
| 阿里云 ADB |     | Y        | Y[^15]   | Y          | Y      | Y        |      |             |
| pgvector   |     | Y        |          | Y          |        |          | Y    |             |
| ES         |     |          |          | Y          | Y      | Y        | Y    |             |
| Pinecone   |     |          |          | Y          |        | Y        |      |             |
| qdrant     |     | Y        |          | Y          | Y      |          | Y    |             |
| vespa      |     | Y        |          | Y          | Y      |          | Y    |             |
| weaviate   | Y   | Y        | Y        |            | Y      | Y        | Y    |             |
| vald       |     |          |          |            | Y      |          | Y    |             |

[^14]: 根据论文中的说明，Milvus 的标量属性是以列存方式存储的，按值排序的，相当于索引了。
[^15]: 示例中有用到一些特征提取函数，不确定是否已经集成在产品里，但是可以支持[注册用户自定义函数](https://help.aliyun.com/document_detail/117843.html)进行特征提取。

## 总结

从产品形态上来看，大部分产品都是在表或者文档的基础上增加向量索引功能，真正把向量作为 first class citizen 的并不多。Milvus 算一个，Pinecone 和 qdrant 勉强可以算是，vald 虽然只存储向量，但是从产品成熟度上来讲差得还比较多。

在相似性度量方面，使用频率最高的是余弦、内积和欧式距离三种。索引算法中最常用的是基于图的 HNSW，因为它对数据的更新更友好，所以更适用于数据库产品。但是 HNSW 对存储空间的占用比较大，如果要把索引放在内存来提升查询性能的话，就不能不考虑这个问题。

在查询能力上，向量和标量的混合检索是一个很重要的功能。大部分产品都会在一定程度上提供支持，在以向量为 first class citizen 的产品中，通常是以属性过滤的形式，对向量查询的结果进行二次筛选。而其他产品（比如 Elastic Search）中，可能是根据标量字段进行检索，再用 brute force 的方式对向量字段进行排序。根据 Milvus 的论文，这两种方式它都是支持的，执行时会根据代价选择最合适的查询方式。

另外一个值得关注的功能是多向量查询，Milvus 在它的论文里特别强调了多向量查询的重要性，用了一节的内容来讨论这个问题。不过我对这个问题有点疑问。在需要多个向量的场景里，是不是可以把多个向量拼接成一个大的向量，转换成单向量的索引和查询来解决？

在其他一些能力上，我认为 GPU 加速是非常值得关注的。GPU 本身就是非常适合进行向量计算的，用它来加速向量索引的构建和查询，应该都有不错的效果。
