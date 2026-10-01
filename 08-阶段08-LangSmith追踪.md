# 阶段 08：LangSmith 追踪与排障

> 定位：没有可观测性就没有优化 ｜ 难度 ★☆☆ ｜ 预计 2 小时

> **【一句话记住】**：一次请求录全程，错在哪步一眼清。
>
> **【生活类比】**：trace 像行车记录仪。print() 只拍到出发和到达两张照片，中间撞在哪、谁变道全靠猜；trace 把全程录下来，每次模型/工具调用都是录像里的一个时间戳节点，连成 run 树后倒带即可定位事故点。

## 1. 学习目标

- [ ] 说清 LangSmith 解决的核心问题
- [ ] 配好环境变量，让 LangChain/LangGraph 代码**零改动**自动上报
- [ ] 在 Web 界面定位一条 trace，读懂 run 树、耗时、token、报错
- [ ] 用 tags / metadata 标记 trace，便于筛选

## 2. 前置知识

- 阶段 01–07（至少跑通一个 agent）

## 3. 核心概念

LLM 应用的失败是**过程性**的：答案错了，你不知道是提示词没说清、检索召回错了、还是工具参数传错，而 `print()` 只能看到首尾。

LangSmith 把一次请求中的每次模型调用、工具调用、检索记录成**带层级的 run 树（trace）**，保存每一步的输入、输出、耗时、token 与报错堆栈。核心价值：**从"猜哪里错了"变成"看到哪一步错了"**。

> **记忆钩子**：把 trace 想成手术全程录像——print 只拍到病人出院的照片，trace 把下刀顺序、每步失血、哪一步溅血都录下来，错在哪刀一眼回放。

> 本阶段只聚焦**追踪**。评测（datasets / experiments）见阶段 10。

> **记忆钩子**：追踪是"事故录像"，评测是"考试成绩单"；录像帮你当场排错，成绩单帮你发版前拦回归——别拿一段录像当分数。

## 4. 动手教程

### 步骤 1：注册并拿密钥

打开 `smith.langchain.com` 注册（选数据区域，之后不可更改）→ Settings → API Keys → Create API Key（只显示一次）。

### 步骤 2：安装 SDK

```bash
pip install -U langsmith openai
```

### 步骤 3：设置环境变量

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=lsv2_...
export LANGSMITH_PROJECT=weather-agent-dev        # 可选，默认 default
# EU/APAC 账号必设（末尾不要带斜杠）：
export LANGSMITH_ENDPOINT=https://eu.api.smith.langchain.com
# Windows PowerShell：.venv\Scripts\Activate.ps1；$env:LANGSMITH_TRACING="true"；$env:LANGSMITH_API_KEY="lsv2_..."
```

Windows PowerShell：`setx LANGSMITH_TRACING true`（持久写入用户环境变量，**需重开终端才生效**；当前会话用 `$env:LANGSMITH_TRACING="true"`）。更推荐写 `.env` + `python-dotenv`，并把 `.env` 加进 `.gitignore`。

### 步骤 4：跑代码（无需改一行）

*（沿用上文阶段 07 已定义的 `agent`）*

```python
result = agent.invoke({"messages": [{"role": "user", "content": "上海明天天气如何"}]})
# -> 示例输出（以实际运行为准）：agent 返回的消息字典，同步在 LangSmith 项目里出现一条 trace
```

### 步骤 5：打标签，方便筛选

```python
result = agent.invoke(
    {"messages": [{"role": "user", "content": "上海明天天气如何"}]},
    config={"tags": ["dev", "weather-agent"],
            "metadata": {"user_id": "u-001", "env": "local"}},
)
```

### 步骤 6：只对单次调用开启（可选）

*（沿用上文已定义的 `agent` 与 `inputs`；`inputs` 即本次要传入的输入字典）*

```python
from langchain_core.tracers import LangChainTracer
tracer = LangChainTracer(project_name="debug-run")
agent.invoke(inputs, config={"callbacks": [tracer]})
# -> 示例输出（以实际运行为准）：该次调用单独记录到 debug-run 项目，不污染默认项目
```

### 步骤 7：查看 trace

打开 `smith.langchain.com` → 左侧 **Tracing Projects** → 选项目 → 点开一条 trace → 看树形/瀑布视图。

### 步骤 8：非 LangChain 代码也能追踪

*（`OpenAI` 需先 `from openai import OpenAI`；其余沿用上文已定义对象）*

```python
from langsmith import traceable
from langsmith.wrappers import wrap_openai
from openai import OpenAI

