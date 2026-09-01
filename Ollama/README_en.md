# GaussDB Vector Search and Ollama Integration Guide

> Applicable versions: GaussDB V2.0-26.861.0 (centralized / distributed, O-mode, enable_vectordb=on)
> Last updated: 2026-08-27
> All code and conclusions in this document have been verified end-to-end in a real environment (both centralized and distributed deployments)

## 1. Overview

GaussDB has built-in vector database capabilities, providing the `floatvector` vector type, `GsIVFFLAT` / `GsDiskANN` vector indexes, and vector distance operators and functions — no extension installation required.

Ollama is a local LLM runtime framework that can run embedding models such as `bge-m3` and `qwen3-embedding` to generate text vectors.

The typical scenario for integrating the two is **RAG (Retrieval-Augmented Generation) and semantic search**:

```
Text ──Ollama embedding model──> Vector ──write──> GaussDB floatvector column
                                                    │
Query text ──Ollama──> Query vector ──<+>/<-> retrieve──> top-k similar documents ──> hand to LLM for generation
```

Ollama is responsible for **vector generation**, while GaussDB handles **vector storage and retrieval**; the two are connected via Python (psycopg2 + the ollama client).

## 2. Prerequisites

| Item | Requirement |
|---|---|
| GaussDB | Centralized instance, O-mode (`datcompatibility=A`), `enable_vectordb=on` (a POSTMASTER-level GUC, requires instance configuration and restart) |
| Account | A database account with table/index creation permissions |
| Python | 3.11+ |
| Ollama | Installed and running (default port 11434) |
| Python dependencies | `pip install psycopg2-binary ollama` |

Pull the embedding model (this guide uses **bge-m3** as the primary example, which outputs **1024-dim** vectors and performs well for both Chinese and English):

```bash
ollama pull bge-m3
```

Common embedding models and their dimensions (tested):

| Model | Ollama output dimensions | Notes |
|---|---|---|
| `bge-m3` | 1024 | Good performance for both Chinese and English; primary example in this guide |
| `qwen3-embedding:0.6b` | **1024** (Ollama default) | Qwen3-Embedding natively supports 32–4096 variable dimensions; the 0.6b build packaged by Ollama defaults to 1024. Tested: correct Chinese semantic retrieval, faster per-item embedding |
| `qwen3-embedding:4b/8b` | 2560/4096 | Use when truly high dimensions are needed (large model size) |

## 3. Quick Start (Common to Both Centralized and Distributed)

> The entire workflow in this section (connection, table creation, indexing, insertion, retrieval) behaves **identically on centralized and distributed deployments**, and the code can be reused directly. Note only two things for distributed: use `DBCOMPATIBILITY 'ORA'` when creating the database (centralized uses `'A'`), and keep vector dimensions at or below 1024 (see 6.1). Adding `DISTRIBUTE BY HASH(...)` to the table DDL is optional.

### 3.1 Configure Connection Information

```bash
export GAUSSDB_HOST=127.0.0.1
export GAUSSDB_PORT=5432
export GAUSSDB_USER=appuser
export GAUSSDB_PASSWORD=<password>
export GAUSSDB_DATABASE=<database>
```

### 3.2 Connect to GaussDB and Generate Vectors

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
    """Generate a vector with the Ollama embedding model"""
    return ollama.embeddings(model=EMBED_MODEL, prompt=text)["embedding"]
```

Notes:
- GaussDB is compatible with the PostgreSQL protocol, so psycopg2 can connect directly; this also applies in O-mode.
- The command-line tool is `gsql` (not psql): `gsql -h <host> -p 5432 -d <db> -U <user> -W`.

### 3.3 Create Table and Vector Index

```python
cur.execute(f"DROP TABLE IF EXISTS {TABLE}")
cur.execute(f"""
CREATE TABLE {TABLE} (
    id varchar(36) PRIMARY KEY,          -- no gen_random_uuid() in O-mode; generate the id in the application
    text text,
    embedding floatvector({EMBED_DIM}),  -- the dimension must be specified at table creation
    metadata jsonb                       -- optional: source, category, and other metadata
)""")

