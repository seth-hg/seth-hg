---
title: "使用Milvus搭建一个古诗词搜索引擎"
date: 2022-05-15T18:59:59+08:00
tags: ["ann", "demo"]
---

作为熟悉向量数据库的一部分，这两天自己手写了一个 demo。Milvus 本身提供了很多不错的例子，但我还是想自己从头做一个。本文是开发过程的记录，心急的可以直接去看完整的[项目代码](https://github.com/seth-hg/chinese-poetry-search)。

## 准备工作

需要以下软件：

- docker 和 docker-compose，用于部署 Milvus
- sqlite3，用于存储向量之外的数据
- Python 3.7 或以上版本
- torch 和 transformers，用于 BERT 模型
- bottle，用于开发一个简单 Web 查询服务
- pymilvus，Milvus 的 Python SDK
- milvus-cli，Milvus 命令行工具

首先需要部署 Milvus 服务。由于数据量不是很大，因此使用 standealone 模式部署一个单节点服务就可以满足需求。下载 [milvus-standalone-docker-compose.yaml](https://github.com/milvus-io/milvus/releases/download/v2.0.2/milvus-standalone-docker-compose.yml) 文件到部署目录，然后执行如下命令启动服务：

```
docker-compose up -d
```

服务启动后 Milvus 的数据会存储在当前目录下的 volumes 子目录里，要确保有足够的存储空间。

也可以顺便部署一个 Milvus 的 Web 前端 Attu：

```
docker run -p 8000:3000 -e HOST_URL=http://{ your machine IP }:8000 -e MILVUS_URL={your machine IP}:19530 zilliz/attu:latest
```

接下来用 pip 安装需要的 Python 库和工具：

```
pip3 install torch transformers bottle pymilvus milvus-cli
```

本文使用的数据集是 GitHub 上的[中文诗歌古典文集数据库](https://github.com/chinese-poetry/chinese-poetry)，把整个仓库 clone 到项目目录下。

NLP 模型使用清华大学提供的预训练模型[BERT-CCPoem](https://github.com/THUNLP-AIPoet/BERT-CCPoem)，这个模型也是针对中文古诗词的，正好适合这个 demo。下载模型，并解压到项目目录下。

## 数据向量化

原始数据集里的诗歌已经整理成了 JSON 格式，不需要再做额外的处理。下面是其中一首诗：

```json
{
  "title": "玄都觀",
  "author": "徐氏",
  "biography": "",
  "paragraphs": [
    "登尋丹壑到玄都，接日紅霞照座隅。",
    "即向周回岩下看，似看曾進畫圖無。"
  ],
  "notes": [""],
  "volume": "卷九",
  "no#": 10
}
```

我们主要关注 paragraphs 字段，这里已经把诗歌分成了段落，我们把这些段落提取出来，每个段落转成一个向量用于检索，并建立向量到诗词和段落的映射关系。下面是关键的代码片段：

```python
    vector_writer = csv.writer(vector_output)
    vector_writer.writerow(["id", "feature"])
    content_writer = csv.writer(content_output)
    content_writer.writerow(["id", "content"])
    vector2poem_writer = csv.writer(vector2poem_output)
    vector2poem_writer.writerow(["id", "poem", "paragraph"])

    vector_id = 0
    poem_id = 0
    for data_file in os.listdir(args.input):
        with open(args.input + "/" + data_file) as f:
            poem_list = json.load(f)
            for poem in poem_list:
                content_writer.writerow([poem_id, json.dumps(poem)])
                vectors = embedding.predict_vec_rep(poem["paragraphs"], model,
                                                    formatter)
                for idx, v in enumerate(vectors):
                    vector2poem_writer.writerow([vector_id, poem_id, idx])
                    vector_writer.writerow([vector_id, v])
                    vector_id += 1
                poem_id += 1
                if args.num > 0 and poem_id >= args.num:
                    break
            else:
                continue
            break
```

这段代码输出 3 个 csv 文件：`content.csv` 包含了诗词的唯一 id 和 JSON 数据；`vectors.csv` 包含了向量的唯一 id 和 BERT 模型编码之后的 512 维向量；`vector2poem.csv` 包含了向量 id 到诗词 id 和段落下标的映射关系。

全唐诗一共有 4 万多首诗，分成 12 万多个段落，在没有 GPU 加速的情况下，这个转换还是挺慢的。

## 导入数据

转换完数据之后可以开始进行数据导入。首先是`vectors.csv`导入 Milvus。先创建一个名为 poetry 的 Collection，只要包含一个主键 id 和向量字段 feature 即可，向量字段的维度为 512。主键字段的 Auto ID 要关闭，因为`vectors.csv`里已经包含了唯一 id。

接下来可以导入数据到 collection。一开始尝试使用 Attu 的图形界面进行数据导入，UI 还是比较友好的，但是单个文件有 128MB 的限制，而且导入速度很慢，一个 10,000 行的文件(100MB 左右)，要 2 分多钟才能导完。看了一下发现 Milvus 后台服务负载并不高，问题可能出在 Attu 上。后来改用 milvus-cli，从部署 Milvus 的机器上导入，速度就要快很多，而且单个文件最大可以到 500MB。需要注意的是，使用 Attu 导入时，csv 文件可以没有标题行，通过 UI 指定 csv 列和字段的对应关系，但是 milvus_cli 导入时是根据标题行来映射字段的，所以每个文件都要有，并且要跟 collection 保持一致，否则会报 schema 错误。

导入命令如下：

```
# 首先要建立连接
connect -h localhost -p 19530
import -c poetry path/to/csv
```

顺便也看了一下 milvus-cli 的实现，其实就是把整个文件读到内存里，然后调用 SDK 的 insert()接口一次，批量插入全部数据。

导入完成之后还要做 2 件事。一是创建索引，这里选择 HNSW，下面索引参数仅供参考。

```
+--------------------------+----------------------+
| Corresponding Collection | poetry               |
+--------------------------+----------------------+
| Corresponding Field      | feature              |
+--------------------------+----------------------+
| Index Type               | HNSW                 |
+--------------------------+----------------------+
| Metric Type              | L2                   |
+--------------------------+----------------------+
| Params                   | M: 16                |
|                          | - efConstruction: 32 |
+--------------------------+----------------------+
```

另一件事是 load collection。Milvus 中 collection 只有 load 之后才可以进行查询。

接下来要把`poetry.csv`和`vector2poem.csv`这两个文件导入 sqlite。这里就不解释了，直接放语句：

```
sqlite3 poetry.db

sqlite> CREATE TABLE poetry(id INT PRIMARY KEY NOT NULL, content TEXT);
sqlite> CREATE TABLE vector2poem(id INT PRIMARY KEY NOT NULL, poem INT NOT NULL, paragraph INT NOT NULL);
sqlite> .mode csv
sqlite> .import --skip 1 out/content.csv poetry
sqlite> .import --skip 1 out/vector2poem.csv vector2poem
```

## 查询代码

查询逻辑在 query.py 里。下面代码是查询的主要逻辑，VdbClient 和 RdbClient 分别封装了 Milvus（向量数据库）和 sqlite（关系数据库）的查询逻辑，方便替换成其他的数据库实现。query()函数包含了跟实现无关的查询逻辑，即根据输入向量 q，从 VdbClient 查询 K 个相似向量的 id 和 score，然后在用向量 id 去 RdbClient 查诗词信息和段落下标，并将结果拼装到一起返回。

其实做完第一个版本之后，又做了一个使用[达摩院 proxima](https://github.com/alibaba/proximabilin) 的版本，代码在 git 仓库的 proxima 分支。当时更换向量数据库实现的时候，发现原来写的代码耦合太紧，更换起来比较麻烦，才改成现在这样。

```python
class VdbClient:

    def __init__(self, host, port, collection):
        connections.connect(alias="default", host=host, port=port)
        self.collection = Collection(collection)

    def query(self, column, v, n):
        search_params = {"metric_type": "L2", "params": {"ef": 32}}
        results = self.collection.search(data=[v],
                                         anns_field=column,
                                         param=search_params,
                                         limit=n,
                                         expr=None,
                                         consistency_level="Strong")
        return (results[0].ids, results[0].distances)

    def close(self):
        pass

class RdbClient:

    def __init__(self, endpoint):
        self.endpoint = endpoint

    def query(self, ids):
        conn = sqlite3.connect(self.endpoint)
        c = conn.cursor()
        cursor = c.execute(
            "SELECT vector2poem.id AS vid, poetry.id AS pid, "
            "poetry.content AS poem, vector2poem.paragraph AS paragraph "
            "FROM vector2poem,poetry "
            "WHERE vector2poem.id in (%s) AND vector2poem.poem=poetry.id;" %
            ",".join([str(x) for x in ids]))
        data = []
        for row in cursor:
            data.append({
                "vid": row[0],
                "pid": row[1],
                "content": json.loads(row[2]),
                "paragraph": row[3]
            })
        conn.close()
        return data

    def close(self):
        pass


def query(vdb, db, q):
    ids, scores = vdb.query("feature", q, 5)
    poem = db.query(ids)

    for idx, _ in enumerate(poem):
        poem[idx]["score"] = scores[idx]

    return poem
```

query.py 里还包含了命令行参数的解析，可以直接用来进行查询：

```
./query.py --milvus_host [Milvus服务地址] --milvus_port [Milvus服务端口] -c poetry -q [查询关键词]
```

## 加个前端

到这里主要任务已经完成了。为了让这个 demo 看起来更完整，我给它加了一个 web 前端，用 bottle 实现，代码在 svr.py 里。只有一个服务接口 index，通过参数 keyword 传递查询关键字。页面用 bottle 自带的模板进行渲染，加上一些简单的 css 修饰一下。页面上会展示完整的诗词，其中向量检索命中的段落会用橙色标记出来，效果如下图。

![poetry search screenshot](/figures/poetry-search.png)

## 小结

做完这个 demo 之后，我兴致勃勃的试用了很久，总体感觉还是挺惊喜的。毕竟这么一个拿来主义的 toy project 并没有花多少时间，主要是在导数据上走了弯路，浪费了比较多的时间。在这个场景下，向量检索的效果很接近全文索引，但是实现起来却要简单的多（当然前提是踩在别人的肩膀上）。用某些关键词查询的时候，会出现感觉相关度应该更高的段落，向量数据库检索结果里的评分却比较低的情况，这个跟 NLP 模型和向量检索的准确率都有关系。如果系统要做得更加完善，不能仅仅依赖向量检索的结果，毕竟向量检索得到的只是**近似**最近邻，要提高结果质量还需要更精细的筛选和排序，类似搜索引擎和推荐系统里的做法。
