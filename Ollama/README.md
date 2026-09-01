# GaussDB 向量检索与 Ollama 集成指南

> 适用版本：GaussDB V2.0-26.861.0（集中式 / 分布式，O模式，enable_vectordb=on）
> 更新日期：2026-08-27
> 本文所有代码与结论均已在真实环境端到端验证通过（集中式与分布式形态）

## 1. 概述

GaussDB 内置向量数据库能力，提供 `floatvector` 向量类型、`GsIVFFLAT` / `GsDiskANN` 向量索引以及向量距离算子与函数，无需安装任何扩展。

Ollama 是本地大模型运行框架，可运行 `bge-m3`、`qwen3-embedding` 等 embedding 模型生成文本向量。

两者集成的典型场景是 **RAG（检索增强生成）与语义搜索**：

```
文本 ──Ollama embedding模型──> 向量 ──写入──> GaussDB floatvector 列
                                              │
查询文本 ──Ollama──> 查询向量 ──<+>/<-> 检索──> top-k 相似文档 ──> 交给 LLM 生成
```

Ollama 负责**向量生成**，GaussDB 负责**向量存储与检索**，二者通过 Python（psycopg2 + ollama 客户端）串联。

## 2. 前提条件

| 项 | 要求 |
|---|---|
| GaussDB | 集中式实例，O模式（`datcompatibility=A`），`enable_vectordb=on`（POSTMASTER 级 GUC，需实例配置并重启） |
| 账号 | 具备建表/建索引权限的数据库账号 |
| Python | 3.11+ |
| Ollama | 已安装并启动（默认端口 11434） |
| Python 依赖 | `pip install psycopg2-binary ollama` |

拉取 embedding 模型（本文以 **bge-m3** 为例，输出 **1024 维**向量，中英文表现良好）：

```bash
ollama pull bge-m3
```

常用 embedding 模型与维度（实测）：

| 模型 | ollama 输出维度 | 说明 |
|---|---|---|
| `bge-m3` | 1024 | 中英文表现好，本文主示例 |
| `qwen3-embedding:0.6b` | **1024**（ollama 默认值） | Qwen3-Embedding 原生支持 32~4096 可变维度，ollama 打包的 0.6b 默认输出 1024；实测中文语义检索正确、单条 embedding 更快 |
| `qwen3-embedding:4b/8b` | 2560/4096 | 需要真正高维时使用（模型体积大） |

## 3. 快速开始（集中式与分布式通用）

> 本节全流程（连接、建表、索引、写入、检索）**集中式与分布式行为一致**，代码可直接复用。分布式仅两处注意：建库时 `DBCOMPATIBILITY 'ORA'`（集中式为 `'A'`）、向量维度不超过 1024（见 6.1）；建表语句可选加 `DISTRIBUTE BY HASH(...)`。

### 3.1 配置连接信息

```bash
export GAUSSDB_HOST=127.0.0.1
export GAUSSDB_PORT=5432
export GAUSSDB_USER=appuser
export GAUSSDB_PASSWORD=<password>
export GAUSSDB_DATABASE=<database>
```

### 3.2 连接 GaussDB 并生成向量

```python
import os, uuid, json
import ollama, psycopg2
from psycopg2.extras import execute_values

EMBED_MODEL = "bge-m3"
EMBED_DIM   = 1024
TABLE       = "ollama_gaussdb_documents"

conn = psycopg2.connect(
    host=os.getenv("GAUSSDB_HOST", "127.0.0.1"),
    port=int(os.getenv("GAUSSDB_PORT", "5432")),
    user=os.getenv("GAUSSDB_USER"),
    password=os.getenv("GAUSSDB_PASSWORD"),
    dbname=os.getenv("GAUSSDB_DATABASE"),
)
cur = conn.cursor()

def emb(text: str) -> list[float]:
    """调用 Ollama embedding 模型生成向量"""
    return ollama.embeddings(model=EMBED_MODEL, prompt=text)["embedding"]
```

说明：
- GaussDB 兼容 PostgreSQL 协议，psycopg2 可直连，O模式下同样适用。
- 命令行工具使用 `gsql`（而非 psql）：`gsql -h <host> -p 5432 -d <db> -U <user> -W`。

### 3.3 建表与向量索引

