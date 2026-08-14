# `langgraph-checkpoint-gaussdb` 用户手册

| 项目 | 适用范围 |
| --- | --- |
| 软件包 | `langgraph-checkpoint-gaussdb 0.1.0` |
| Python | 3.10 及以上 |
| LangGraph 示例基线 | 1.2.9 |
| 数据库 | GaussDB 集中式 A 兼容或分布式 ORA 兼容 |

## 1. 手册说明

`langgraph-checkpoint-gaussdb` 为 LangGraph 应用提供两类持久化能力：

- `GaussDBSaver` 保存图运行状态，使同一 thread 可以恢复、继续执行、查看历史和删除状态。
- `GaussDBStore` 保存由应用管理的长期数据，支持 namespace/key、JSON 条件、向量检索和 TTL。

Checkpoint 与 Store 是彼此独立的数据域。Checkpoint 表示图执行到了哪里，Store 保存用户偏好、知识、记忆等业务数据。删除 checkpoint thread 不会删除 Store 数据，清理 Store 过期项也不会影响 checkpoint。

本手册面向应用开发、部署和运维人员，内容按照实际使用顺序组织。首次接入时，建议依次完成安装、数据库准备、安全连接、`setup()` 和快速入门示例，再按业务需要启用向量、TTL 或异步接口。

### 1.1 功能范围

| 使用需求 | 入口 |
| --- | --- |
| 保存并恢复 LangGraph 状态 | `builder.compile(checkpointer=saver)` |
| 查看当前状态和历史状态 | `graph.get_state()`、`graph.get_state_history()` |
| 从历史 checkpoint 继续运行 | 使用历史 snapshot 的 `config` 调用 `invoke()` |
| 删除完整 thread 历史 | `saver.delete_thread()` |
| 保存和读取长期 JSON 数据 | `store.put()`、`store.get()` |
| 按条件检索长期数据 | `store.search(filter=...)` |
| 执行向量语义检索 | `store.search(query=...)` |
| 创建 ANN 索引 | `store.create_vector_index()` |
| 设置数据有效期 | `put(..., ttl=...)` |
| 清理过期数据 | `sweep_ttl()` 或 TTL sweeper |
| 在异步应用中使用 | 同一对象上的 `a...` 方法 |

版本 0.1.0 不提供独立的 Async Saver/Store 类型，也不提供 BM25/hybrid 检索、Shallow Saver、`copy_thread`、`delete_for_runs` 或 `prune`。

## 2. 安装

### 2.1 从源码安装

