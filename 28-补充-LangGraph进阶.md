# 补充篇 28：LangGraph 进阶（Send 扇出 / 重试 / 缓存 / 持久化等级 / 图可视化）

> 定位：补上阶段 06 到阶段 09 之间的断层——从"会画图"到"敢上生产" ｜ 难度 ★★★ ｜ 预计 6 小时
> **建议学习时机**：学完阶段 06（StateGraph 基础）后读；若已学阶段 09（多代理）更佳，本课解释了"子图/Supervisor 底下那些机制"。

> **【一句话记住】**：LangGraph 的进阶能力几乎都挂在**两个地方**——`add_node()` 的参数（重试/缓存/超时）和 `compile()/stream()` 的参数（缓存后端/持久化等级）。
>
> **【生活类比】**：阶段 06 你学会了搭一条"固定的流水线"；这一篇教的是给流水线装**智能装置**：`Send` 是"来料一件就临时开一条支线"（不是固定几条），`RetryPolicy` 是"这一道工序坏了自动重做"，`CachePolicy` 是"同样的料做过就贴个已完成标签"，`durability` 是"多久存一次盘"，`draw_mermaid` 是"把车间图纸打印出来"。

## 1. 学习目标

- [ ] 用 `Send` 实现**动态并行扇出**（map-reduce），并说清为什么给收集字段配 reducer 是必须的
- [ ] 给单个节点配 `RetryPolicy`（重试次数 / 退避 / 只重试哪些异常）
- [ ] 给单个节点配 `CachePolicy` + 在 `compile(cache=...)` 挂缓存后端
- [ ] 说清 `durability` 三档（`exit` / `async` / `sync`）的取舍
- [ ] 用 `Command` 在节点里**同时**改状态和决定下一跳
- [ ] 用 `draw_mermaid()` 把图导成可读的流程图（排障与写文档）

## 2. 前置知识

- 阶段 06（StateGraph / 条件边 / Checkpointer）——本课全部建立在这上面
- 阶段 12（短期记忆与 Store）——理解"状态"与"持久化"的区别
- 补充篇 24（异步与并发）——`Send` 扇出本质是并发，值得先懂并发

## 3. 核心概念

### 3.1 一张表看清"五个旋钮"

| 旋钮 | 挂在哪 | 解决什么 | 一句话类比 |
|------|--------|---------|-----------|
| `Send` | 条件边的返回值 | **动态**生成 N 条并行分支（N 运行时才知道） | 来料几件就开几条支线 |
| `retry_policy` | `add_node(...)` | 节点因**瞬时故障**（网络/429/5xx）失败后自动重做 | 这道工序坏了自动重做 |
| `cache_policy` | `add_node(...)` + `compile(cache=...)` | **相同输入**跳过重复计算（省时间与钱） | 同料同工，贴"已完成"标签 |
| `durability` | `stream()` / `invoke()` 的调用参数 | 多久把检查点写一次盘（性能 vs 可恢复性） | 多久存一次盘 |
| `Command` | 节点**返回值** | 一次返回里同时"更新状态 + 指定下一跳" | 边干边决定下一步去哪 |

> **记忆钩子**：**两个地方挂旋钮**——节点本身的能力挂在 `add_node()`，跨节点的能力挂在 `compile()` / 调用参数。搞混了就会"参数写了却不生效"（比如只写了 `cache_policy` 却没在 `compile()` 传 `cache`）。

### 3.2 `Send`：静态边 vs 动态扇出

阶段 06 的条件边只能"从固定几条路里选一条"；但很多任务的**分支数在运行时才知道**——比如把一篇长文按主题切成 7 段分别摘要、把 50 个文件分别处理。

`Send` 让路由函数**返回一个列表**，LangGraph 按列表长度临时开出对应数量的并行分支：

```python
from langgraph.types import Send

def fan_out(state) -> list[Send]:
    return [Send("summarize", {"topic": t}) for t in state["topics"]]
```

**关键约束**：并行分支的写回会**落在同一个 super-step**，所以承载结果的字段必须配 reducer（把多个分支的结果合并进同一字段的规则），否则报 `InvalidUpdateError`：

```python
summaries: Annotated[list, operator.add]      # 没有 reducer → 并行写回直接报错
```

> **记忆钩子**：并行分支像**多个收银员同时往同一个钱箱放钱**——不规定"放钱"这个动作是"叠加"（`operator.add`）还是"覆盖"，系统只能报错拒绝。reducer 就是那条"叠加"的规矩。

### 3.3 重试：`RetryPolicy`

