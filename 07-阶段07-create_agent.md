# 阶段 07：create_agent —— 开箱即用的 Agent

> 定位：把阶段 03 的手写工具循环产品化 ｜ 难度 ★★☆ ｜ 预计 4 小时

> **【一句话记住】**：一行 create_agent 帮你把"模型调工具直到收工"的循环全包了。
>
> **【生活类比】**：阶段 03 是你自己在后厨一道道炒菜、跑堂传菜；create_agent 像雇了个值班经理——你把菜单（tools）和规矩（system_prompt）交给他，他自动循环"问模型、跑工具、再问模型"直到出菜，还自带记忆（checkpointer）和监控（stream）。中间件就是你能插在值班经理每个动作前后的"规矩补丁"。

## 1. 学习目标

- [ ] 说清 `create_agent` 与旧版 `AgentExecutor` 的核心区别，以及为什么官方推荐前者
- [ ] 用 `create_agent` 创建带多个工具的 agent 并跑通多步调用
- [ ] 理解中间件（middleware）机制，能区分**工具中间件**与**模型中间件**
- [ ] 知道什么时候该用中间件、什么时候改 prompt 就够了

## 2. 前置知识

- 阶段 03（工具与 tool_calls 循环）
- 阶段 06（LangGraph）

## 3. 核心概念

### 3.1 与旧版 AgentExecutor 的区别

| | AgentExecutor（旧） | create_agent（新） |
|---|---|---|
| 底层 | 黑盒循环，不基于 LangGraph | **基于 LangGraph，返回已编译的图** |
| 工具调用 | 把工具描述塞进固定 ReAct 提示词，**靠解析文本** | 用模型**原生 tool calling**（结构化 `tool_calls`） |
| 状态 | 自定义 memory 类 | 标准 `AgentState.messages` + Checkpointer |
| 流式 | 弱 | `.stream(stream_mode=...)` 原生支持 |
| 人工介入 | 不支持 | `interrupt` + 内置 HITL 中间件 |
| 自定义 | 改提示词或继承类 | **中间件**，可插桩循环的每个阶段 |

核心循环没变（调模型 → 执行工具 → 直到不再调工具），但底层换成了可持久化、可观测、可插桩（能在关键位置插入自己的逻辑）的图运行时——这也正是官方推荐它的原因。

> **记忆钩子**：旧 AgentExecutor 是"黑盒录音机"——靠解析模型吐出来的文本来判断调了啥工具，脆；新 create_agent 是"结构化点单"——直接用模型原生 tool_calls，而且底层就是 LangGraph，能存盘、能打断、能插桩。

### 3.2 导入路径

```python
from langchain.agents import create_agent      # 官方导入路径
```

### 3.3 中间件：六个钩子

模型链：`before_agent` → `before_model` → `wrap_model_call` → `after_model` → `after_agent`；工具链：`wrap_tool_call`（包在工具执行前后）。

> 注意：模型—工具循环内 `before_model` / `wrap_model_call` / `after_model` / `wrap_tool_call` 会**多轮重复执行**（每次模型调用、每次工具调用都跑一遍），不是整条链只走一次。

- `before/after_*`：在特定时点跑逻辑
- `wrap_*`：**包裹式**，接收 `(request, handler)`，可改请求再交给 `handler`、可捕获异常重试、可短路返回；另外，`request.override(model=..., tools=...)` 能动态换模型和裁剪工具

| 工具中间件 `wrap_tool_call` | 模型中间件 `before_model` / `wrap_model_call` / `after_model` |
|---|---|
| 拦截**工具执行**：鉴权、参数改写、结果缓存、重试、输出截断 | 拦截**模型调用**：动态提示词、历史裁剪/摘要、模型降级、限流、输出校验 |
| 内置：`ToolCallLimitMiddleware`、`HumanInTheLoopMiddleware`、`ToolRetryMiddleware` | 内置：`SummarizationMiddleware`、`PIIMiddleware`、`ModelFallbackMiddleware`、`LLMToolSelectorMiddleware`、`ModelCallLimitMiddleware` |
| 能拿到 `request.tool`、`request.tool_call`，可返回 `ToolMessage` 覆盖结果 | 能拿到 `ModelRequest`（messages/tools/system_prompt/model），`override` 后交给 `handler` |

**判断标准**：这件事能不能靠改提示词解决？能，就别用中间件；只有在需要确定性、每次都必执行、涉及安全或成本控制时，才用它。

> **记忆钩子**：中间件是"硬规矩"，prompt 是"软说服"。想让模型"最好别删库"，写 prompt 里它偶尔会犯；想"删库一律拦截"，就得用中间件——因为中间件在代码层每次必跑，不靠模型自觉。

## 4. 动手教程

### 步骤 1：定义工具