# Increase session-level maintenance memory before creating the vector index (the default 16MB is not enough for index builds)
cur.execute("SET maintenance_work_mem = '512MB'")
cur.execute(f"""
CREATE INDEX idx_{TABLE}_emb
ON {TABLE} USING gsivfflat (embedding cosine)
WITH (ivf_nlist = 64)
""")
conn.commit()
```

Notes:
- **The dimension must match the embedding model's output**: bge-m3 is 1024, qwen3-embedding:0.6b (Ollama) is 1024, nomic-embed-text is 768. Inserting vectors with mismatched dimensions raises an error. You can check the actual dimension with `len(emb("test"))` before creating the table.
- The `gsivfflat` index supports the `cosine` and `l2` metrics (see 4.2), with a dimension limit of 1024.
- `ivf_nlist` is the number of cluster centers; 64–256 is fine for small datasets.

### 3.4 Insert Document Vectors

```python
docs = [
    ("GaussDB is Huawei's enterprise-grade distributed database, supporting both centralized and distributed deployment forms.", {"topic": "database"}),
    ("Ollama is a local LLM runtime framework that makes it easy to run open-source models like llama and qwen.", {"topic": "llm"}),
    ("A vector database maps text into high-dimensional vectors via embedding, then uses similarity search to find semantically related content.", {"topic": "vector"}),
    ("RAG combines vector retrieval with LLM generation: retrieve relevant documents first, then generate the answer.", {"topic": "rag"}),
    ("GaussDB's vector capabilities are built in — no pgvector or similar extension needed.", {"topic": "database"}),
]

rows = [(str(uuid.uuid4()), text, str(emb(text)), json.dumps(meta))
        for text, meta in docs]
execute_values(cur,
    f"INSERT INTO {TABLE} (id, text, embedding, metadata) VALUES %s", rows)
conn.commit()
```

Notes:
- Vectors are passed as `"[v1,v2,...]"` string parameters, and GaussDB converts them to `floatvector` automatically.
- `floatvector` does not allow NULL/NaN/Inf as elements, nor an entire NULL column; violations raise errors (e.g. `GAUSS-29756`).

### 3.5 Semantic Search

```python
question = "How do I do semantic similarity search?"
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

Sample output (tested):

```
  [vector]   0.6909  A vector database maps text into high-dimensional vectors via embedding, then uses similarity search to find semantically related content...
  [database] 0.4052  ...
  ...
```

Notes:
- `<+>` is the **cosine distance** operator; `score = 1 - distance` converts it to similarity (higher means more similar).
- `LIMIT k` gives top-k retrieval; when k exceeds the row count, all rows are returned, and an empty table returns 0 rows (both verified by testing).

## 4. GaussDB Vector Capabilities in Detail

### 4.1 Distance Metrics and Vector Functions

For Ollama-generated embeddings, **cosine distance is recommended** for semantic retrieval (model outputs are already normalized, and cosine measures directional similarity). The complete set of operators/functions provided by GaussDB (all tested and working):

| Category | Interface | Notes |
|---|---|---|
| Cosine distance | `a <+> b` | Recommended for semantic retrieval; score = `1 - d` |
| L2 Euclidean distance | `a <-> b` | Use when an absolute distance metric is needed |
| Inner product | `inner_product(a, b)` | Dot product, for certain model scenarios |
| Negative inner product | `vector_negative_inner_product(a, b)` | |
| Norm / dimensions | `vector_norm(v)` / `vector_dims(v)` | |
| Vector add/subtract | `a + b` / `a - b` / `vector_add` / `vector_sub` | e.g. computing aggregate centers |
| Hamming distance | `boolvector <#> boolvector` | For binary vectors (boolvector type) only |
| Type conversion | `floatvector(ARRAY[...])`, `vector_to_array(v)`, `text_to_vector(s)` | Convert between array ↔ vector ↔ text |

Precautions:
- **Dimensions must match**: computing a distance between vectors of different dimensions raises an error.
- **Single precision**: `floatvector` members are single-precision floats; with very large element magnitudes, intermediate results of distance computations may overflow, returning NaN/Inf.
- For embeddings with extremely large or small element values, normalize first (mainstream embedding model outputs are already normalized, so no processing is needed).

### 4.2 Vector Index Selection