```python
from langgraph.types import RetryPolicy

builder.add_node("call_api", call_api, retry_policy=RetryPolicy(
    max_attempts=3,          # 最多尝试 3 次（含首次）
    initial_interval=0.5,    # 首次退避 0.5s
    backoff_factor=2.0,      # 每次翻倍：0.5 → 1 → 2
    jitter=True,             # 加随机抖动，避免"重试风暴"同时打满上游
    retry_on=TimeoutError,   # 只重试这类异常；不传则用默认规则
))
```

三个必须记住的点：

1. **默认不会重试"你自己的逻辑错误"**：默认策略对 `ValueError` / `TypeError` / `ImportError` 这类**代码 bug** 是不重试的——因为它们重试一万次也是错。默认只兜"大概率是瞬时"的异常。
2. **重试是"整个节点从头再跑一次"**，所以节点必须**幂等**（同一操作执行多次，结果与执行一次相同）：如果节点里有写库、扣款，要么带幂等键，要么把副作用移到节点之后。
3. **`timeout` 只对 async 节点生效**：同步节点是阻塞调用，框架没法从外部打断它，写了 `timeout` 也不会按预期超时。要给同步节点限时间，只能自己在节点内 `asyncio.wait_for`（本来就是 async）或用外部超时机制包住。

> **记忆钩子**：重试像"重新洗牌重发一遍牌"——不是从断点续，是**从头再来**。所以节点里不能有"已经发出去的不可撤销动作"。

### 3.4 缓存：`CachePolicy`

```python
from langgraph.types import CachePolicy
from langgraph.cache.memory import InMemoryCache

builder.add_node("embed_docs", embed_docs, cache_policy=CachePolicy(ttl=120))   # 120 秒内同输入直接命中
graph = builder.compile(cache=InMemoryCache())                                  # ← 别忘这一行
```

- **缓存的键**默认由节点输入生成；命中就**完全跳过节点执行**，直接返回上次结果。
- `ttl` 是"存活秒数"，不设则永不过期。
- 后端：`InMemoryCache`（单进程）、`SqliteCache`（本地持久）、生产可用 Redis 后端；**多副本部署必须用共享后端**，否则缓存各存各的等于没有。

> **记忆钩子**：`cache_policy` 与 `compile(cache=...)` 是"**锁 + 钥匙**"——只配了锁没配钥匙，门根本不会锁。

### 3.5 持久化等级：`durability`

Checkpointer 决定了"能恢复"，`durability` 决定"多快落盘"：

| 取值 | 行为 | 适合 |
|------|------|------|
| `"sync"` | 每个 super-step 结束**立刻同步落盘**才继续 | 关键业务、要求"崩了也不丢一步" |
| `"async"` | 落盘在后台异步进行（**多数版本的默认**，以你安装的版本为准） | 大多数生产场景：性能与安全折中 |
| `"exit"` | 只在**图执行结束时**落盘 | 一次性批处理、不在乎中途崩溃 |

```python
graph.stream(inputs, cfg, durability="sync")
```

> **两个前提**：① **只有配了 checkpointer 时 `durability` 才有意义**——没有检查点，压根没有"落盘"这件事；② 默认值在不同版本间**变过**，所以别依赖"不写就是 async"——**关键业务请显式写 `durability="sync"`**，让意图写在代码里而不是靠默认值。

`"sync"` 最安全但最慢，`"exit"` 最快但中途崩溃就等于全丢。给"扣款、审批这类关键步骤"配 `sync`，给"跑一遍出报告"的批处理配 `exit`。

### 3.6 `Command`：状态更新 + 路由一体

普通节点只能"改状态"，下一跳由边决定。`Command` 让你**一次返回里把两件事都做了**——这正是多代理"交接（handoff）"和 HITL 恢复的标准写法：

```python
from langgraph.types import Command

def triage(state) -> Command:
    if "退款" in state["messages"][-1].content:
        return Command(goto="refund_agent", update={"route": "refund"})
    return Command(goto="faq_agent", update={"route": "faq"})
```

> **记忆钩子**：`Command` = "**改完状态顺便说一句'去那儿'**"，比"改状态 + 条件边再算一次"少绕一圈。

### 3.7 图可视化：排障与写文档的刚需

图一旦超过 8 个节点，靠脑子记连接关系必然出错。**先把它打出来**：

```python
print(graph.get_graph().draw_mermaid())          # 生成 Mermaid 文本，贴到任何支持 Mermaid 的地方
# graph.get_graph().draw_mermaid_png()           # 直接出 PNG（需额外依赖，可指定输出文件）
```

排障时的用法：把这张图和实际 trace（阶段 08）对照，立刻能看出"模型走的是哪条边、有没有进不该进的循环"。