```python
cur.execute(f"DROP TABLE IF EXISTS {TABLE}")
cur.execute(f"""
CREATE TABLE {TABLE} (
    id varchar(36) PRIMARY KEY,          -- O模式无 gen_random_uuid()，id 由应用层生成
    text text,
    embedding floatvector({EMBED_DIM}),  -- 建表必须指定维度
    metadata jsonb                       -- 可选：来源、分类等元数据
)""")

# 建向量索引前调大会话级维护内存（默认 16MB 不足以构建索引）
cur.execute("SET maintenance_work_mem = '512MB'")
cur.execute(f"""
CREATE INDEX idx_{TABLE}_emb
ON {TABLE} USING gsivfflat (embedding cosine)
WITH (ivf_nlist = 64)
""")
conn.commit()
```

说明：
- **维度必须与 embedding 模型输出一致**：bge-m3 为 1024，qwen3-embedding:0.6b（ollama）为 1024，nomic-embed-text 为 768。插入维度不匹配的数据会报错。可先 `len(emb("测试"))` 确认实际维度再建表。
- `gsivfflat` 索引支持 `cosine` 与 `l2` 两种度量（见 4.2），维度上限 1024。
- `ivf_nlist` 为聚类中心数，数据量小时取 64~256 即可。

### 3.4 写入文档向量

```python
docs = [
    ("GaussDB 是华为的企业级分布式数据库，支持集中式和分布式两种部署形态。", {"topic": "database"}),
    ("Ollama 是一个本地大模型运行框架，可以方便地运行 llama、qwen 等开源模型。", {"topic": "llm"}),
    ("向量数据库通过 embedding 将文本映射为高维向量，再用相似度检索找到语义相近的内容。", {"topic": "vector"}),
    ("RAG 结合向量检索与 LLM 生成，先检索相关文档再生成回答。", {"topic": "rag"}),
    ("GaussDB 向量能力内置，无需安装 pgvector 等扩展。", {"topic": "database"}),
]

rows = [(str(uuid.uuid4()), text, str(emb(text)), json.dumps(meta))
        for text, meta in docs]
execute_values(cur,
    f"INSERT INTO {TABLE} (id, text, embedding, metadata) VALUES %s", rows)
conn.commit()
```

说明：
- 向量以 `"[v1,v2,...]"` 字符串形式作为参数传入，GaussDB 自动转换为 `floatvector`。
- `floatvector` 不支持 NULL/NaN/Inf 作为元素，也不支持整列为 NULL，违反时报错（`GAUSS-29756` 等）。

### 3.5 语义检索

```python
question = "如何做语义相似度搜索？"
qv = str(emb(question))

cur.execute(f"""
SELECT text,
       metadata->>'topic',
       1 - (embedding <+> %s::floatvector) AS score
FROM {TABLE}
ORDER BY embedding <+> %s::floatvector
LIMIT 3
""", (qv, qv))

for text, topic, score in cur.fetchall():
    print(f"  [{topic}] {score:.4f}  {text[:40]}")
```

输出示例（实测）：

```
  [vector]   0.6909  向量数据库通过 embedding 将文本映射为高维向量，再用相似度检索...
  [database] 0.4052  ...
  ...
```

说明：
- `<+>` 为**余弦距离**算子，`score = 1 - distance` 转换为相似度（越大越相似）。
- `LIMIT k` 即 top-k 检索；k 大于行数时返回全部行，空表返回 0 行（均实测验证）。

## 4. GaussDB 向量能力详解

### 4.1 距离度量与向量函数

Ollama 输出的 embedding 做语义检索**推荐余弦距离**（模型输出已归一化，余弦衡量方向相似性）。GaussDB 提供的完整算子/函数（均实测可用）：

| 类别 | 接口 | 说明 |
|---|---|---|
| 余弦距离 | `a <+> b` | 语义检索推荐，score = `1 - d` |
| L2 欧氏距离 | `a <-> b` | 需要绝对距离度量时使用 |
| 内积 | `inner_product(a, b)` | 点积，部分模型场景使用 |
| 负内积 | `vector_negative_inner_product(a, b)` | |
| 范数/维度 | `vector_norm(v)` / `vector_dims(v)` | |
| 向量加减 | `a + b` / `a - b` / `vector_add` / `vector_sub` | 聚合中心计算等 |
| 汉明距离 | `boolvector <#> boolvector` | 二值向量（boolvector 类型）专用 |
| 类型转换 | `floatvector(ARRAY[...])`、`vector_to_array(v)`、`text_to_vector(s)` | 数组↔向量↔文本互转 |

