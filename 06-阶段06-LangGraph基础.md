# 阶段 06：LangGraph 基础（StateGraph / 条件边 / Checkpointer）

> 定位：从"管道"升级到"有状态、可循环、可持久化"的编排 ｜ 难度 ★★★ ｜ 预计 6 小时

> **【一句话记住】**：节点只交"补丁"，reducer 决定追加还是覆盖，thread_id 分出不同会话。
>
> **【生活类比】**：LangGraph 像一个工厂车间——State 是车间中央那块白板，每个节点（工人）只在白板上贴一张"我改了哪几格"的便签（返回补丁），而不是擦掉重写整面墙。`add_messages` 这个 reducer 规定"消息便签要往上贴、不许盖住旧的"；Checkpointer 是下班时给白板拍快照，第二天凭 thread_id（哪个工位哪班）把快照还原。

## 1. 学习目标

- [ ] 说清 Node、Edge、Conditional Edge 三者的关系与写法
- [ ] 理解"节点返回的是状态补丁（patch），不是新状态"
- [ ] 用 `StateGraph` 搭出带条件分支的最小图并跑通
- [ ] 用 Checkpointer + `thread_id` 实现多轮会话持久化，并解释 `thread_id` 为什么是会话唯一标识

## 2. 前置知识

- 阶段 01–05（尤其 LCEL）
- 图的基本概念（节点、边）

## 3. 核心概念

### 3.1 三要素

```
          ┌───────────────┐
  START ─►│  Node A       │   函数签名: (state) -> dict（返回的是"状态补丁"）
          └───────┬───────┘
                  │  Conditional Edge：add_conditional_edges("A", route_fn, {...})
          ┌───────┴───────┐
     even ▼               ▼ odd
   ┌────────────┐   ┌────────────┐
   │ B_even     │   │ B_odd      │
   └─────┬──────┘   └─────┬──────┘
         └───────┬────────┘  Edge（静态边）
                ▼  END
```

| 概念 | 定义 | 关键点 |
|------|------|--------|
| **Node** | `(state) -> dict` 的普通函数 | 返回值是**补丁**，按 State 的 reducer 合并 |
| **Edge** | `add_edge("A","B")` | 无条件跳转 |
| **Conditional Edge** | `add_conditional_edges("A", route_fn, mapping)` | `route_fn` 只读 state，返回分支名 |
| **State** | 全局共享的数据容器 | 节点之间唯一的通信介质 |

> **记忆钩子**：节点返回的不是"新状态"而是"补丁"——像你在共享文档里只写"把第 3 行改成 42"，而不是把整篇文档重抄一遍。你没提到的字段，原样保留。

### 3.2 Reducer（为什么 messages 不会被覆盖）

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]   # 追加而非覆盖
```

`Annotated[类型, reducer函数]` 决定"新值如何与旧值合并"。用普通 `list` 且直接返回新列表会**覆盖**历史——这是最常见的 bug。

> **记忆钩子**：不加 `add_messages`，每轮对话就像拿橡皮擦把黑板上之前的话全擦掉再写新的——模型自然失忆。加了 `Annotated[list, add_messages]`，新消息是"往上贴"，旧消息一直都在。

### 3.3 Checkpointer 与 thread_id

- **Checkpointer** 在每个 super-step（每个节点执行完）把 State 快照（一份完整的状态拷贝）写入存储，下次调用前自动读回。没有它，图是无状态的。
- **`thread_id` 是对话的唯一标识**：相同 `thread_id` 的多次 `invoke` 共享并累积同一份 state；不同 `thread_id` 完全隔离。也正因如此，一个编译好的图能并发服务多个会话。

| 后端 | 适用 |
|------|------|
| `InMemorySaver` | 开发/测试，进程重启即丢 |
| `SqliteSaver` | 本地开发，文件持久 |
| `PostgresSaver` | 生产 |

> **记忆钩子**：`thread_id` 就像医院的病历号——同一个号下，下次来看病医生能翻出你上次的记录；换个号就是另一个人的全新病历。两个用户串台，十有八九是病历号发重了。

## 4. 动手教程

### 步骤 1：安装

```bash
pip install -U langgraph
# 用文件/SQL 持久化时按需追加：
# pip install -U langgraph-checkpoint-sqlite       # SqliteSaver
# pip install -U langgraph-checkpoint-postgres     # PostgresSaver
```

### 步骤 2：定义 State 与节点

```python
import random
from typing_extensions import TypedDict