## 4. 动手教程

### 步骤 1：`Send` 扇出 + reducer 合并（map-reduce）

*（`init_chat_model` 见阶段 01；`model` 为主模型，`Topics` 为输入主题列表）*

```python
import operator
from typing import Annotated, Sequence, TypedDict

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send

model = init_chat_model("openai:gpt-4o-mini", temperature=0)


class State(TypedDict):
    topics: list[str]
    summaries: Annotated[list, operator.add]      # ← 收集字段必须配 reducer


def summarize(topic_state: dict) -> dict:
    """被 Send 并行调用的 worker：只拿到 Send 传进来的那一小份状态。"""
    topic = topic_state["topic"]
    text = model.invoke(f"用一句话解释：{topic}").content
    return {"summaries": [f"{topic}：{text}"]}     # 返回列表，交给 reducer 追加


def fan_out(state: State) -> Sequence[Send]:
    return [Send("summarize", {"topic": t}) for t in state["topics"]]   # 每个主题一条支线


def merge(state: State) -> dict:
    report = "\n".join(sorted(state["summaries"]))
    return {"summaries": [f"=== 汇总 ===\n{report}"]}


builder = StateGraph(State)
builder.add_node("summarize", summarize)
builder.add_node("merge", merge)
builder.add_conditional_edges(START, fan_out, ["summarize"])   # 入口处动态扇出
builder.add_edge("summarize", "merge")
builder.add_edge("merge", END)
graph = builder.compile()

print(graph.invoke({"topics": ["RAG", "LCEL", "Checkpointer", "MCP"], "summaries": []})["summaries"][-1])
```

要点：**worker 收到的是 `Send` 里那一小份 dict**（不是完整 State），但它**写回的是主 State 的字段**（靠 reducer 合并）。这是 `Send` 最反直觉的一点。

### 步骤 2：给易抖动的节点配重试

```python
from langgraph.types import RetryPolicy

builder.add_node(
    "fetch_docs",
    fetch_docs,
    retry_policy=RetryPolicy(
        max_attempts=4,
        initial_interval=0.5,
        backoff_factor=2.0,
        jitter=True,
        retry_on=TimeoutError,        # 只重试超时；业务异常立刻上抛
    ),
)
```

要点：**只重试"值得重试的"**。把 `retry_on` 收窄，比无脑 `max_attempts=10` 更专业——重试不该掩盖 bug。

### 步骤 3：给昂贵且幂等的节点配缓存

```python
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy

builder.add_node("embed_chunks", embed_chunks, cache_policy=CachePolicy(ttl=300))
graph = builder.compile(cache=InMemoryCache())      # ← 必须
```

要点：适合缓存的节点 = **纯函数式、贵、输入稳定**（embedding、打分、检索）。含时间戳/随机数/当前用户的输入会让缓存**永远不命中**，这一点与阶段 19 的 LLM 缓存是同一个坑。

### 步骤 4：按业务重要性选 durability

```python
cfg = {"configurable": {"thread_id": "order-42"}}
graph.stream({"messages": [("user", "帮我退款")]}, cfg, durability="sync")     # 关键流程：步步落盘
graph.stream({"messages": [("user", "生成本月报表")]}, cfg, durability="exit")  # 批处理：跑完再存
```

### 步骤 5：用 `Command` 做交接

```python
from langgraph.types import Command

def triage(state) -> Command:
    text = state["messages"][-1].content
    if "退款" in text:
        return Command(goto="refund_agent", update={"route": "refund"})
    return Command(goto="faq_agent", update={"route": "faq"})

builder.add_node("triage", triage)
# 无需再写 add_conditional_edges：goto 就是这一跳的路由
```

### 步骤 6：把图打印出来

```python
print(graph.get_graph().draw_mermaid())
# -> 一段 flowchart 文本；贴进 README 或支持 Mermaid 的编辑器即可看到节点与边
```

### 步骤 7：节点超时与"优雅停机"（1.2 起）

长跑节点最怕两件事：**卡死**和**部署时被硬杀**。较新版本给了三个对应开关：

```python
builder.add_node("long_job", long_job, timeout=120)   # 超时上限，避免单个节点挂死整个图
builder.add_node("risky", risky, error_handler=on_error)   # 重试耗尽后的补偿/回滚钩子
# 运行侧：RunControl.request_drain() 请求"跑完当前安全点就停"，便于滚动发布
```

> 如果你用的版本还没有 `timeout` / `error_handler`，别慌——用外层 `asyncio.wait_for` 或在节点内部自己兜也是等效做法，本课的重点是**知道生产上必须有人管这两件事**。