```python
from langchain_core.tools import tool

@tool
def get_weather(city: str, date: str) -> str:
    """查询指定城市、指定日期的天气与气温（单位：摄氏度）。"""
    data = {("上海","明天"): ("小雨", 22.0), ("北京","明天"): ("晴", 28.0)}
    w, t = data.get((city, date), ("未知", 0.0))
    return f"{city}{date}{w}，气温 {t} 摄氏度"

@tool
def celsius_to_fahrenheit(c: float) -> str:
    """把摄氏温度换算成华氏温度。"""
    return f"{c} 摄氏度 = {c * 9 / 5 + 32:.1f} 华氏度"
```

### 步骤 2：创建 agent

```python
from langchain.chat_models import init_chat_model
from langchain.agents import create_agent

model = init_chat_model("openai:gpt-4o", temperature=0)
agent = create_agent(
    model=model,
    tools=[get_weather, celsius_to_fahrenheit],
    system_prompt="你是天气助手，必须先调用工具拿到真实数据再回答，禁止编造。",
)
```

### 步骤 3：调用并取结果

```python
result = agent.invoke({
    "messages": [{"role": "user", "content": "上海明天天气如何，气温换算成华氏度是多少"}]
})
print(result["messages"][-1].content)
# 上海明天小雨，气温 22 摄氏度，换算后约为 71.6 华氏度。
```

执行链路：`get_weather` → 22°C → `celsius_to_fahrenheit(22)` → 71.6°F → 生成回答。

### 步骤 4：观察每一步（调试必备）

```python
for step in agent.stream({"messages": [...]}, stream_mode="updates"):
    print(step)
```

### 步骤 5：加多轮记忆

```python
from langgraph.checkpoint.memory import InMemorySaver
agent = create_agent(model, tools, checkpointer=InMemorySaver())
cfg = {"configurable": {"thread_id": "u-1"}}
agent.invoke({"messages": [{"role": "user", "content": "我叫张三"}]}, cfg)
agent.invoke({"messages": [{"role": "user", "content": "我叫什么？"}]}, cfg)   # 记得
```

### 步骤 6：写一个自定义中间件

```python
from langchain.agents.middleware import wrap_tool_call
from langchain_core.messages import ToolMessage

@wrap_tool_call
def block_dangerous(request, handler):
    if request.tool_call["name"] == "delete_user":
        # 短路：直接返回拦截消息、不调用 handler，危险操作不会被执行
        return ToolMessage(
            content="该操作已被安全策略拦截",
            name=request.tool_call["name"],
            tool_call_id=request.tool_call["id"],
        )
    return handler(request)

agent = create_agent(model, tools, middleware=[block_dangerous])
```

### 步骤 7：用内置中间件解决常见需求

```python
from langchain.agents.middleware import SummarizationMiddleware, ModelCallLimitMiddleware

agent = create_agent(model, tools, middleware=[
    SummarizationMiddleware(model="gpt-4o-mini", trigger={"tokens": 3000}, keep=("messages", 6)),
    ModelCallLimitMiddleware(run_limit=20),     # 防失控循环：单次运行最多 20 次模型调用（thread_limit 限整个会话线程，需配 checkpointer）
])
```

## 5. 完整示例

*（最小可运行骨架：含工具、Checkpointer 与 thread_id；自定义中间件见步骤 6/7）*

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain_core.tools import tool
from langgraph.checkpoint.memory import InMemorySaver

@tool
def get_weather(city: str, date: str) -> str:
    """查询指定城市、指定日期的天气与气温（摄氏度）。"""
    w, t = {("上海","明天"): ("小雨", 22.0)}.get((city, date), ("未知", 0.0))
    return f"{city}{date}{w}，气温 {t} 摄氏度"

@tool
def celsius_to_fahrenheit(c: float) -> str:
    """把摄氏温度换算成华氏温度。"""
    return f"{c} 摄氏度 = {c * 9 / 5 + 32:.1f} 华氏度"

agent = create_agent(
    model=init_chat_model("openai:gpt-4o", temperature=0),
    tools=[get_weather, celsius_to_fahrenheit],
    system_prompt="你是天气助手，先调用工具拿数据，再回答；资料不足就说不知道。",
    checkpointer=InMemorySaver(),
)