class State(TypedDict):
    number: int
    result: str

def node_a(state: State) -> dict:
    return {"number": random.randint(0, 99)}    # 补丁：只写自己关心的字段（此处随机数以演示两条分支）

def node_b_even(state: State) -> dict:
    return {"result": f"{state['number']} 是偶数"}
```

### 步骤 3：写路由函数与条件边

*（`builder` 是 `StateGraph(State)` 实例：需在步骤 2 之后先写 `builder = StateGraph(State)`，再 add_node）*

```python
def route(state: State) -> str:
    return "even" if state["number"] % 2 == 0 else "odd"

builder.add_conditional_edges("A", route, {"even": "B_even", "odd": "B_odd"})
```

### 步骤 4：组装并编译

*（`node_b_odd` 与 `node_b_even` 对称，需自行定义，如返回 `f"{state['number']} 是奇数"`）*

```python
from langgraph.graph import StateGraph, START, END

graph = (StateGraph(State)
    .add_node("A", node_a)
    .add_node("B_even", node_b_even)
    .add_node("B_odd",  node_b_odd)
    .add_edge(START, "A")
    .add_conditional_edges("A", route, {"even": "B_even", "odd": "B_odd"})
    .add_edge("B_even", END).add_edge("B_odd", END)
    .compile())

print(graph.invoke({"number": 0, "result": ""})["result"])
```

### 步骤 5：加 Checkpointer 实现多轮

*（`builder_chat` 是用 ChatState 定义好的 StateGraph 实例，需先 add_node 并连好边）*

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph.message import add_messages
from typing import Annotated

class ChatState(TypedDict):
    messages: Annotated[list, add_messages]        # 关键：add_messages reducer

chat = builder_chat.compile(checkpointer=InMemorySaver())

cfg = {"configurable": {"thread_id": "user-123"}}   # thread_id = 会话唯一标识
chat.invoke({"messages": [("user", "我叫张三")]}, cfg)
r = chat.invoke({"messages": [("user", "我叫什么名字？")]}, cfg)   # 能答出「张三」
```

### 步骤 6：验证隔离性

```python
alice = {"configurable": {"thread_id": "user-alice"}}
bob   = {"configurable": {"thread_id": "user-bob"}}
chat.invoke({"messages": [("user", "我叫Alice")]}, alice)
chat.invoke({"messages": [("user", "我叫什么？")]}, bob)   # 答不出 Alice
```

### 步骤 7：查看/回滚状态（时间旅行）

```python
snapshot = chat.get_state(cfg)
print(snapshot.values, snapshot.next)
chat.update_state(cfg, {"messages": [("user", "换个话题")]})   # 修改后继续
```

## 5. 完整示例（带持久化的对话图）

```python
import random
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import InMemorySaver

class ChatState(TypedDict):
    messages: Annotated[list, add_messages]

def chatbot(state: ChatState) -> dict:
    from langchain.chat_models import init_chat_model
    model = init_chat_model("openai:gpt-4o", temperature=0)
    return {"messages": [model.invoke(state["messages"])]}

graph = (StateGraph(ChatState)
         .add_node("chat", chatbot)
         .add_edge(START, "chat")
         .add_edge("chat", END)
         .compile(checkpointer=InMemorySaver()))

cfg = {"configurable": {"thread_id": "demo-1"}}
graph.invoke({"messages": [("user", "我叫张三，记住我")]}, cfg)
print(graph.invoke({"messages": [("user", "我叫什么？")]}, cfg)["messages"][-1].content)
```

## 6. 练习

**基础**：给上面的奇偶图加一个"重新生成"分支：当数字 > 90 时回到节点 A 重新生成（构成循环）。别忘了同时设 `recursion_limit` 防死循环。

**进阶**：实现一个"计数器"图：每经过一个节点 `count` 加 1（用 `Annotated[int, operator.add]`），跑 5 次 `invoke` 验证它跨调用累加——体会 Checkpointer 的持久化。

**挑战**：做一个"带审批的发布流程"图：`draft → review（条件边：通过则 publish，否则回 draft 重写）→ END`，用同一个 `thread_id` 中途 `update_state` 修改草稿后继续。

<details>
<summary>参考答案要点</summary>