## 5. 完整示例

一个"多主题并行摘要 → 汇总"的完整图，把扇出、reducer、重试、缓存、可视化串起来：

```python
# fanout_summary.py
import operator
from typing import Annotated, Sequence, TypedDict

from langchain.chat_models import init_chat_model
from langgraph.cache.memory import InMemoryCache
from langgraph.graph import END, START, StateGraph
from langgraph.types import CachePolicy, RetryPolicy, Send

model = init_chat_model("openai:gpt-4o-mini", temperature=0)


class State(TypedDict):
    topics: list[str]
    summaries: Annotated[list, operator.add]
    final: str


def summarize(part: dict) -> dict:
    topic = part["topic"]
    text = model.invoke(f"用一句话解释：{topic}").content
    return {"summaries": [f"- {topic}：{text}"]}


def fan_out(state: State) -> Sequence[Send]:
    return [Send("summarize", {"topic": t}) for t in state["topics"]]


def merge(state: State) -> dict:
    return {"final": "知识卡：\n" + "\n".join(sorted(state["summaries"]))}


builder = StateGraph(State)
builder.add_node("summarize", summarize, retry_policy=RetryPolicy(max_attempts=3, initial_interval=0.5))
builder.add_node("merge", merge, cache_policy=CachePolicy(ttl=60))
builder.add_conditional_edges(START, fan_out, ["summarize"])
builder.add_edge("summarize", "merge")
builder.add_edge("merge", END)

graph = builder.compile(cache=InMemoryCache())

if __name__ == "__main__":
    print(graph.get_graph().draw_mermaid())          # 先看图，再跑
    out = graph.invoke({"topics": ["RAG", "LCEL", "Checkpointer", "MCP"], "summaries": [], "final": ""})
    print(out["final"])
```

再跑一次同样的输入，观察 `merge` 节点被缓存跳过（第二次更快）——这就是 `cache_policy` 的直观效果。

## 6. 练习

**基础**：把上面的 `fan_out` 改成从命令行/文件读主题列表，送给 `graph.invoke`，验证支线数量随输入动态变化。

**进阶**：给 `summarize` 加 `RetryPolicy`，再用一个"前两次必抛 `TimeoutError`、第三次成功"的假函数替换模型调用，验证它最终成功（顺便体会 `jitter` 与 `backoff_factor`）。

**挑战**：把 `durability` 分别设成 `async` 和 `exit`，在节点中 `raise` 一个异常模拟中途崩溃，用 `graph.get_state(cfg)` 对比"检查点是否已存下刚才的进度"，并解释差异。

<details>
<summary>参考答案要点</summary>

- 基础：把 `topics` 做成外部输入即可——`Send` 的条数 = `len(state["topics"])`，图结构**不用改**。这正是 `Send` 相对"写死 N 条边"的价值：**图的形状由数据决定，而不是由代码决定**。
- 进阶：`retry_on=TimeoutError` + `max_attempts=3`，用闭包计数器控制"第 3 次才成功"。观察点：日志里会有 2 次重试；`jitter=True` 让退避时间随机化，避免多个并发支线同时重试造成"重试风暴"。要点：**重试必须配幂等**，否则"已经写进去的副作用"会被写两遍。
- 挑战：`durability="async"` 时崩溃点之后的部分可能还没落盘，`get_state` 可能只见较早的检查点；`"sync"` / `"exit"` 的行为差异同理——`"exit"` 在中途崩溃时几乎什么都存不下。要点：**durability 是"性能 vs 能恢复到哪一步"的旋钮**，关键业务选 `sync`。
</details>

## 7. 自测清单

- [ ] 能说出 `Send` 与普通条件边的区别（动态条数 vs 固定选项）
- [ ] 知道并行扇出的收集字段必须配 reducer，并能说出不配会怎样
- [ ] 会配 `RetryPolicy`，并知道它重试的是**整个节点**、节点必须幂等
- [ ] 会配 `CachePolicy`，且知道**必须同时在 `compile(cache=...)` 挂后端**
- [ ] 能说清 `durability` 三档的取舍
- [ ] 会用 `Command(goto=..., update=...)` 做交接
- [ ] 会用 `draw_mermaid()` 把图导出来排障