注意事项：
- **维度必须一致**：两个不同维度向量做距离计算会报错。
- **单精度**：`floatvector` 成员为单精度浮点；元素绝对值过大时距离计算的中间结果可能溢出，返回 NaN/Inf。
- 元素值过大/过小的 embedding 场景建议先归一化（主流 embedding 模型输出已归一化，无需处理）。

### 4.2 向量索引选择

| 索引 | 适用维度 | 度量 | 说明 |
|---|---|---|---|
| `gsivfflat` | ≤ 1024 | cosine / l2 | IVF 类索引，构建快，默认首选 |
| `gsdiskann`（无 PQ） | ≤ 1024 | l2 等 | DiskANN 类索引，大数据量更优 |
| `gsdiskann` + PQ | ≤ 4096（集中式） | l2 等 | 高维必选（见第 7 节） |

```sql
-- gsivfflat 余弦索引（本文主路径）
CREATE INDEX idx_emb ON t USING gsivfflat (embedding cosine) WITH (ivf_nlist = 64);

-- gsivfflat L2 索引
CREATE INDEX idx_emb ON t USING gsivfflat (embedding l2) WITH (ivf_nlist = 64);

-- gsdiskann 索引（默认参数）
CREATE INDEX idx_emb ON t USING gsdiskann (embedding l2);
```

索引健康检查（实测可用）：

```sql
SELECT * FROM gs_ivfflat_inspect('idx_emb');   -- 聚簇分布、磁盘使用等
SELECT * FROM gs_diskann_inspect('idx_emb');   -- PQ 压缩准确度、图连接健康度等
```

性能参考（实测：1024 维、1000 行、gsivfflat cosine）：批量插入 0.59s，索引构建 0.24s，top-5 检索延迟 **1.4ms**（无索引对照 3.8ms）。

### 4.3 数据边界行为（实测）

| 操作 | 结果 |
|---|---|
| 插入维度不匹配的向量 | 报错：`the dimension of vector is 4 rather than 3` |
| 插入 NULL 向量 | 报错：`Floatvector does not support null` |
| 插入含 NaN/Inf 元素的向量 | 报错：`NaN not allowed in vector` / `infinite not allowed in vector` |
| 对不同维度向量计算距离 | 报错：`The dimensions of the two input vectors are different` |

## 5. RAG 场景进阶

### 5.1 元数据过滤 + 向量混合检索

检索时先用 jsonb 条件过滤，再按向量距离排序（RAG 按来源/权限/分类筛选的关键能力）：

```python
cur.execute(f"""
SELECT text FROM {TABLE}
WHERE metadata->>'topic' = 'database'          -- jsonb 等值过滤
  AND metadata ? 'lang'                        -- jsonb 键存在检查
ORDER BY embedding <+> %s::floatvector
LIMIT 5
""", (qv,))
```

O模式下 jsonb 的 `->`、`->>`、`?` 操作符均实测可用。

### 5.2 向量的更新与删除

```python
# 按条件更新向量（重新 embedding 后覆盖）
cur.execute(f"UPDATE {TABLE} SET embedding = %s WHERE metadata->>'topic' = 'news'",
            (str(emb(new_text)),))

# 按元数据删除
cur.execute(f"DELETE FROM {TABLE} WHERE metadata->>'topic' = 'expired'")
```

### 5.3 向量 upsert（O模式用 MERGE INTO）

O模式不支持 `INSERT ... ON CONFLICT`，upsert 使用 `MERGE INTO`：

```python
cur.execute(f"""
MERGE INTO {TABLE} t
USING (SELECT %s::varchar AS id, %s::floatvector AS v, %s::text AS txt) s
ON t.id = s.id
WHEN MATCHED THEN UPDATE SET embedding = s.v, text = s.txt
WHEN NOT MATCHED THEN
    INSERT (id, text, embedding, metadata)
    VALUES (s.id, s.txt, s.v, '{{"topic":"synced"}}'::jsonb)
""", (doc_id, str(emb(text)), text))
```

注意：MERGE INTO 不支持 RETURNING 子句（O模式实测），需要返回行数据时在 MERGE 后单独 SELECT。

### 5.4 批量写入与检索参数调优

```python
# 批量插入：execute_values 分批（每批 200~500 条）
execute_values(cur, f"INSERT INTO {TABLE} (id, text, embedding) VALUES %s", rows)

# gsivfflat 检索时可通过 probes 控制查探的聚类数（召回率/延迟权衡）
cur.execute("SET gsivfflat_probes = 10")   # 默认较保守，加大可提升召回
```