| Index | Applicable dimensions | Metrics | Notes |
|---|---|---|---|
| `gsivfflat` | ≤ 1024 | cosine / l2 | IVF-type index, fast to build, the default first choice |
| `gsdiskann` (no PQ) | ≤ 1024 | l2 etc. | DiskANN-type index, better for large datasets |
| `gsdiskann` + PQ | ≤ 4096 (centralized) | l2 etc. | Required for high dimensions (see Section 7) |

```sql
-- gsivfflat cosine index (main path in this guide)
CREATE INDEX idx_emb ON t USING gsivfflat (embedding cosine) WITH (ivf_nlist = 64);

-- gsivfflat L2 index
CREATE INDEX idx_emb ON t USING gsivfflat (embedding l2) WITH (ivf_nlist = 64);

-- gsdiskann index (default parameters)
CREATE INDEX idx_emb ON t USING gsdiskann (embedding l2);
```

Index health checks (tested and working):

```sql
SELECT * FROM gs_ivfflat_inspect('idx_emb');   -- cluster distribution, disk usage, etc.
SELECT * FROM gs_diskann_inspect('idx_emb');   -- PQ compression accuracy, graph connectivity health, etc.
```

Performance reference (tested: 1024-dim, 1000 rows, gsivfflat cosine): batch insert 0.59s, index build 0.24s, top-5 retrieval latency **1.4ms** (3.8ms without an index).

### 4.3 Data Boundary Behavior (Tested)

| Operation | Result |
|---|---|
| Insert a vector with mismatched dimensions | Error: `the dimension of vector is 4 rather than 3` |
| Insert a NULL vector | Error: `Floatvector does not support null` |
| Insert a vector containing NaN/Inf elements | Error: `NaN not allowed in vector` / `infinite not allowed in vector` |
| Compute a distance between vectors of different dimensions | Error: `The dimensions of the two input vectors are different` |

## 5. Advanced RAG Scenarios

### 5.1 Metadata Filtering + Vector Hybrid Retrieval

During retrieval, first filter with jsonb conditions, then sort by vector distance (a key capability for filtering RAG by source/permission/category):

```python
cur.execute(f"""
SELECT text FROM {TABLE}
WHERE metadata->>'topic' = 'database'          -- jsonb equality filter
  AND metadata ? 'lang'                        -- jsonb key existence check
ORDER BY embedding <+> %s::floatvector
LIMIT 5
""", (qv,))
```

The jsonb operators `->`, `->>`, and `?` all work in O-mode (tested).

### 5.2 Updating and Deleting Vectors

```python
# Update vectors by condition (re-embed and overwrite)
cur.execute(f"UPDATE {TABLE} SET embedding = %s WHERE metadata->>'topic' = 'news'",
            (str(emb(new_text)),))

# Delete by metadata
cur.execute(f"DELETE FROM {TABLE} WHERE metadata->>'topic' = 'expired'")
```

### 5.3 Vector Upsert (MERGE INTO in O-Mode)

O-mode does not support `INSERT ... ON CONFLICT`; use `MERGE INTO` for upserts:

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

Note: MERGE INTO does not support the RETURNING clause (tested in O-mode); when you need row data back, run a separate SELECT after the MERGE.

### 5.4 Batch Insertion and Retrieval Parameter Tuning

```python
# Batch insert: execute_values in chunks (200–500 rows per batch)
execute_values(cur, f"INSERT INTO {TABLE} (id, text, embedding) VALUES %s", rows)

# For gsivfflat retrieval, control the number of probed clusters with probes (recall/latency trade-off)
cur.execute("SET gsivfflat_probes = 10")   # conservative by default; increase to improve recall
```

## 6. Differences Between Centralized and Distributed Usage (Distributed Tested)

From the application side, connecting to distributed and centralized deployments is **exactly the same** (PG protocol, direct psycopg2 connection), and SQL and index syntax are largely identical. All differences below were verified by testing (distributed engine, O-mode/ORA-compatible databases, enable_vectordb=on):

### 6.1 Centralized vs. Distributed Capability Comparison