## 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| `InvalidUpdateError` | 并行扇出的收集字段没有 reducer | 给该字段加 `Annotated[list, operator.add]` |
| `cache_policy` 写了没生效 | 只在 `add_node` 配了，`compile()` 没传 `cache` | `compile(cache=InMemoryCache())` |
| 缓存永不命中 | 输入含时间戳/随机数/内存地址 | 缓存键只保留语义上稳定的字段 |
| 重试把 bug 掩盖了 | 用默认策略或 `retry_on` 放太宽 | `retry_on` 收窄到瞬时异常（超时/连接/5xx） |
| 重试后数据写了两遍 | 节点不幂等 | 带幂等键，或把副作用移出节点 |
| 多副本下缓存行为不一致 | 用了 `InMemoryCache` | 生产换共享后端（如 Redis） |
| `Send` 的 worker 里拿不到全局状态 | worker 只收到 `Send` 传的那份 dict | 需要的信息要在 `Send` 的第二参里显式带上 |
| 图改乱了不自知 | 靠脑子记边 | 改完 `draw_mermaid()` 看一眼 |

## 9. 延伸

- 官方文档：Graph API 的 `Send` / `RetryPolicy` / `CachePolicy` / `durability` 章节
- 阶段 09 多代理：Supervisor / 子图内部大量使用 `Command` 与扇出
- 阶段 13 HITL：`interrupt` 恢复时用 `Command(resume=...)`，与本课的 `Command(goto=...)` 是同一个类的两种用法
- 补充篇 26 生产监控：`durability` 与检查点策略直接影响"崩溃后能恢复到哪一步"这一线上指标

## 10. 记忆强化

**口诀**：

> 旋钮分两处：能力挂 add_node，后端挂 compile；
> Send 动态开支线，收集字段配 reducer；
> 重试重跑整节点，节点不幂等会写两遍；
> 缓存锁加钥匙，durability 选落盘节奏；
> 改完图先 draw_mermaid，看一眼再上线。

**5 分钟回顾闪卡**（先默答，再展开核对）：

1. 问：`Send` 和普通条件边有什么本质区别？
   <details><summary>点击看答案</summary>条件边是"从**固定几条**出路里选一条"；`Send` 是路由函数**返回一个列表**，在运行时**动态生成 N 条并行支线**（N 由数据决定，图结构不用改）。典型场景：把长文按主题切几段就开几条支线、把文件列表逐个处理。</details>
2. 问：为什么并行扇出的收集字段一定要配 reducer？
   <details><summary>点击看答案</summary>多个并行分支在**同一个 super-step** 写回同一字段，谁覆盖谁没有确定答案。配 `Annotated[list, operator.add]` 就是告诉 LangGraph"这个字段是叠加的"，不配则报 `InvalidUpdateError`。</details>
3. 问：`RetryPolicy` 默认会重试哪些异常？为什么故意不重试某些异常？
   <details><summary>点击看答案</summary>默认重试"大概率是瞬时"的异常（网络/超时/5xx 类），而**不重试** `ValueError` / `TypeError` / `ImportError` 这类**代码 bug**——因为重试一万次也还是错，只会浪费时间并掩盖问题。可用 `retry_on` 精确指定。</details>
4. 问：配了 `cache_policy` 但缓存不生效，最可能漏了什么？
   <details><summary>点击看答案</summary>漏了 `compile(cache=...)`。`cache_policy` 是"锁"，`compile` 时传的缓存后端是"钥匙"，两者缺一不可。另外多副本部署要用共享后端（如 Redis），否则各进程各存一份。</details>
5. 问：`durability` 的 `sync` / `async` / `exit` 分别是什么行为？
   <details><summary>点击看答案</summary>`sync`=每个 super-step 同步落盘才继续（最安全最慢）；`async`=后台异步落盘（默认，折中）；`exit`=只在图执行结束时落盘（最快，中途崩溃全丢）。关键业务选 `sync`，一次性批处理可选 `exit`。</details>
6. 问：`Command(goto=..., update=...)` 一次做了哪两件事？用在哪？
   <details><summary>点击看答案</summary>同时"**更新状态**"和"**指定下一跳**"，省掉"改状态 + 条件边再算一次"的一绕。典型用于多代理的交接（handoff）和 HITL 的 `Command(resume=...)` 恢复。</details>

**费曼任务**：

- 用大白话向一位"完全不懂编程的朋友"讲清【为什么"运行时才知道要开几条支线"这件事，用固定几条边画不出来】，限时 2 分钟（建议录音/对着镜子讲）。哪里卡住、哪里要回头翻书，那个点就是你还没真正懂的点——回到对应小节重看后再讲一遍。

---

上一阶段：[06 LangGraph 基础](06-阶段06-LangGraph基础.md) ｜ 返回[总览与学习路线](00-总览与学习路线.md) ｜ 下一阶段：[09 多代理架构](09-阶段09-多代理架构.md) ｜ 补充篇导航：[29 语音与实时 Agent](29-前瞻-语音与实时Agent.md)