## 6. 集中式与分布式的使用差异（分布式场景实测）

应用侧连接分布式与集中式**方式完全相同**（PG 协议、psycopg2 直连），SQL 与索引语法基本一致。以下差异均已实测验证（分布式形态引擎、O模式/ORA 兼容库、enable_vectordb=on）：

### 6.1 集中式与分布式能力对照

| 能力项 | 集中式 | 分布式 | 差异说明 |
|---|---|---|---|
| 连接方式 | psycopg2 直连实例端口 | psycopg2 直连 CN 端口 | **一致**，仅连接串不同 |
| 建库兼容模式 | `DBCOMPATIBILITY 'A'`（单字母） | **只接受长名 `ORA`/`MYSQL`/`PG`** | **有差异**：分布式不接受 'A'/'B'/'C'，报 `Compatibility args A is invalid` |
| 向量维度上限 | 列类型可达 4096 | **列类型硬限 1024**（`floatvector(1025)` 建表报错 `dimensions for type vector cannot exceed 1024`） | **有差异**：集中式限制在索引层面（gsivfflat ≤1024、gsdiskann+PQ ≤4096），分布式在列类型层面 |
| 距离算子/函数 | `<+>` `<->` inner_product 汉明等全支持 | 同左 | **一致** |
| 边界拒绝 | 维度不匹配/NULL/NaN/Inf 报错 | 同左 | **一致** |
| 向量索引 | gsivfflat（cosine/l2）、gsdiskann、gsdiskann+PQ 均支持 | gsivfflat、gsdiskann 均支持（≤1024 维）；**gsdiskann+PQ 高维不可用**（受列类型 1024 限制） | 见第 7 节（仅集中式） |
| 索引健康检查 | `gs_ivfflat_inspect` / `gs_diskann_inspect` | 同左 | **一致** |
| O模式行为 | 无 gen_random_uuid、空串→NULL、ON CONFLICT 报错（用 MERGE INTO）、jsonb 过滤可用 | 同左 | **一致**（分布式 Oracle 兼容为 ORA 模式，行为与集中式 O模式相同） |
| RAG 全链路 | Ollama embedding → 存取检索全通 | 同左（含 DISTRIBUTE BY HASH 表、中文语义检索、混合过滤、MERGE INTO upsert） | **一致** |
| 性能（1024 维 1000 条） | 插入 0.59s、索引 0.24s、检索 1.4ms | 插入 0.57s、索引 0.22s、检索 1.4ms | **相当** |
| 建表语法 | 普通 CREATE TABLE | 建议显式 `DISTRIBUTE BY HASH(分布键)` | 可选差异：分布键建议主键或查询条件列 |
| 部署 | 单机/HA | TPOPS 管理平台部署（最小 3 节点 3C3D） | 不在本文范围 |

### 6.2 分布式向量建表与 RAG 全链路（实测代码）

```python
# 连接分布式实例（与集中式仅连接串不同）
conn = psycopg2.connect(host="<CN_HOST>", port=5433, user="appuser",
                        password="<password>", dbname="dist_vec_test")

# 建库注意：分布式用长名 ORA（集中式用 'A'）
# CREATE DATABASE dist_vec_test DBCOMPATIBILITY 'ORA';

# 建表：显式指定分布键（建议主键或查询条件列）
cur.execute("""
CREATE TABLE rag_documents (
    id varchar(36) PRIMARY KEY,
    text text,
    embedding floatvector(1024),   -- 分布式硬限 1024 维
    metadata jsonb
) DISTRIBUTE BY HASH(id)
""")

# 索引、插入、检索代码与集中式完全一致（第 3 节）
```

### 6.3 应用迁移要点

从集中式迁移到分布式，应用代码仅需两处调整：
1. 建库语句的 `DBCOMPATIBILITY` 值：`'A'` → `'ORA'`。
2. 若原表维度超过 1024（高维模型场景），分布式不支持，需换用 ≤1024 维的 embedding 模型。

其余代码（连接、建表语句加 DISTRIBUTE BY 可选、索引、CRUD、检索）**无需修改**。

## 7. 高维场景：>1024 维与 GsDiskANN+PQ（仅集中式支持）

> **本场景仅集中式支持**。分布式形态的向量列类型硬限 1024 维（见 6.1），无法使用高维模型；如需高维 embedding，请选择集中式实例。