client = wrap_openai(OpenAI())      # 自动记录每次 OpenAI 调用
# -> 包装后 client 的每次 chat/completions 调用都会上报成一条 run

@traceable(run_type="tool")
def my_retriever(q: str) -> str: ...
```

## 5. 完整示例

把阶段 07 的 agent 接上追踪，跑一次**能看到完整 run 树**的调用：

```python
# trace_demo.py
import os

from langchain.agents import create_agent
from langchain_core.tools import tool

os.environ.setdefault("LANGSMITH_TRACING", "true")
os.environ.setdefault("LANGSMITH_PROJECT", "weather-agent-dev")


@tool
def get_weather(city: str) -> str:
    """查询指定城市的当前天气。"""
    return f"{city}：晴，26℃"


agent = create_agent("openai:gpt-4o-mini", tools=[get_weather],
                     system_prompt="你是天气助手，必须调用工具查天气，不要凭记忆回答。")

result = agent.invoke(
    {"messages": [{"role": "user", "content": "上海明天天气如何"}]},
    config={"tags": ["dev", "weather-agent"], "metadata": {"user_id": "u-001", "env": "local"}},
)
print(result["messages"][-1].content)
# -> 示例输出（以实际运行为准）：一段引用工具结果的天气回答；同时 LangSmith 项目里出现一条完整 trace
```

跑完到 LangSmith 项目里打开这条 trace，你应该能看到：

- **完整 run 树与执行顺序**：model → ChatOpenAI → 工具 → 再调模型，几次调用、什么顺序一目了然

  > **1.x 注意**：agent 的流式节点名是 **`"model"`**（0.x 的教程里写的是 `"agent"`）。若你按节点名过滤事件（`chunk["metadata"]["langgraph_node"] == "model"`），照抄老代码会**不报错但永远匹配不到**。见 [阶段 16 §3](16-阶段16-流式与服务化.md)。
- **每步精确输入/输出**：完整 messages（含 system prompt 与工具 schema）、模型返回的 `tool_calls` 及参数、工具的返回值
- **耗时瀑布图**：每层的 latency，判断是模型慢、网络慢还是工具慢
- **token 用量与成本**：input/output tokens、模型名、调用次数
- **报错与堆栈**：失败步骤标红，带完整 traceback，不用复现即可定位
- **运行上下文**：run_id、temperature、tags/metadata；配合 Checkpointer 还能按 `thread_id` 串起多轮会话

> **这就是追踪的价值**：同一段代码不开追踪时你只看到"最终答案"；开了之后，"哪一步慢、哪一步错、哪一步贵"全在图上——阶段 10 的评测、阶段 17 的 CI、补充篇 26 的线上监控，全都建立在"看得见"这件事上。

## 6. 练习

**基础**：故意把工具参数类型写错（如让 `celsius_to_fahrenheit` 收到字符串），在 LangSmith 里找到失败步骤，截图保存 `error` 字段。

**进阶**：连续发起 5 次调用，比较每次的 token 与 latency，找出最贵最慢的一次，说明原因。

**挑战**：给同一段代码分别用 `default` 和自建项目两个 `LANGSMITH_PROJECT` 各跑一次，体会"开发/生产分离"；再给生产 trace 打上 `env=prod` 的 tag 并只用 tag 过滤出来。

<details>
<summary>参考答案要点</summary>

- 基础：让 `celsius_to_fahrenheit(celsius: float)` 收到 `"20°C"` 这类字符串，工具内部 `float()` 转换会抛 `ValueError`。在 LangSmith 里展开那条 trace 的 tool 子节点，`error` 字段会显示异常类型与堆栈——报错能**定位到具体节点**，这正是 trace 的核心价值。
- 进阶：在 trace 列表里按 latency 排序，逐条看 `total_tokens` 与耗时。最贵最慢的通常是**输出更长**（output_tokens 高）或**多跑了一轮工具循环**（消息轮次多）。结论：成本与延迟主要由"输出长度 + 轮次"决定，而不是由提问长短决定。
- 挑战：靠 `LANGSMITH_PROJECT` 环境变量区分（不设时落到 `default` 项目）；给生产 trace 打 tag 用 `config={"tags": ["env=prod"], "metadata": {...}}`，再在 UI 用 tag 过滤器只筛 `env=prod`。项目隔离 + tag 过滤 = 开发/生产分离的最小方案。
</details>

## 7. 自测清单

- [ ] 能独立配好 4 个核心环境变量
- [ ] 知道 EU/APAC 必须设置 `LANGSMITH_ENDPOINT` 且末尾不带斜杠
- [ ] 能在界面上找到 run 树、token、耗时、报错四项信息
- [ ] 会用 tags / metadata 做筛选
- [ ] 知道密钥不能进 Git

## 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 界面上没有 trace | `LANGSMITH_TRACING` 不是 `true` | 检查拼写与是否生效（`echo $LANGSMITH_TRACING`） |
| 鉴权失败 | 区域不匹配 | 设置对应 `LANGSMITH_ENDPOINT` |
| 报错 "trailing slash" | URL 末尾多了 `/` | 去掉 |
| trace 混在一起 | 没设 `LANGSMITH_PROJECT` | 开发/生产分项目 |
| 敏感数据进了 trace | 未脱敏 | 用 anonymizer 或 `PIIMiddleware`（阶段 18） |
| 只看 trace 没有评测 | — | 阶段 10 |

## 9. 延伸

- 阶段 10：把感兴趣的 trace 一键存成数据集，进入评测闭环
- 阶段 17：生产环境 trace 与成本监控

## 10. 记忆强化

**口诀**：

> 四个变量先配齐，零改代码自动记；
> run 树里倒带找错，标签项目好隔离；
> 区域端点别斜杠，密钥莫进 Git 里。

**5 分钟回顾闪卡**（先默答，再展开核对）：

1. 问：开启 LangSmith 追踪的四个核心环境变量是什么？
   <details><summary>点击看答案</summary>`LANGSMITH_TRACING=true`、`LANGSMITH_API_KEY`、`LANGSMITH_PROJECT`（可选）、以及 EU/APAC 账号必设的 `LANGSMITH_ENDPOINT`（末尾不带斜杠）。</details>
2. 问：界面上完全看不到 trace，最先排查什么？
   <details><summary>点击看答案</summary>先查 `LANGSMITH_TRACING` 是否真的等于字符串 `true`（注意拼写、是否 export 生效），再查鉴权/区域是否匹配。</details>
3. 问：trace 里 run 树、耗时、token、报错分别帮你判断什么？
   <details><summary>点击看答案</summary>run 树看执行顺序与调用了几次模型/工具；耗时瀑布图判断慢在模型、网络还是工具；token 用量算成本；报错标红带堆栈，不用复现即可定位失败步骤。</details>
4. 问：tags 和 metadata 有什么区别、各用来干嘛？
   <details><summary>点击看答案</summary>两者都用于筛选；tags 是扁平字符串标签（如 `["dev","weather-agent"]`），metadata 是结构化键值（如 `user_id`、`env`）。配合 `LANGSMITH_PROJECT` 做开发/生产分离。</details>
5. 问：非 LangChain 原生代码怎么接入追踪？
   <details><summary>点击看答案</summary>用 `wrap_openai(OpenAI())` 包装裸 OpenAI 客户端自动记录；用 `@traceable(run_type="tool")` 装饰自定义函数，把它也挂进 run 树。</details>
6. 问：为什么密钥不能进 Git？
   <details><summary>点击看答案</summary>`LANGSMITH_API_KEY` 只显示一次、泄露即被盗刷；应写进 `.env` 并把 `.env` 加入 `.gitignore`，用 `python-dotenv` 加载。</details>

**费曼任务**：

- 用大白话向一位"完全不懂编程的朋友"讲清【LangSmith 的 run 树追踪】，限时 2 分钟（建议录音/对着镜子讲）。哪里卡住、哪里要回头翻书，那个点就是你还没真正懂的点——回到对应小节重看后再讲一遍。

---

上一阶段：[07 create_agent](07-阶段07-create_agent.md) ｜ 返回[总览与学习路线](00-总览与学习路线.md) ｜ 下一阶段：[09 多代理架构](09-阶段09-多代理架构.md)
