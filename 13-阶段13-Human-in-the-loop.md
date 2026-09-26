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

LangGraph 的实现：节点里调用 `interrupt(payload)` → 图在此处抛出中断并**保存检查点** → 外部拿到 payload → 用户决策 → 用 `Command(resume=decision)` 恢复，从中断处继续。

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
print(res["__interrupt__"])       # 中断信息在这里（Interrupt 对象组成的 tuple，payload 在 .value 属性里）
# -> 取 payload：res["__interrupt__"][0].value（含 draft、question）
```

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

**注意区分两种恢复格式**——自定义节点 `interrupt()` 的 resume 值格式自定（`interrupt()` 的返回值就是你传给 `Command(resume=...)` 的值）；而 `HumanInTheLoopMiddleware`（步骤 4）的 resume 必须是 `{"decisions": [...]}` 结构：

```python
# 场景 A：自定义节点 interrupt() —— 格式自定，先改状态再恢复
graph.update_state(cfg, {"draft": "修改后的草稿"})     # 先改状态（key 必须是图 state 里真实存在的字段）
graph.invoke(Command(resume={"type": "approve"}), cfg) # 再恢复

# 场景 B：HumanInTheLoopMiddleware —— 必须用 decisions 结构
graph.invoke(Command(resume={"decisions": [{"type": "approve"}]}), cfg)
# edit 决策要带修改后的动作：
graph.invoke(Command(resume={"decisions": [{"type": "edit",
    "edited_action": {"name": "send_email", "args": {"to": "new@b.com", "body": "..."}} }]}), cfg)
```

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
print(r.get("__interrupt__"))                        # 等待审批
print(graph.invoke(Command(resume={"type": "approve"}), cfg)["result"])
```

## 6. 练习

**基础**：给"删除文件"工具加中断，分别测试 approve 与 reject 两条路径的结果。

**进阶**：实现 `edit` 决策——用户在审批时修改收件人，系统按修改后的值执行（提示：先 `update_state` 再 resume）。

**挑战**：把审批做成 FastAPI 接口：`POST /run` 触发任务并返回 interrupt payload，`POST /resume` 携带决策恢复。体会"长任务 + 异步审批"的服务形态。

## 7. 自测清单

- [ ] 能在节点里写 `interrupt` 并拿到结构化 payload
- [ ] 能用 `Command(resume=...)` 恢复，且用同一 `thread_id`
- [ ] 知道没有 Checkpointer 中断会失效
- [ ] 会用 `HumanInTheLoopMiddleware` 给指定工具加审批
- [ ] 审批 payload 是结构化的（便于前端渲染）

## 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 中断后恢复报"找不到状态" | 没配 Checkpointer 或换了 thread_id | 两者都要对 |
| 拿到的是纯文本，前端没法渲染 | payload 用了字符串 | 传 dict |
| resume 后从开头重跑 | 用了新的 thread_id | 保持一致 |
| 审批后仍执行了旧参数 | 没 `update_state` 就 resume | 先改状态再恢复 |
| 每个工具都弹审批，体验差 | 审批粒度太粗 | 只对高风险工具 + 设阈值 |

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
