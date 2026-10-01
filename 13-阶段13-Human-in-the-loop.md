# 阶段 13：Human-in-the-loop（中断与人工审批）

> 定位：agent 要产生真实副作用时的安全阀 ｜ 难度 ★★☆ ｜ 预计 3 小时

> **【一句话记住】**：副作用前先暂停，同 thread 再恢复。
>
> **【生活类比】**：像银行大额取现的人工复核。钱要动、邮件要发这种不可逆动作，机器先停在半路等人按确认；人点头后从断点继续，而不是从头再跑一遍。没有 Checkpointer 就像没存盘的游戏——人一走，进度全丢。

## 1. 学习目标

- [ ] 用 `interrupt` 在关键节点暂停，把结构化信息交给人
- [ ] 用 `Command(resume=...)` + 同一 `thread_id` 恢复执行
- [ ] 用内置 `HumanInTheLoopMiddleware` 快速给工具加审批
- [ ] 知道中断必须配合 Checkpointer 才有效

## 2. 前置知识

- 阶段 06（Checkpointer / thread_id）、07（agent）

## 3. 核心概念

Agent 能调用工具 = 能造成真实副作用（转账、发邮件、删数据、下订单）。高风险动作必须在执行前**暂停等人工确认**。

> **记忆钩子**：只读操作（查天气、搜资料）随便跑；一旦动作是"动真格的钱、信、数据"，就像开车上高速前先拉手刹——机器再自信也得等人点头，因为错了撤不回。

LangGraph 的实现：节点里调用 `interrupt(payload)` → 图在此处抛出中断并**保存检查点** → 外部拿到 payload → 用户决策。随后用 `Command(resume=decision)` 恢复，从中断处继续。

**两个硬性前提**：
1. 必须编译进 Checkpointer（否则中断后状态丢失，无法恢复）
2. 恢复时必须用**同一个 `thread_id`**

> **记忆钩子**：中断 = 游戏即时存档，恢复 = 读档继续。读档必须读同一个存档槽（thread_id）；换个槽读档，人物就从头开始，前面算的钱、攒的草稿全白干。

## 4. 动手教程

### 步骤 1：在节点里中断

```python
from langgraph.types import interrupt

def send_email_node(state):
    draft = draft_email(state)
    decision = interrupt({                    # 在此暂停
        "action": "send_email",
        "draft": draft,
        "question": "确认发送这封邮件？",
    })
    if decision["type"] == "approve":
        return {"result": send(draft)}
    return {"result": f"用户取消：{decision.get('reason')}"}
```

### 步骤 2：编译并触发

*（`builder` 是已建好的 StateGraph；`InMemorySaver` 从 `langgraph.checkpoint.memory` 导入）*

```python
graph = builder.compile(checkpointer=InMemorySaver())
cfg = {"configurable": {"thread_id": "t-1"}}
res = graph.invoke({"to": "a@b.com", "body": "..."}, cfg)

# 取出中断信息（旧写法，仍兼容但已 deprecated）
for i in res["__interrupt__"]:          # Interrupt 对象组成的 tuple
    payload = i.value                    # 你 interrupt() 传进去的那个 dict

# ✅ 1.x 推荐写法：直接读 result.interrupts，不用碰 "__interrupt__" 这个魔法键
for i in res.interrupts:
    payload = i.value
# -> 取 payload：{"action": "send_email", "draft": ..., "question": ...}
```

> **为什么要换写法**：`res["__interrupt__"]` 依赖字典里的魔法键名，内部实现一改就失效（官方已标 deprecated）。`result.interrupts` 是**属性访问**，拼错会直接 `AttributeError`——这正是你想要的：**响亮地失败**，而不是悄悄返回 `None`。
>
> **怎么判断有没有被中断**：不要用 `if "__interrupt__" in res`，用 `if result.interrupts:`——空 tuple 表示正常跑完了。
>
> ⚠️ 中断生效的前提是 **必须有 checkpointer**：没有它，图无法存下"执行到哪一步"，`interrupt()` 会直接抛错。这条也是阶段 27 实战项目里给 `build_agent` 加断言的原因。

### 步骤 3：恢复执行

```python
from langgraph.types import Command

graph.invoke(Command(resume={"type": "approve"}), cfg)
# 或：graph.invoke(Command(resume={"type": "reject", "reason": "收件人不对"}), cfg)
```

### 步骤 4：在 create_agent 里用内置中间件

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware

agent = create_agent(model, tools, checkpointer=InMemorySaver(), middleware=[
    HumanInTheLoopMiddleware(interrupt_on={
        "send_email": {"allowed_decisions": ["approve", "edit", "reject"]},
        "delete_record": True,
    })
])
```

### 步骤 5：修改后再继续（edit 决策）

**注意区分两种恢复格式**：自定义节点 `interrupt()` 的 resume 值格式自定（`interrupt()` 的返回值就是你传给 `Command(resume=...)` 的值）。而 `HumanInTheLoopMiddleware`（步骤 4）的 resume 必须是 `{"decisions": [...]}` 结构：

```python
# 场景 A：自定义节点 interrupt() —— 格式自定，先改状态再恢复
graph.update_state(cfg, {"draft": "修改后的草稿"})     # 先改状态（key 必须是图 state 里真实存在的字段）
graph.invoke(Command(resume={"type": "approve"}), cfg) # 再恢复

# 场景 B：HumanInTheLoopMiddleware —— 必须用 decisions 结构
graph.invoke(Command(resume={"decisions": [{"type": "approve"}]}), cfg)
# edit 决策要带修改后的动作：
graph.invoke(Command(resume={"decisions": [{"type": "edit",
    "edited_action": {"name": "send_email", "args": {"to": "new@b.com", "body": "..."}} }]}), cfg)
# reject 的说明文本键名是 "message"（不是 "reason"）：
graph.invoke(Command(resume={"decisions": [{"type": "reject",
    "message": "收件人不对"}]}), cfg)
```

> **两套 resume 结构的键名不一样，别混用**：
>
> | | 自定义 `interrupt()` | `HumanInTheLoopMiddleware` |
> |---|---|---|
> | 外层 | 直接就是你传的 dict | 必须包一层 `{"decisions": [...]}` |
> | 拒绝理由 | 自己定（本教程用 `reason`） | 键名是 **`message`** |
> | 编辑动作 | 自己用 `update_state` 先改 | 用 **`edited_action`** 包裹 |
>
> 中间件实际发出的中断 payload 里含 `action_requests` 与 `review_configs` 两个字段——前端渲染审批卡片时读这两个，而不是自定义节点那种扁平 dict。两套结构不一致正是"审批页永远白屏"的常见原因。

### 步骤 6：前端对接

中断 payload 必须**结构化**（不要用纯文本），前端才好渲染审批卡片并把决策回传：

```json
{"action": "send_email", "to": "a@b.com", "subject": "...", "body_preview": "..."}
```

## 5. 完整示例

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt, Command

class State(TypedDict):
    amount: float
    to: str
    result: str

def transfer(state: State) -> dict:
    if state["amount"] > 1000:                       # 大额才审批
        d = interrupt({"action": "transfer", "amount": state["amount"],
                       "to": state["to"], "question": "超过 1000 元，是否继续？"})
        if d["type"] != "approve":
            return {"result": f"已取消：{d.get('reason')}"}
    return {"result": f"已向 {state['to']} 转账 {state['amount']} 元"}

graph = (StateGraph(State).add_node("transfer", transfer)
         .add_edge(START, "transfer").add_edge("transfer", END)
         .compile(checkpointer=InMemorySaver()))

cfg = {"configurable": {"thread_id": "tx-1"}}
r = graph.invoke({"amount": 5000, "to": "bob"}, cfg)
print(r.interrupts[0].value)                         # 等待审批（1.x 写法）
print(graph.invoke(Command(resume={"type": "approve"}), cfg)["result"])
```

## 6. 练习

**基础**：给"删除文件"工具加中断，分别测试 approve 与 reject 两条路径的结果。

**进阶**：实现 `edit` 决策——用户在审批时修改收件人，系统按修改后的值执行（提示：先 `update_state` 再 resume）。

**挑战**：把审批做成 FastAPI 接口：`POST /run` 触发任务并返回 interrupt payload，`POST /resume` 携带决策恢复。体会"长任务 + 异步审批"的服务形态。

<details>
<summary>参考答案要点</summary>

- 基础：工具执行前先 `decision = interrupt({"action": "delete_file", "path": ...})`。approve 走 `Command(resume={"type": "approve"})` 后继续执行删除；reject 走 `Command(resume={"type": "reject"})`，节点内判断为 reject 后**直接返回、不调用删除**。要点：reject 分支必须显式短路，不能靠"忘了调工具"。
- 进阶：自定义节点用 `graph.update_state(cfg, {"to": 新收件人})` 改状态，再 `Command(resume={"type": "approve"})` 恢复；`HumanInTheLoopMiddleware` 则必须用 `Command(resume={"decisions": [{"type": "edit", "edited_action": {"name": "send_email", "args": {...}}}]})`——**两种 resume 格式不能混用**。edit 的本质是"恢复前把状态/动作改掉"。
- 挑战：`POST /run` 触发 `graph.invoke`，把返回里的 `result.interrupts[0].value` 作为 payload 用 200 返回（**注意中断不是错误，返回码仍是 200**）（**中断不是错误**）；`POST /resume` 拿前端决策调 `graph.invoke(Command(resume=决策), cfg)`。要点：中断是服务端主动留下的"待办"，必须把 `thread_id` 一起回给客户端才能恢复。
</details>