- 基础：`add_conditional_edges("A", lambda s: "retry" if s["number"]>90 else "ok", {"retry":"A", ...})`；`graph.invoke(..., {"recursion_limit": 10})`。
- 进阶：`import operator` 后写 `count: Annotated[int, operator.add]`，节点 `return {"count": 1}`；必须同 `thread_id` 才累加。
- 挑战：条件边返回 `"rewrite"` 回到 draft 节点；`update_state` 后可 `invoke(None, cfg)` 从中断处继续。
</details>

## 7. 自测清单

- [ ] 能说清节点返回值是"补丁"
- [ ] 能写出条件边 + 路由函数 + 分支映射
- [ ] 知道 `messages` 必须用 `add_messages` reducer
- [ ] 能在 `compile(checkpointer=...)` 注入检查点
- [ ] 能解释 `thread_id` 相同/不同分别意味着什么

## 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 多轮对话"失忆" | `messages` 没用 `add_messages`，被整段覆盖 | `Annotated[list, add_messages]` |
| 加了 checkpointer 仍不记忆 | 忘记在 `invoke` 传 `thread_id` | `config={"configurable": {"thread_id": ...}}` |
| 两个用户串台 | 用了同一个 `thread_id` | 每会话一个 id（用户 id / 会话 id） |
| `InvalidUpdateError` | 节点返回了 State 里没有的 key | 检查 State 定义与返回 key 一致 |
| 图跑不完/爆栈 | 循环没有出口 | 加条件出口 + `recursion_limit` |
| 进程重启后状态没了 | 用了 `InMemorySaver` | 换 Sqlite / Postgres |

## 9. 延伸

- 阶段 07：`create_agent` 就是 LangGraph 上的一个预置图
- 阶段 12：Store 长期记忆（跨 thread）
- 阶段 13：`interrupt` 中断与人工审批

## 10. 记忆强化

**口诀**：

> 节点交补丁，reducer 定合并；条件边看路，兜底要记清；checkpointer 存快照，thread_id 分会话。

**5 分钟回顾闪卡**（先默答，再展开核对）：

1. 问：节点函数的返回值是"新状态"还是"补丁"？
   <details><summary>点击看答案</summary>是补丁（dict），只写自己关心的字段，按 State 里各字段的 reducer 合并进全局状态；没提到的字段原样保留。不是返回整个新 State。</details>
2. 问：为什么 `messages` 要写成 `Annotated[list, add_messages]`？不加会怎样？
   <details><summary>点击看答案</summary>add_messages 是 reducer，规定新消息追加进列表而非覆盖。不加它、直接返回新 list，历史消息会被整段覆盖，多轮对话就"失忆"了。</details>
3. 问：`thread_id` 相同和不同分别意味着什么？
   <details><summary>点击看答案</summary>相同 thread_id 的多次 invoke 共享并累积同一份 state（同一人的连续对话）；不同 thread_id 完全隔离（不同用户互不串台）。一个编译好的图靠它并发服务多会话。</details>
4. 问：Checkpointer 是在什么时机存状态的？
   <details><summary>点击看答案</summary>每个 super-step（每个节点执行完）把 State 快照写入存储，下次调用前自动读回。没有它图是无状态的。</details>
5. 问：路由函数 `route` 拿到的参数是什么？返回值是什么？
   <details><summary>点击看答案</summary>只读当前 state，返回一个分支名字符串（如 "even"/"odd"），再由 add_conditional_edges 的 mapping 把名字映射到下一个节点。它不修改 state。</details>
6. 问：`InMemorySaver` 和 `SqliteSaver` / `PostgresSaver` 怎么选？
   <details><summary>点击看答案</summary>InMemorySaver 用于开发测试、进程重启即丢；SqliteSaver 本地文件持久；PostgresSaver 上生产。进程重启后状态没了，先检查是不是用了内存版。</details>

**费曼任务**：

- 用大白话向一位"完全不懂编程的朋友"讲清【为什么节点返回补丁而不是整个新状态、reducer 又是干嘛的】，限时 2 分钟（建议录音/对着镜子讲）。哪里卡住、哪里要回头翻书，那个点就是你还没真正懂的点——回到对应小节重看后再讲一遍。

---

上一阶段：[05 阶段05-LCEL](05-阶段05-LCEL.md) ｜ 返回[总览与学习路线](00-总览与学习路线.md) ｜ 下一阶段：[07 阶段07-create_agent](07-阶段07-create_agent.md)