版本 0.1.0 的源码位于 [`huaweicloud-samples/langgraph-gaussdb` 仓库的 `v0.1.0` 分支](https://github.com/huaweicloud-samples/langgraph-gaussdb/tree/v0.1.0)。克隆该分支并从仓库根目录安装：

```bash
git clone \
  --branch v0.1.0 \
  --single-branch \
  https://github.com/huaweicloud-samples/langgraph-gaussdb.git

cd langgraph-gaussdb
python -m pip install .
```

软件包会安装以下运行时依赖：

- `langgraph-checkpoint>=4.1,<5.0`
- `psycopg2-binary>=2.9.11,<3.0`

如果应用需要编译和运行 `StateGraph`，还必须安装 LangGraph。本文示例使用 1.2.9：

```bash
python -m pip install "langgraph==1.2.9"
```

### 2.2 验证安装

运行以下命令确认版本和公开导入：

```bash
python - <<'PY'
from importlib.metadata import version

from langgraph.checkpoint.gaussdb import GaussDBSaver
from langgraph.store.gaussdb import GaussDBIndexConfig, GaussDBStore

print("langgraph-checkpoint-gaussdb:", version("langgraph-checkpoint-gaussdb"))
print("Saver:", GaussDBSaver.__name__)
print("Store:", GaussDBStore.__name__)
print("Index config:", GaussDBIndexConfig.__name__)
PY
```

预期的软件包版本为 `0.1.0`。

## 3. 准备数据库

### 3.1 创建业务 schema

`setup()` 会创建和维护所需数据库对象，但不会创建 `schema_name` 指定的 schema。部署前由 DBA 创建 schema，例如：

```sql
CREATE SCHEMA agent_runtime;
```

生产环境建议显式设置 `schema_name`，并确保所有应用连接都能访问同一个 schema。

### 3.2 配置账号权限

部署账号需要在目标 schema 中创建和检查数据库对象。业务运行账号需要对 Saver 和 Store 使用的对象执行 SELECT、INSERT、UPDATE 和 DELETE。

可以采用以下权限分工：

1. 部署阶段使用具备 DDL 权限的账号执行 `saver.setup()` 和 `store.setup()`。
2. 业务阶段使用只具备所需 DML 权限的账号运行应用。
3. schema 结构发生版本变更时，再由部署账号执行 `setup()`。

普通 `put()`、`get()` 和 `search()` 不会自动执行 setup。

### 3.3 确认连接允许写入

业务连接必须允许写 transaction。推荐在 DSN 或连接参数中设置：

```text
options='-c default_transaction_read_only=off'
```

该设置不能把物理只读节点变成可写节点，也不能补充账号缺少的 DDL 或 DML 权限。连接仍然无法写入时，需要检查节点角色和数据库授权。

### 3.4 向量功能前置条件

只有使用向量检索时才需要检查以下项目：

```sql
SHOW enable_vectordb;

SELECT typname
FROM pg_catalog.pg_type
WHERE lower(typname) = 'floatvector';
```

`enable_vectordb` 应处于开启状态，并且查询应返回 `floatvector` 类型。

## 4. 配置数据库连接

### 4.1 使用环境变量保存 DSN

DSN 应由 Secret Manager、部署平台或环境变量提供，不要写入源码。在本地 Bash 中可以安全输入：

```bash
read -rsp "GaussDB DSN: " GAUSSDB_DSN
printf '\n'
export GAUSSDB_DSN
```

输入内容使用 libpq keyword/value 格式，例如：

```text
host=db.example.internal port=8000 dbname=agent_app user=langgraph_app password='replace-me' connect_timeout=10 application_name=langgraph-gaussdb options='-c default_transaction_read_only=off'
```

示例中的地址、账号和密码必须替换为实际值。密码包含单引号或反斜杠时，使用第 4.4 节的 kwargs mapping 或 `make_dsn()` 生成 DSN。

`sslmode` 必须与目标实例的 TLS 配置一致。只有数据库已经启用 TLS 时才使用 `require` 或更严格的模式。

### 4.2 测试连接

运行以下只读检查：

```bash
python - <<'PY'
import os

import psycopg2

dsn = os.environ.get("GAUSSDB_DSN")
if not dsn:
    raise RuntimeError("GAUSSDB_DSN is not configured")

connection = psycopg2.connect(dsn)
try:
    with connection.cursor() as cursor:
        cursor.execute("SELECT 1")
        print("connection:", cursor.fetchone()[0])

        cursor.execute("SHOW sql_compatibility")
        print("sql_compatibility:", cursor.fetchone()[0])

        cursor.execute("SHOW default_transaction_read_only")
        print("default_transaction_read_only:", cursor.fetchone()[0])
finally:
    try:
        connection.rollback()
    finally:
        connection.close()
PY
```

`sql_compatibility` 应为 `A` 或 `ORA`，`default_transaction_read_only` 应为 `off`。

### 4.3 让 Saver 或 Store 管理连接池

推荐使用 `from_conn_string()` 创建内部连接池：

```python
import os

from langgraph.checkpoint.gaussdb import GaussDBSaver
from langgraph.store.gaussdb import GaussDBStore

dsn = os.environ["GAUSSDB_DSN"]

with GaussDBSaver.from_conn_string(
    dsn,
    schema_name="agent_runtime",
    minconn=1,
    maxconn=8,
    max_workers=8,
) as saver:
    saver.setup()

with GaussDBStore.from_conn_string(
    dsn,
    schema_name="agent_runtime",
    minconn=1,
    maxconn=8,
    max_workers=8,
) as store:
    store.setup()
```

`minconn` 和 `maxconn` 控制数据库连接数量，`max_workers` 控制可执行任务的工作线程数量。

也可以直接把 DSN 传给构造函数。此时连接池参数名称为 `min_size` 和 `max_size`：

```python
with GaussDBSaver(
    dsn,
    schema_name="agent_runtime",
    min_size=1,
    max_size=8,
    max_workers=8,
) as saver:
    saver.setup()
```

### 4.4 使用连接参数 mapping

连接参数较多或需要安全处理特殊字符时，可以传入 mapping：

```python
import os

from langgraph.store.gaussdb import GaussDBStore

connection_kwargs = {
    "host": os.environ["GAUSSDB_HOST"],
    "port": int(os.environ.get("GAUSSDB_PORT", "8000")),
    "dbname": os.environ["GAUSSDB_DATABASE"],
    "user": os.environ["GAUSSDB_USER"],
    "password": os.environ["GAUSSDB_PASSWORD"],
    "connect_timeout": 10,
    "application_name": "langgraph-gaussdb",
    "options": "-c default_transaction_read_only=off",
}

sslmode = os.environ.get("GAUSSDB_SSLMODE")
if sslmode:
    connection_kwargs["sslmode"] = sslmode

with GaussDBStore(
    connection_kwargs,
    schema_name="agent_runtime",
    min_size=1,
    max_size=8,
    max_workers=8,
) as store:
    store.setup()
```

连接参数属于 psycopg2/libpq。`schema_name`、`min_size`、`max_size`、`max_workers`、`index` 和 `ttl` 属于 Saver/Store，不能混放。

需要基于现有 DSN 增加连接参数时，使用 `make_dsn()`：

```python
import os

from psycopg2.extensions import make_dsn

dsn = make_dsn(
    os.environ["GAUSSDB_DSN"],
    connect_timeout=10,
    application_name="langgraph-gaussdb",
    options="-c default_transaction_read_only=off",
)
```

### 4.5 共享外部连接池

Saver 和 Store 可以共享一个 `ThreadedConnectionPool`：

```python
import os

from psycopg2.pool import ThreadedConnectionPool

from langgraph.checkpoint.gaussdb import GaussDBSaver
from langgraph.store.gaussdb import GaussDBStore

pool = ThreadedConnectionPool(
    1,
    8,
    dsn=os.environ["GAUSSDB_DSN"],
    options="-c default_transaction_read_only=off",
)

saver = GaussDBSaver(
    pool,
    schema_name="agent_runtime",
    max_workers=8,
)
store = GaussDBStore(
    pool,
    schema_name="agent_runtime",
    max_workers=8,
)

try:
    saver.setup()
    store.setup()
    # 在这里编译并运行图。
finally:
    store.close(timeout=30)
    saver.close(timeout=30)
    pool.closeall()
```

外部 pool 由创建它的应用关闭。不要在 Saver/Store 完成 `close()` 前调用 `pool.closeall()`。

只使用线程安全的 `ThreadedConnectionPool`。`SimpleConnectionPool` 不适合并发使用，会被拒绝。

### 4.6 连接方式参数速查

| 参数 | 适用入口 | 说明 |
| --- | --- | --- |
| `schema_name` | Saver/Store | 业务对象所在 schema |
| `minconn/maxconn` | `from_conn_string()` | 内部 pool 的连接上下限 |
| `min_size/max_size` | 直接构造 | 内部 pool 的连接上下限 |
| `max_workers` | Saver/Store | 内部线程池大小 |
| `executor` | Saver/Store | 应用提供的 `ThreadPoolExecutor` |
| `owns_connection` | 外部 pool/connection | 是否由对象关闭外部资源 |
| `serde` | Saver | 自定义 checkpoint serializer |
| `async_page_size` | Saver | 历史列表的内部页大小 |
| `index` | Store | 向量配置 |
| `ttl` | Store | TTL 默认行为 |

不能同时传 `executor` 和 `max_workers`。外部 pool 的连接数量由 pool 创建者配置。

## 5. 初始化 Saver 和 Store

Saver 与 Store 使用不同的数据对象，需要分别执行 setup：

```python
saver.setup()
store.setup()
```

`setup()` 可以重复执行。首次部署、应用升级和 schema 检查时都可以调用。发现同名但不兼容的数据库对象时，setup 会失败，应用不应绕过错误继续运行。

构造 Store 时提供 `index` 后，`store.setup()` 还会准备向量数据对象，但不会创建 ANN 索引。ANN 索引由第 9 章的 `create_vector_index()` 显式创建。

## 6. 使用 Checkpointer

### 6.1 保存并恢复图状态

下面是一个完整的最小示例。相同 `thread_id` 的第二次调用会恢复第一次保存的状态，因此 `count` 从 1 增长到 2：

```python
import operator
import os
from typing import Annotated

from langgraph.graph import START, StateGraph
from typing_extensions import TypedDict

from langgraph.checkpoint.gaussdb import GaussDBSaver


class CounterState(TypedDict, total=False):
    count: Annotated[int, operator.add]
    events: Annotated[list[str], operator.add]


def increment(state: CounterState) -> CounterState:
    return {
        "count": 1,
        "events": [f"node-from-{state['count']}"],
    }


builder = StateGraph(CounterState)
builder.add_node("increment", increment)
builder.add_edge(START, "increment")

with GaussDBSaver.from_conn_string(
    os.environ["GAUSSDB_DSN"],
    schema_name="agent_runtime",
    minconn=1,
    maxconn=8,
    max_workers=8,
) as saver:
    saver.setup()
    graph = builder.compile(checkpointer=saver)
    config = {"configurable": {"thread_id": "thread-1"}}

    first = graph.invoke(
        {"count": 0, "events": ["first-input"]},
        config,
        durability="sync",
    )
    second = graph.invoke(
        {"count": 0, "events": ["second-input"]},
        config,
        durability="sync",
    )

    assert first["count"] == 1
    assert second["count"] == 2
```

`thread_id` 是状态恢复边界。复用同一个值表示继续同一条历史，使用新值表示创建独立历史。`checkpoint_ns` 可以在同一 thread 中进一步隔离状态；省略时使用逻辑空字符串。

`thread_id` 和 `checkpoint_ns` 必须是字符串，不能包含 NUL。

### 6.2 读取当前状态

在 Saver 仍然打开时，通过编译后的 graph 读取状态：

```python
snapshot = graph.get_state(config)

print(snapshot.values)
print(snapshot.config)
print(snapshot.metadata)
```

需要直接检查持久化内容时，可以使用：

```python
checkpoint_tuple = saver.get_tuple(config)

if checkpoint_tuple is not None:
    print(checkpoint_tuple.checkpoint)
    print(checkpoint_tuple.metadata)
    print(checkpoint_tuple.pending_writes)
```

config 只包含 `thread_id` 和 `checkpoint_ns` 时返回最新 checkpoint；同时提供 `checkpoint_id` 时返回指定 checkpoint，不存在时返回 `None`。

### 6.3 查看历史

`graph.get_state_history()` 按最新到最旧返回状态：

```python
recent = list(
    graph.get_state_history(
        config,
        limit=10,
    )
)

for snapshot in recent:
    checkpoint_id = snapshot.config["configurable"]["checkpoint_id"]
    print(checkpoint_id, snapshot.values, snapshot.metadata)
```

可以按 metadata 过滤，也可以使用 `before` 继续读取更早历史：

```python
loop_states = list(
    graph.get_state_history(
        config,
        filter={"source": "loop"},
        limit=10,
    )
)

older = list(
    graph.get_state_history(
        config,
        before=recent[-1].config,
        limit=10,
    )
)
```

直接使用 Saver 时，对应入口为 `saver.list()`。

### 6.4 从历史状态继续运行

历史 snapshot 的 `config` 包含确定的 checkpoint id。将它传给下一次调用即可从该时间点形成分支：

```python
base = recent[-1]

forked = graph.invoke(
    {"count": 100, "events": ["fork-input"]},
    base.config,
    durability="sync",
)
```

该操作不会覆盖原有历史。

### 6.5 删除 thread

确认不再保留某条运行历史后执行：

```python
saver.delete_thread("thread-1")
```

删除覆盖该 thread 的所有 checkpoint namespace。重复删除不存在的 thread 不会失败。

该操作不会删除 Store 数据。应用需要单独定义长期数据的保留和删除规则。

### 6.6 Saver API 速查

| 同步方法 | 异步方法 | 用途 |
| --- | --- | --- |
| `setup()` | `asetup()` | 初始化和检查数据库对象 |
| `get_tuple()` | `aget_tuple()` | 读取指定或最新 checkpoint |
| `list()` | `alist()` | 列出 checkpoint 历史 |
| `delete_thread()` | `adelete_thread()` | 删除完整 thread 历史 |
| `get_delta_channel_history()` | `aget_delta_channel_history()` | 读取 DeltaChannel 历史 |
| `close()` | `aclose()` | 等待任务结束并关闭自有资源 |

`put()`、`put_writes()` 和 `get_next_version()` 通常由 LangGraph runtime 调用，普通应用不需要手工调用。

## 7. 使用 Store

### 7.1 数据模型和约束

Store item 由以下字段标识：

- `namespace: tuple[str, ...]`：层级范围。
- `key: str`：namespace 内的唯一键。
- `value: dict[str, Any]`：JSON object。
- `created_at` 和 `updated_at`：数据库时间戳。
- 可选 TTL 和向量数据。

namespace 必须非空。每个 component 必须是非空字符串，不能包含 `.` 或 NUL，根 component 不能是 `langgraph`。key 可以是逻辑空字符串，但不能包含 NUL。value 顶层必须是字典，且必须能严格序列化为 JSON；NaN 和 Infinity 不被接受。

### 7.2 写入、覆盖和读取

```python
namespace = ("users", "u-42", "preferences")

store.put(
    namespace,
    "response-style",
    {
        "style": "concise",
        "language": "zh-CN",
        "priority": 10,
    },
)

item = store.get(
    namespace,
    "response-style",
    refresh_ttl=False,
)

if item is not None:
    print(item.namespace)
    print(item.key)
    print(item.value)
    print(item.created_at, item.updated_at)
```

再次写入相同 namespace/key 会完整替换 value，不会合并旧字段。`created_at` 保持不变，`updated_at` 更新。

读取不存在的 key 返回 `None`：

```python
assert store.get(namespace, "missing") is None
```

删除是幂等操作：

```python
store.delete(namespace, "response-style")
store.delete(namespace, "response-style")
```

### 7.3 结构化搜索

`search()` 的 namespace 参数按 component 前缀匹配自身和后代：

```python
store.put(
    ("users", "u-42", "memories"),
    "m-1",
    {
        "kind": "preference",
        "text": "likes coffee",
        "score": 0.9,
    },
)

store.put(
    ("users", "u-42", "memories", "archive"),
    "m-2",
    {
        "kind": "event",
        "text": "visited Hangzhou",
        "score": 0.7,
    },
)

results = store.search(
    ("users", "u-42", "memories"),
    filter={
        "kind": "preference",
        "score": {"$gte": 0.8},
    },
    limit=20,
    offset=0,
    refresh_ttl=False,
)
```

空 tuple 表示搜索整个 Store：

```python
all_high_score = store.search(
    (),
    filter={"score": {"$gt": 0.8}},
    limit=100,
)
```

filter 只匹配 value 的顶层 key。支持以下形式：

| 形式 | 说明 |
| --- | --- |
| `{"kind": "event"}` | 精确相等 |
| `{"kind": {"$eq": "event"}}` | 精确相等 |
| `{"kind": {"$ne": "deleted"}}` | key 存在且不等于指定值 |
| `{"score": {"$gt": 0.5}}` | 大于 |
| `{"score": {"$gte": 0.5}}` | 大于等于 |
| `{"score": {"$lt": 1.0}}` | 小于 |
| `{"score": {"$lte": 1.0}}` | 小于等于 |
| `{"profile": {"country": "CN"}}` | nested object 整体相等 |

多个条件使用 AND。结构化搜索的 `score` 为 `None`。

`limit` 和 `offset` 必须是非负整数。`limit=0` 返回空列表。

### 7.4 列出 namespace

```python
namespaces = store.list_namespaces(
    prefix=("users", "u-42", "*"),
    suffix=("memories",),
    max_depth=4,
    limit=100,
    offset=0,
)
```

`*` 表示该位置匹配任意 component。`max_depth` 会截断更深 namespace 后去重。只有包含至少一个 item 的 namespace 才会出现在结果中。

### 7.5 批量操作

需要在一次调用中提交多个 Store 操作时使用 `batch()`：

```python
from langgraph.store.base import GetOp, PutOp, SearchOp

results = store.batch(
    [
        GetOp(
            namespace=("users", "u-42"),
            key="profile",
            refresh_ttl=False,
        ),
        SearchOp(
            namespace_prefix=("users", "u-42"),
            filter={"kind": "preference"},
            limit=10,
            refresh_ttl=False,
        ),
        PutOp(
            namespace=("users", "u-42"),
            key="profile",
            value={"name": "Alice"},
        ),
    ]
)

old_profile = results[0]
preferences = results[1]
assert results[2] is None
```

返回值顺序与输入顺序一致。同一 batch 中的读取看到的是 batch 开始时的状态，不会看到同一 batch 内的写入。同一物理 key 有多个 Put 时，最后一个值生效。

### 7.6 Store API 速查

| 同步方法 | 异步方法 | 用途 |
| --- | --- | --- |
| `get()` | `aget()` | 精确读取 item |
| `put()` | `aput()` | 写入或替换 item |
| `delete()` | `adelete()` | 删除 item |
| `search()` | `asearch()` | 结构化或向量搜索 |
| `list_namespaces()` | `alist_namespaces()` | 列出 namespace |
| `batch()` | `abatch()` | 执行批量操作 |
| `setup()` | `asetup()` | 初始化和检查数据库对象 |
| `close()` | `aclose()` | 等待任务结束并关闭自有资源 |

## 8. 在 LangGraph 节点中使用 Store

将 Store 传给 `compile()` 后，节点可以通过 `runtime.store` 访问长期数据：

```python
from dataclasses import dataclass

from langchain_core.runnables import RunnableConfig
from langgraph.graph import START, StateGraph
from langgraph.runtime import Runtime
from typing_extensions import TypedDict


@dataclass(frozen=True)
class Context:
    user_id: str


class MemoryState(TypedDict, total=False):
    key: str
    text: str
    found: list[str]


def remember(
    state: MemoryState,
    config: RunnableConfig,
    runtime: Runtime[Context],
) -> MemoryState:
    namespace = ("memories", runtime.context.user_id)

    runtime.store.put(
        namespace,
        state["key"],
        {
            "text": state["text"],
            "source_thread": config["configurable"]["thread_id"],
        },
    )

    found = runtime.store.search(namespace, limit=10)
    return {"found": [item.key for item in found]}


builder = StateGraph(MemoryState, context_schema=Context)
builder.add_node("remember", remember)
builder.add_edge(START, "remember")

graph = builder.compile(store=store)

result = graph.invoke(
    {"key": "m-1", "text": "likes coffee"},
    {"configurable": {"thread_id": "thread-1"}},
    context=Context(user_id="u-42"),
)
```

namespace 是否按用户、组织或 thread 隔离由应用的数据模型决定。Saver 和 Store 可以同时注入：

```python
graph = builder.compile(
    checkpointer=saver,
    store=store,
)
```

## 9. 向量语义检索

### 9.1 配置 embedding

向量检索需要提供同步 `embed_documents()` 和 `embed_query()` 的 LangChain `Embeddings` 对象：

```python
from langgraph.store.gaussdb import GaussDBIndexConfig, GaussDBStore

index: GaussDBIndexConfig = {
    "dims": 1536,
    "embed": embeddings,
    "fields": ["text", "context[*].content"],
    "distance_type": "cosine",
}

store = GaussDBStore(
    pool,
    schema_name="agent_runtime",
    index=index,
    max_workers=8,
)

store.setup()
```

`embeddings` 由应用提供。`dims` 必须与 embedding 输出维度完全一致。`distance_type` 支持 `cosine` 和 `l2`。

`fields` 可以是字符串或字符串列表。默认值为 `["$"]`，表示索引整个 value。数组路径可以使用 `[*]`，例如 `context[*].content`。

同一 Store schema 的所有写入和清理实例应使用一致的 dims、embedding 模型、fields 和 distance type。

### 9.2 写入向量数据

```python
namespace = ("knowledge", "product-a")

store.put(
    namespace,
    "doc-1",
    {
        "text": "GaussDB stores LangGraph checkpoints",
        "context": [
            {"content": "Supports checkpoint recovery"},
            {"content": "Supports long-term Store memory"},
        ],
        "kind": "documentation",
    },
)
```

单个 item 可以从多个字段生成多条向量。更新 item 时会替换该 item 的向量数据。

可以覆盖单个 item 的索引字段：

```python
store.put(
    namespace,
    "doc-2",
    {
        "title": "Only index this title",
        "body": "not indexed",
    },
    index=["title"],
)
```

也可以保留 JSON item 但关闭其向量索引：

```python
store.put(
    namespace,
    "doc-3",
    {"text": "not semantically searchable"},
    index=False,
)
```

对已有 item 使用 `index=False` 更新时，会删除该 item 原有的向量数据，JSON value 仍然保留。

### 9.3 执行 exact 向量搜索

完成 `store.setup()` 后，无需 ANN 索引即可执行 exact 搜索：

```python
matches = store.search(
    namespace,
    query="How can I resume a graph?",
    filter={"kind": "documentation"},
    limit=5,
    refresh_ttl=False,
)

for match in matches:
    print(match.key, match.score, match.value)
```

有多个向量字段时，一个 item 在结果中最多出现一次。Cosine 模式的分数越接近 1 越相似；L2 模式使用负距离，分数越高越相似。

没有配置 `index` 时，`query` 不会触发 embedding，搜索退化为结构化搜索，`score` 为 `None`。

### 9.4 创建 GsIVFFLAT 索引

GsIVFFLAT 支持最多 1024 维，使用 `lists` 配置索引列表数量：

```python
index_name = store.create_vector_index(
    kind="gsivfflat",
    lists=100,
)

print(index_name)
```

相同定义可以重复调用，返回同一个索引名称。`lists` 应根据数据规模和查询效果通过真实数据测试确定。

### 9.5 创建 GsDiskANN 索引

创建 GsDiskANN 前，先确认建索引会话的 `maintenance_work_mem` 满足目标 GaussDB 版本要求：

```sql
SHOW maintenance_work_mem;
```

需要提高会话值时，使用 DBA 批准的配置创建专用建索引连接。例如：

```python
import os

from psycopg2.extensions import make_dsn

from langgraph.store.gaussdb import GaussDBStore

index_dsn = make_dsn(
    os.environ["GAUSSDB_DSN"],
    options=(
        "-c default_transaction_read_only=off "
        "-c maintenance_work_mem=128MB"
    ),
)

with GaussDBStore.from_conn_string(
    index_dsn,
    schema_name="agent_runtime",
    index=index,
    minconn=1,
    maxconn=2,
) as index_store:
    index_store.setup()

    index_name = index_store.create_vector_index(
        kind="gsdiskann",
        queue_size=64,
        enable_pq=True,
    )

    print(index_name)
```

`128MB` 是配置示例，不是所有版本的通用最低值。数据库返回 “Maintenance_work_mem is below the required value” 时，应由 DBA 按当前版本、维度和数据规模调整。

GsDiskANN 还支持 `pq_nseg` 和 `pq_nclus`。参数范围最终由目标数据库版本校验。

### 9.6 ANN 使用注意事项

`kind="auto"` 当前选择 GsDiskANN。`kind="none"` 不创建或删除索引：

```python
assert store.create_vector_index(kind="none") is None
```

`none` 不会关闭已经启用的 ANN 查询，也不会删除数据库中的索引。删除或重建索引应由明确的数据库运维流程执行。

上线前使用真实数据比较 exact 结果、ANN 召回率、查询耗时和执行计划。

## 10. TTL 数据有效期

### 10.1 配置默认行为

```python
store = GaussDBStore(
    pool,
    schema_name="agent_runtime",
    ttl={
        "default_ttl": 60.0,
        "refresh_on_read": True,
        "omit_expired": True,
        "sweep_interval_minutes": 5,
    },
)

store.setup()
```

所有 TTL 数值单位为分钟。

| 配置 | 说明 |
| --- | --- |
| `default_ttl` | Put 未指定 TTL 时使用的默认分钟数 |
| `refresh_on_read` | Get/Search 返回 item 时是否默认续期 |
| `omit_expired` | 读取时是否隐藏已过期但尚未清理的 item |
| `sweep_interval_minutes` | 后台 sweeper 的默认间隔 |

构造和 `setup()` 不会自动启动 sweeper。

### 10.2 设置 item TTL

```python
# 使用 default_ttl。
store.put(
    ("sessions",),
    "default",
    {"value": 1},
)

# 5 分钟后过期。
store.put(
    ("sessions",),
    "short",
    {"value": 2},
    ttl=5,
)

# 永不过期，不继承 default_ttl。
store.put(
    ("sessions",),
    "permanent",
    {"value": 3},
    ttl=None,
)
```

更新已有 item 时，从本次写入时间重新计算 TTL。`ttl=None` 会清除旧的过期时间。

即使构造 Store 时没有提供 `ttl` 配置，也可以在单次 `put()` 中传入 TTL，并手工调用 `sweep_ttl()`。

### 10.3 控制读取续期

```python
item = store.get(
    ("sessions",),
    "short",
    refresh_ttl=False,
)

results = store.search(
    ("sessions",),
    filter={"kind": "active"},
    refresh_ttl=True,
)
```

`refresh_ttl=None` 使用 Store 默认配置，显式 `True` 或 `False` 覆盖本次调用。

`omit_expired=True` 时，Get、Search 和 ListNamespaces 都不会返回过期 item。`omit_expired=False` 时，尚未物理清理的过期 item 仍可能被读取。

### 10.4 手工清理

```python
deleted_count = store.sweep_ttl()
print("deleted:", deleted_count)
```

返回值是本次删除的 item 数。重复清理且没有新过期数据时返回 0。

如果 Store 启用了向量，同一个 schema 的 TTL 清理任务也应使用相同的向量配置。

### 10.5 周期清理

```python
handle = store.start_ttl_sweeper()

stopped = store.stop_ttl_sweeper(timeout=10)
if not stopped:
    raise RuntimeError("TTL sweeper is still running")
```

同一个 Store 重复调用 `start_ttl_sweeper()` 会返回同一个正在运行的 handle。sweeper 启动后先等待一个完整间隔，再执行第一次清理。需要立即清理时，先调用 `sweep_ttl()`。

多实例部署应明确由哪个进程或外部调度器负责清理，避免不必要的重复扫描。

## 11. 异步使用

Saver 和 Store 的异步方法使用同一组类，不需要 Async 专用类型或异步数据库 pool。

### 11.1 异步 Checkpointer

下面假定应用已经按第 6 章创建了 `builder` 和 `input_value`：

```python
import os

from langgraph.checkpoint.gaussdb import GaussDBSaver

async with GaussDBSaver.from_conn_string(
    os.environ["GAUSSDB_DSN"],
    schema_name="agent_runtime",
    minconn=1,
    maxconn=8,
    max_workers=8,
) as saver:
    await saver.asetup()

    graph = builder.compile(checkpointer=saver)

    result = await graph.ainvoke(
        input_value,
        {"configurable": {"thread_id": "thread-1"}},
        durability="sync",
    )
```

### 11.2 异步 Store

```python
import os

from langgraph.store.gaussdb import GaussDBStore

namespace = ("users", "u-42")

async with GaussDBStore.from_conn_string(
    os.environ["GAUSSDB_DSN"],
    schema_name="agent_runtime",
    minconn=1,
    maxconn=8,
    max_workers=8,
) as store:
    await store.asetup()

    await store.aput(
        namespace,
        "profile",
        {"name": "Alice"},
    )

    item = await store.aget(
        namespace,
        "profile",
        refresh_ttl=False,
    )

    matches = await store.asearch(
        namespace,
        limit=10,
        refresh_ttl=False,
    )

    namespaces = await store.alist_namespaces(
        prefix=("users",),
    )

    deleted_count = await store.asweep_ttl()
```

### 11.3 取消行为

取消一个异步 await 不保证数据库操作回滚。已经开始的写操作仍可能完成并提交。

收到 `CancelledError` 后：

1. 使用相同 namespace/key 或 thread/checkpoint identity 查询最终状态。
2. 只有在确认结果后再决定是否重试。
3. 非幂等业务操作不要直接更换新 id 重放。
4. 需要限制数据库执行时间时配置 statement timeout。

`alist()` 会先完整取得匹配历史再逐项 yield。历史量较大时始终设置合理的 `limit`。

## 12. 关闭和资源管理

### 12.1 使用 context manager

内部 pool 场景优先使用 `with` 或 `async with`：

```python
with GaussDBStore.from_conn_string(
    dsn,
    schema_name="agent_runtime",
) as store:
    store.setup()
    # 使用 Store。
```

退出 context 后不要继续使用该对象或依赖它的 graph。

### 12.2 显式关闭

```python
store.close(timeout=30)
saver.close(timeout=30)
```

异步版本：

```python
await store.aclose(timeout=30)
await saver.aclose(timeout=30)
```

关闭会停止接收新工作，并等待已经接受的操作结束。timeout 不会强制终止数据库线程；超时后可以在阻塞原因解除后再次调用 `close()`。

Store 会在关闭前停止自己的 TTL sweeper。

### 12.3 外部资源关闭顺序

共享外部资源时按以下顺序关闭：

1. 停止新的应用请求。
2. 关闭 Store。
3. 关闭 Saver。
4. 关闭外部 executor。
5. 最后关闭外部 pool。

外部 `ThreadPoolExecutor` 示例：

```python
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=8)

saver = GaussDBSaver(
    pool,
    schema_name="agent_runtime",
    executor=executor,
)

try:
    saver.setup()
finally:
    saver.close(timeout=30)
    executor.shutdown(wait=True)
    pool.closeall()
```

只使用 `ThreadPoolExecutor`，不要使用 `ProcessPoolExecutor`。

## 13. Serializer 和安全

### 13.1 自定义 serializer

`GaussDBSaver` 的 `serde` 参数接受 LangGraph `SerializerProtocol`：

```python
saver = GaussDBSaver(
    pool,
    schema_name="agent_runtime",
    serde=serializer,
)
```

共享同一 checkpoint schema 的所有读写实例必须使用彼此兼容的 serializer 和密钥配置。滚动升级前确保新旧实例都能读取数据库中已有 payload。

扩展 msgpack allowlist 时可以创建派生 Saver：

```python
derived = saver.with_allowlist(
    {
        ("my_package", "MyType"),
    }
)
```

派生对象与原 Saver 共享生命周期，由原 Saver 统一关闭资源。

### 13.2 安全要求

生产环境至少应满足以下要求：

- DSN 和 serializer key 由 Secret Manager 或部署平台管理。
- 连接启用符合组织要求的 TLS。
- 数据库账号使用最小权限。
- 日志不记录密码、完整 DSN、敏感 payload 或 embedding 原文。
- embedding 使用外部服务时，确认数据驻留、传输和保留策略。
- 数据库备份、静态加密和密钥轮换有明确流程。

Serializer 加密不等于整个数据库记录都被加密。Store JSON、业务标识符、时间戳和其他可查询字段需要依赖数据库和部署层保护。

## 14. 常见问题

### 14.1 `setup()` 报 schema 不存在

确认：

- `schema_name` 已由 DBA 创建。
- setup 账号具备目标 schema 的 DDL 权限。
- 所有 pool connection 的 search path 一致。
- 目标 schema 中没有不兼容的同名对象。

不要绕过 setup 继续运行。

### 14.2 写操作报告只读 transaction

在 DSN 或 kwargs 中设置：

```text
options='-c default_transaction_read_only=off'
```

仍然失败时检查节点是否物理只读，以及账号是否具备所需权限。

### 14.3 外部 connection 被拒绝

外部 psycopg2 connection 必须满足：

- `autocommit=False`。
- 当前 transaction 已结束并处于 idle。
- 调用方没有未提交或未回滚的操作。

在交给 Saver/Store 前显式执行 `commit()` 或 `rollback()`。

### 14.4 向量 setup 失败

依次检查：

1. `enable_vectordb` 是否开启。
2. `floatvector` 类型是否存在。
3. `dims` 是否与 embedding 输出一致。
4. 已有向量数据对象是否与当前 dims 一致。
5. embedding 输出是否包含 NaN、Infinity 或错误维度。

### 14.5 GsDiskANN 创建失败

先查看：

```sql
SHOW maintenance_work_mem;
```

错误包含 “Maintenance_work_mem is below the required value” 时，使用 DBA 批准的建索引会话配置后重试。

还应检查 `queue_size`、`pq_nseg`、`pq_nclus` 和 `enable_pq` 是否被当前 GaussDB 版本接受。GsIVFFLAT 的维度不能超过 1024。

### 14.6 向量搜索的 `score` 为 `None`

只有同时满足以下条件才会产生向量分数：

- Store 构造时提供 `index`。
- query 是非空字符串。
- `limit` 大于 0。

ANN 索引不是获得 score 的必要条件；完成 vector setup 后 exact 搜索即可返回 score。

### 14.7 过期数据仍然可见

检查 `omit_expired`：

- `True`：读取时隐藏过期数据。
- `False`：过期但尚未 sweep 的数据仍可能读取。

物理删除需要手工 `sweep_ttl()`、后台 sweeper 或外部调度任务。

### 14.8 异步任务取消后数据仍被写入

这是当前异步接口的可观察行为。取消只停止等待方，已经开始的数据库操作仍可能提交。使用相同业务 key 读回确认。

### 14.9 `close()` 超时

常见原因包括：

- embedding 调用仍在运行。
- worker 正等待数据库连接。
- SQL 被锁或 statement timeout 尚未触发。
- TTL sweeper 尚未退出。
- 外部 pool 被提前关闭。

解除阻塞后再次调用 `close()`。

### 14.10 错误包含 `outcome_unknown=True`

这表示客户端在 commit 期间失去连接，无法确认数据库最终提交还是回滚。

Store 操作使用相同 namespace/key 查询，Saver 操作使用相同 thread/checkpoint identity 查询。确认最终状态前不要盲目重试非幂等操作。

## 15. 上线检查清单

### 15.1 安装与数据库

- Python 和软件包版本符合要求。
- 目标环境的 `sql_compatibility` 为 `A` 或 `ORA`。
- Saver 和 Store 已分别执行 `setup()`。
- schema 和权限由部署流程管理。
- 业务连接允许写 transaction。

### 15.2 功能

- 相同 thread id 能够恢复图状态。
- Store namespace 满足用户或租户隔离要求。
- JSON filter 使用真实业务数据验证。
- 使用向量时已验证维度、分数和召回结果。
- 使用 ANN 时已比较 exact 基准和执行计划。
- TTL 隐藏、续期和清理行为符合数据保留要求。

### 15.3 资源

- pool `maxconn` 符合数据库连接配额。
- `max_workers` 已经过并发测试。
- 外部 pool 和 executor 有唯一创建者。
- 关闭顺序和 timeout 已纳入应用停机流程。
- 多实例 TTL 清理责任明确。

### 15.4 安全

- 凭据和 serializer key 不在源码中。
- TLS、最小权限、备份和静态加密符合组织要求。
- 日志和监控不包含敏感数据。
- embedding 外部服务通过安全评审。

## 16. 公共 API 总表

### 16.1 `GaussDBSaver`

| 方法 | 返回值 | 说明 |
| --- | --- | --- |
| `from_conn_string()` | `GaussDBSaver` | 创建拥有内部 pool 的 Saver |
| `setup()/asetup()` | `None` | 初始化和检查数据库对象 |
| `put()/aput()` | config | 保存 checkpoint；通常由 LangGraph 调用 |
| `put_writes()/aput_writes()` | `None` | 保存 pending writes；通常由 LangGraph 调用 |
| `get()/aget()` | checkpoint 或 `None` | 读取 checkpoint payload |
| `get_tuple()/aget_tuple()` | `CheckpointTuple \| None` | 读取完整 checkpoint |
| `list()/alist()` | iterator | 列出历史 |
| `delete_thread()/adelete_thread()` | `None` | 删除 thread |
| `get_delta_channel_history()/aget_delta_channel_history()` | mapping | 读取 DeltaChannel 历史 |
| `get_next_version()` | `str` | 生成下一 channel version |
| `with_allowlist()` | Saver | 创建扩展 serializer allowlist 的派生 Saver |
| `close()/aclose()` | `None` | 关闭资源 |

### 16.2 `GaussDBStore`

| 方法 | 返回值 | 说明 |
| --- | --- | --- |
| `from_conn_string()` | `GaussDBStore` | 创建拥有内部 pool 的 Store |
| `setup()/asetup()` | `None` | 初始化和检查数据库对象 |
| `get()/aget()` | `Item \| None` | 精确读取 |
| `put()/aput()` | `None` | 写入或替换 |
| `delete()/adelete()` | `None` | 删除 |
| `search()/asearch()` | `list[SearchItem]` | 结构化或向量搜索 |
| `list_namespaces()/alist_namespaces()` | `list[tuple[str, ...]]` | 列出 namespace |
| `batch()/abatch()` | `list` | 批量操作 |
| `create_vector_index()/acreate_vector_index()` | `str \| None` | 创建或检查 ANN 索引 |
| `sweep_ttl()/asweep_ttl()` | `int` | 清理过期 item |
| `start_ttl_sweeper()/astart_ttl_sweeper()` | handle | 启动周期清理 |
| `stop_ttl_sweeper()/astop_ttl_sweeper()` | `bool` | 停止周期清理 |
| `close()/aclose()` | `None` | 关闭资源 |