| Capability | Centralized | Distributed | Difference |
|---|---|---|---|
| Connection method | psycopg2 direct connection to the instance port | psycopg2 direct connection to the CN port | **Identical**; only the connection string differs |
| Database compatibility mode | `DBCOMPATIBILITY 'A'` (single letter) | **Only long names `ORA`/`MYSQL`/`PG` accepted** | **Different**: distributed rejects 'A'/'B'/'C' with `Compatibility args A is invalid` |
| Vector dimension limit | Column type up to 4096 | **Column type hard-limited to 1024** (`floatvector(1025)` fails at table creation with `dimensions for type vector cannot exceed 1024`) | **Different**: centralized limits live at the index level (gsivfflat ≤1024, gsdiskann+PQ ≤4096); distributed at the column-type level |
| Distance operators/functions | `<+>`, `<->`, inner_product, Hamming, etc. all supported | Same as left | **Identical** |
| Boundary rejection | Errors on dimension mismatch / NULL / NaN / Inf | Same as left | **Identical** |
| Vector indexes | gsivfflat (cosine/l2), gsdiskann, gsdiskann+PQ all supported | gsivfflat and gsdiskann supported (≤1024 dims); **gsdiskann+PQ unavailable for high dimensions** (limited by the 1024 column-type cap) | See Section 7 (centralized only) |
| Index health checks | `gs_ivfflat_inspect` / `gs_diskann_inspect` | Same as left | **Identical** |
| O-mode behavior | No gen_random_uuid; empty string → NULL; ON CONFLICT errors (use MERGE INTO); jsonb filtering works | Same as left | **Identical** (distributed Oracle compatibility is the ORA mode, which behaves the same as the centralized O-mode) |
| Full RAG pipeline | Ollama embedding → full storage/retrieval works | Same as left (including DISTRIBUTE BY HASH tables, Chinese semantic search, hybrid filtering, MERGE INTO upsert) | **Identical** |
| Performance (1024-dim, 1000 rows) | Insert 0.59s, index 0.24s, retrieval 1.4ms | Insert 0.57s, index 0.22s, retrieval 1.4ms | **Comparable** |
| Table DDL | Plain CREATE TABLE | Explicit `DISTRIBUTE BY HASH(distribution key)` recommended | Optional difference: use the primary key or a frequently queried column as the distribution key |
| Deployment | Standalone / HA | TPOPS-managed deployment (minimum 3 nodes, 3C3D) | Out of scope for this guide |

### 6.2 Distributed Table Creation and Full RAG Pipeline (Tested Code)

```python
# Connect to the distributed instance (only the connection string differs from centralized)
conn = psycopg2.connect(host="<CN_HOST>", port=5433, user="appuser",
                        password="<password>", dbname="dist_vec_test")

# Note for database creation: distributed uses the long name ORA (centralized uses 'A')
# CREATE DATABASE dist_vec_test DBCOMPATIBILITY 'ORA';

# Create the table: specify the distribution key explicitly (primary key or frequently queried column recommended)
cur.execute("""
CREATE TABLE rag_documents (
    id varchar(36) PRIMARY KEY,
    text text,
    embedding floatvector(1024),   -- distributed hard limit: 1024 dims
    metadata jsonb
) DISTRIBUTE BY HASH(id)
""")

# Indexing, insertion, and retrieval are identical to centralized (Section 3)
```

### 6.3 Application Migration Notes

Migrating from centralized to distributed requires only two application changes:
1. The `DBCOMPATIBILITY` value in the database creation statement: `'A'` → `'ORA'`.
2. If the original tables use dimensions above 1024 (high-dim model scenarios), distributed does not support them — switch to an embedding model with ≤1024 dimensions.

The rest of the code (connection, table creation with an optional DISTRIBUTE BY clause, indexing, CRUD, retrieval) **needs no changes**.

## 7. High-Dimensional Scenarios: >1024 Dimensions with GsDiskANN+PQ (Centralized Only)

> **This scenario is supported on centralized deployments only.** The vector column type is hard-limited to 1024 dimensions on distributed deployments (see 6.1), so high-dimensional models cannot be used there; choose a centralized instance if you need high-dimensional embeddings.

For 1024 dimensions and below (bge-m3, qwen3-embedding:0.6b, etc.), gsivfflat is sufficient. If you use a model on centralized whose **output exceeds 1024 dimensions** (e.g. qwen3-embedding:4b/8b), you exceed the gsivfflat limit (tested error: `Vector field cannot have more than 1024 dimensions for gsivfflat index`) and must use `gsdiskann + PQ`:

```python
EMBED_MODEL = "qwen3-embedding:8b"   # defer to the model's actual output dimension
EMBED_DIM   = len(emb("dimension probe"))   # probe the actual dimension before creating the table

cur.execute("SET maintenance_work_mem = '512MB'")
cur.execute(f"""
CREATE INDEX idx_{TABLE}_emb
ON {TABLE} USING gsdiskann (embedding cosine)
WITH (pq_nseg = 128, pq_nclus = 16, enable_pq = true,
      quantization_type = 'pq', subgraph_count = 1, enable_vector_copy = false)
""")
```

**`pq_nseg` must divide the dimension evenly**, otherwise index creation fails (`Invalid product quantizer parameter`). Reference values: 768→384, 1536→96, 3072→96, 4096→128.

High-dimensional test results (centralized, 500 random vectors, for reference):

| Dimensions | pq_nseg | Insert | Index build | Top-5 cosine retrieval |
|---|---|---|---|---|
| 1536 | 96 | 0.6s | 6.3s | 23.8ms |
| 3072 | 96 | 1.9s | 10.0s | 28.9ms |
| 4096 | 128 | 2.1s | 12.8s | 37.1ms |

Also tested: a 1025-dim column can be created and inserted into on centralized; gsivfflat refuses to build the index while GsDiskANN+PQ works — meaning **the 1024 limit on gsivfflat is an index-level constraint**, and the column itself and high-dimensional indexes are not subject to it.

## 8. O-Mode (Oracle Compatibility) Notes Summary

| Item | Notes |
|---|---|
| `gen_random_uuid()` / `uuid_generate_v4()` | Not available. Use `varchar(36)` for the id and generate it with `uuid.uuid4()` in the application |
| `INSERT ... ON CONFLICT` | Not supported (syntax error). Use `MERGE INTO` for upserts |
| Empty string → NULL | Empty strings in top-level columns are automatically converted to NULL; test for emptiness with `IS NULL`, not `= ''` |
| jsonb operators | `->` / `->>` / `?` work; metadata filtering is unaffected |
| `enable_vectordb` | POSTMASTER-level GUC, cannot be SET at session level; requires instance configuration and restart |
| Dimension limits | gsivfflat ≤1024; centralized gsdiskann+PQ ≤4096; distributed table creation hard-limited to 1024 |

## 9. FAQ

**Q: Creating a vector index fails with `memory required is XX MB, maintenance_work_mem is 16 MB`**
A: Run `SET maintenance_work_mem = '512MB';` before creating the index (session level is enough).

**Q: Inserting a vector fails with a dimension mismatch**
A: Make sure the table's dimension matches the embedding model's actual output. Different models have different dimensions (bge-m3=1024 / qwen3-embedding:0.6b=1024 / nomic-embed-text=768), and variants of the same model may differ as well; probe with `len(emb("test"))` before creating the table.

**Q: What format does Ollama embedding return?**
A: `ollama.embeddings(model, prompt)["embedding"]` returns a Python list[float]; convert it with `str()` into a `"[v1,v2,...]"` string and pass it directly as an SQL parameter.

**Q: Retrieval results are inaccurate?**
A: ① Make sure the index metric matches the query operator (a cosine index uses `<+>`, an l2 index uses `<->`); ② increase `gsivfflat_probes` appropriately; ③ with very little data, the index may not take effect — this is normal.

## 10. References

- Vector data type / Vector functions and operators / Vector indexes (GsIVFFLAT, GsDiskANN) sections: GaussDB product documentation
- Tested environment: GaussDB Kernel 507.0.0 (centralized + distributed instances, O-mode/ORA mode, enable_vectordb=on) + Ollama 0.33.1 + bge-m3 / qwen3-embedding:0.6b (both 1024-dim) + psycopg2 2.9.12
- All 94 test scenarios passed: **62 on centralized** (operators/functions/conversions 14, boundaries 6, indexes 4, RAG 17, high-dim GsDiskANN+PQ 13, model switching 8) + **32 on distributed** (form/compat mode/dimension constraints 6, operator/boundary/index 15, ORA behavior 5, full RAG pipeline incl. performance 6)