## 7. 自测清单

- [ ] 能在节点里写 `interrupt` 并拿到结构化 payload
- [ ] 能用 `Command(resume=...)` 恢复，且用同一 `thread_id`
- [ ] 知道没有 Checkpointer 中断会失效
- [ ] 会用 `HumanInTheLoopMiddleware` 给指定工具加审批
- [ ] 读中断用 `result.interrupts`，分得清两套 resume 结构（`reason` vs `message`）
- [ ] 审批 payload 是结构化的（便于前端渲染）

## 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 中断后恢复报"找不到状态" | 没配 Checkpointer 或换了 thread_id | 两者都要对 |
| 拿到的是纯文本，前端没法渲染 | payload 用了字符串 | 传 dict |
| resume 后从开头重跑 | 用了新的 thread_id | 保持一致 |
| 审批后仍执行了旧参数 | 没 `update_state` 就 resume | 先改状态再恢复 |
| 每个工具都弹审批，体验差 | 审批粒度太粗 | 只对高风险工具 + 设阈值 |
| **用中间件却按自定义节点的格式 resume** | 两者结构不同（中间件要 `{"decisions":[...]}`，reject 的键名是 `message`） | 见步骤 5 的对照表 |
| **前端判断中断用 `if "__interrupt__" in res`** | 旧魔法键已 deprecated | 用 `if result.interrupts:` |

## 9. 延伸

- 阶段 17：长任务队列 + 审批通知
- 阶段 18：HITL 是安全体系的最后一道防线

## 10. 记忆强化

**口诀**：

> 高风险先 interrupt，payload 要结构化；
> 恢复靠 resume 同 thread；
> 改参先 update_state 再批准；
> 大额才批别全拦。

**5 分钟回顾闪卡**（先默答，再展开核对）：

1. 问：`interrupt` 的两个硬性前提是什么？
   <details><summary>点击看答案</summary>① 图必须编译进 Checkpointer（否则中断后状态丢失）；② 恢复时必须用同一个 `thread_id`。缺一就报"找不到状态"或从头重跑。</details>
2. 问：怎么从断点恢复执行？
   <details><summary>点击看答案</summary>用 `graph.invoke(Command(resume={"type": "approve"}), cfg)`，`cfg` 里的 `thread_id` 必须与触发中断时一致；resume 的值会作为 `interrupt()` 的返回值回到节点里。</details>
3. 问：为什么审批 payload 必须是结构化 dict 而不是纯文本？
   <details><summary>点击看答案</summary>结构化 dict（action/to/subject/body_preview）前端才能渲染成审批卡片并把决策按字段回传；纯文本前端没法解析、没法按按钮。</details>
4. 问：用户想改完草稿再批准，正确顺序是什么？
   <details><summary>点击看答案</summary>先 `graph.update_state(cfg, {"draft": "修改后草稿"})` 改状态，再 `Command(resume={"type":"approve"})` 恢复；不 update_state 就 resume 会执行旧参数。</details>
5. 问：不想每个工具都弹审批，怎么控制粒度？
   <details><summary>点击看答案</summary>只对高风险工具挂 `HumanInTheLoopMiddleware`，并加阈值（如金额 > 1000 才中断），低风险动作自动放行。</details>
6. 问：`HumanInTheLoopMiddleware` 怎么给工具加审批？
   <details><summary>点击看答案</summary>在 `interrupt_on` 里按工具名配置：`"send_email": {"allowed_decisions": ["approve","edit","reject"]}` 或 `"delete_record": True`，被点名的工具调用前自动中断等人决策。</details>

**费曼任务**：

- 用大白话向一位"完全不懂编程的朋友"讲清【interrupt 暂停、Checkpointer 存盘、Command(resume) 恢复的完整流程】，限时 2 分钟（建议录音/对着镜子讲）。哪里卡住、哪里要回头翻书，那个点就是你还没真正懂的点——回到对应小节重看后再讲一遍。

---

上一阶段：[12 记忆系统](12-阶段12-记忆系统.md) ｜ 返回[总览与学习路线](00-总览与学习路线.md) ｜ 下一阶段：[14 MCP](14-阶段14-MCP.md)