cfg = {"configurable": {"thread_id": "demo"}}
r = agent.invoke({"messages": [{"role": "user", "content": "上海明天天气如何，气温换算成华氏度是多少"}]}, cfg)
print(r["messages"][-1].content)
```

## 6. 练习

**基础**：加入第三个工具 `get_city_info(city)`，问一个需要三个工具协作的问题，用 `stream_mode="updates"` 打印调用顺序。

**进阶**：写一个 `before_model` 中间件，在消息数超过 10 条时打印警告并截断最早的用户消息（体会"上下文压缩"的最小实现）。

**挑战**：写一个 `wrap_model_call` 中间件，根据问题长度动态切换模型（短问题用 `gpt-4o-mini`，长问题用 `gpt-4o`），并用 LangSmith 对比切换前后的成本与质量。

<details>
<summary>参考答案要点</summary>

- 基础：`updates` 模式每次输出一个节点的增量，能看到 agent→tools→agent 的往复。
- 进阶：`request.messages` 可读写，`request.override(messages=...)` 返回修改后的请求。
- 挑战：`len(request.messages)` 或字符数做阈值；成本对比看 `usage_metadata`，质量用阶段 10 的评测集打分。
</details>

## 7. 自测清单

- [ ] 能说出 create_agent 相对 AgentExecutor 的至少三点改进
- [ ] 知道导入路径是 `langchain.agents`
- [ ] 能创建 agent、传工具、配 system_prompt、加 checkpointer
- [ ] 能区分工具中间件和模型中间件，各举一个应用场景
- [ ] 能说出"能用 prompt 解决就别用中间件"的判断标准

## 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 模型不调工具 | 工具 docstring 描述不清 | 明确"当……时使用"，写清参数含义 |
| 多轮不记忆 | 没传 `checkpointer` 或 `thread_id` | 两者都要 |
| 中间件不生效 | 忘了放进 `middleware=[...]` | 检查参数名 |
| agent 无限循环 | 无调用上限 | `ModelCallLimitMiddleware` + `recursion_limit` |
| 结果里一堆中间消息 | 直接打印了 `result["messages"]` | 取 `[-1].content` |
| 成本失控 | 每轮都传全量工具 schema + 历史 | 阶段 11 / 19 |

## 9. 延伸

- 阶段 09：多个 agent 协作
- 阶段 13：HITL 中间件做人工审批
- 阶段 18：安全类中间件

## 10. 记忆强化

**口诀**：

> 一行 create_agent，工具循环自动跑；中间件分两类，包工具还是包模型；prompt 能解决，就别上中间件；call 上限防失控。

**5 分钟回顾闪卡**（先默答，再展开核对）：

1. 问：create_agent 相比旧 AgentExecutor 的三个核心改进是什么？
   <details><summary>点击看答案</summary>底层基于 LangGraph（返回已编译图，可持久化可观测）；用模型原生 tool_calls 而非解析文本；自定义靠中间件插桩每个阶段，而非改提示词或继承类。</details>
2. 问：工具中间件 `wrap_tool_call` 和模型中间件 `before_model` / `wrap_model_call` 分别拦截什么？
   <details><summary>点击看答案</summary>工具中间件拦截工具执行：鉴权、参数改写、结果缓存、重试、输出截断；模型中间件拦截模型调用：动态提示词、历史裁剪/摘要、模型降级、限流、输出校验。</details>
3. 问：什么时候该用中间件、什么时候改 prompt 就够了？
   <details><summary>点击看答案</summary>能靠改提示词解决的就别用中间件；需要确定性、每次必执行、涉及安全或成本控制时才用中间件（因为它在代码层强制生效，不靠模型自觉）。</details>
4. 问：多轮记忆要哪两样东西缺一不可？
   <details><summary>点击看答案</summary>create_agent 时传 checkpointer（如 InMemorySaver），并且每次 invoke 时在 config 里传 thread_id。两者缺一个都记不住。</details>
5. 问：怎么防止 agent 无限循环烧钱？
   <details><summary>点击看答案</summary>加 ModelCallLimitMiddleware(run_limit=...)（单次运行上限）/ thread_limit=...（线程累计上限，需配 checkpointer）限制模型调用次数，同时设 recursion_limit；这是安全兜底，不能只靠 prompt 让模型"适可而止"。</details>
6. 问：调用 agent 后怎么取最终回答，而不是一堆中间消息？
   <details><summary>点击看答案</summary>结果是 messages 列表，取最后一条：`result["messages"][-1].content`。直接打印整个 result["messages"] 会看到工具调用等中间过程。</details>

**费曼任务**：

- 用大白话向一位"完全不懂编程的朋友"讲清【create_agent 到底帮你自动做了阶段 03 的哪些手写工作】，限时 2 分钟（建议录音/对着镜子讲）。哪里卡住、哪里要回头翻书，那个点就是你还没真正懂的点——回到对应小节重看后再讲一遍。

---

上一阶段：[06 阶段06-LangGraph基础](06-阶段06-LangGraph基础.md) ｜ 返回[总览与学习路线](00-总览与学习路线.md) ｜ 下一阶段：[08 阶段08-LangSmith追踪](08-阶段08-LangSmith追踪.md)