1024 维及以下（bge-m3、qwen3-embedding:0.6b 等）走 gsivfflat 即可。若在集中式上使用**输出超过 1024 维**的模型（如 qwen3-embedding:4b/8b），超过 gsivfflat 上限（实测报错：`Vector field cannot have more than 1024 dimensions for gsivfflat index`），必须使用 `gsdiskann + PQ`：

```python
EMBED_MODEL = "qwen3-embedding:8b"   # 以模型实际输出维度为准
EMBED_DIM   = len(emb("维度探测"))   # 先探测实际维度再建表

cur.execute("SET maintenance_work_mem = '512MB'")
cur.execute(f"""
CREATE INDEX idx_{TABLE}_emb
ON {TABLE} USING gsdiskann (embedding cosine)
WITH (pq_nseg = 128, pq_nclus = 16, enable_pq = true,
      quantization_type = 'pq', subgraph_count = 1, enable_vector_copy = false)
""")
```

**pq_nseg 必须能整除维度**，否则建索引报错（`Invalid product quantizer parameter`）。参考取值：768→384、1536→96、3072→96、4096→128。

高维实测结果（集中式，500 行随机向量，供参考）：

| 维度 | pq_nseg | 插入 | 索引构建 | top-5 余弦检索 |
|---|---|---|---|---|
| 1536 | 96 | 0.6s | 6.3s | 23.8ms |
| 3072 | 96 | 1.9s | 10.0s | 28.9ms |
| 4096 | 128 | 2.1s | 12.8s | 37.1ms |

另实测：1025 维列在集中式可正常建表插入，gsivfflat 拒绝建索引而 GsDiskANN+PQ 可用——即 **gsivfflat 的 1024 上限是索引层面的约束**，列本身与高维索引不受此限。

## 8. O模式（Oracle 兼容）注意事项汇总

| 项 | 说明 |
|---|---|
| `gen_random_uuid()` / `uuid_generate_v4()` | 不存在。id 用 `varchar(36)` + 应用层 `uuid.uuid4()` 生成 |
| `INSERT ... ON CONFLICT` | 不支持（语法报错）。upsert 改用 `MERGE INTO` |
| 空串 → NULL | 顶层列空串自动转 NULL，判空用 `IS NULL` 而非 `= ''` |
| jsonb 操作符 | `->` / `->>` / `?` 可用，metadata 过滤不受影响 |
| `enable_vectordb` | POSTMASTER 级 GUC，不能会话级 SET，需实例配置并重启 |
| 维度上限 | gsivfflat ≤1024；集中式 gsdiskann+PQ ≤4096；分布式建表硬限 1024 |

## 9. 常见问题

**Q: 建向量索引报 `memory required is XX MB, maintenance_work_mem is 16 MB`**
A: 建索引前执行 `SET maintenance_work_mem = '512MB';`（会话级即可）。

**Q: 插入向量报维度不一致**
A: 确认建表维度与 embedding 模型实际输出一致。不同模型维度不同（bge-m3=1024 / qwen3-embedding:0.6b=1024 / nomic-embed-text=768），且同名模型不同变体维度可能不同，建表前用 `len(emb("测试"))` 探测。

**Q: Ollama embedding 返回什么格式？**
A: `ollama.embeddings(model, prompt)["embedding"]` 返回 Python list[float]，`str()` 转为 `"[v1,v2,...]"` 字符串直接作为 SQL 参数即可。

**Q: 检索结果不准？**
A: ① 确认索引度量与查询算子匹配（cosine 索引用 `<+>`，l2 索引用 `<->`）；② 适当调大 `gsivfflat_probes`；③ 数据量过小时索引可能不生效，属正常现象。

## 10. 参考信息

- GaussDB 向量数据类型 / 向量函数和操作符 / 向量索引（GsIVFFLAT、GsDiskANN）章节：GaussDB 产品文档
- 本文实测环境：GaussDB Kernel 507.0.0（集中式 + 分布式形态实例，O模式/ORA 模式，enable_vectordb=on）+ Ollama 0.33.1 + bge-m3 / qwen3-embedding:0.6b（均 1024 维）+ psycopg2 2.9.12
- 全部 94 项场景实测通过：**集中式 62 项**（算子/函数/转换 14、边界 6、索引 4、RAG 17、高维 GsDiskANN+PQ 13、模型切换 8）+ **分布式 32 项**（形态/兼容模式/维度约束 6、算子边界索引 15、ORA 行为 5、RAG 全链路含性能 6）
