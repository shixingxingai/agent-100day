# 阶段 01：ChatModel —— 统一模型入口与消息模型

> 定位：整个生态的地基 ｜ 难度 ★☆☆ ｜ 预计 2 小时

> **【一句话记住】**：换模型只改一个字符串，业务代码零改动。
>
> **【生活类比】**：`init_chat_model` 像一个万能电源插座——不管你插的是 OpenAI、Anthropic 还是 Google 的插头（模型），插座输出的电压接口（`invoke/stream/batch`）永远一样。`SystemMessage` 是前台经理交代规矩，`HumanMessage` 是顾客开口点单，`AIMessage` 是服务员上一轮的应答——下一轮对话要把它一起端回去，服务员才记得你说过什么。

## 1. 学习目标

学完你应该能够：

- [ ] 用 `init_chat_model` 一行初始化任意厂商的模型，并说清它"统一"了什么
- [ ] 区分 `SystemMessage` / `HumanMessage` / `AIMessage`，并正确组装多轮对话历史
- [ ] 从返回的 `AIMessage` 中取出文本、token 用量和元信息
- [ ] 在不改动业务代码的前提下，把 OpenAI 模型换成 Anthropic / Google 模型

## 2. 前置知识

- Python 基础与虚拟环境（`python -m venv`、`pip install`）
- 环境变量的使用（`os.environ` 或 `.env` + `python-dotenv`）

## 3. 核心概念

### 3.1 `init_chat_model` 是什么

它是 LangChain 的**统一模型初始化入口**。给它一个形如 `"openai:gpt-4o"` 的字符串，它会自动导入对应厂商的集成包（如 `langchain-openai`）并返回实例。

**统一在哪里**：无论底层是 OpenAI、Anthropic 还是 Google，返回的都是同一个 `ChatModel` 接口对象，都支持：

| 通用方法 | 作用 |
|---------|------|
| `.invoke(x)` | 同步单次调用 |
| `.stream(x)` | 流式返回 |
| `.batch([x1, x2])` | 批量并发 |
| `.ainvoke` / `.astream` | 异步版本 |
| `temperature` 等参数 | 跨厂商一致的通用参数 |

因此换模型只改一个字符串，业务代码零改动。

> **记忆钩子**：`init_chat_model("openai:gpt-4o")` 里的冒号就像"厂商:型号"——记住这个格式，换厂商就是换冒号前的词，像换电视频道一样顺手。

### 3.2 三种消息类型

| 类型 | 含义 | 使用场景 |
|------|------|----------|
| `SystemMessage` | 系统级指令，设定角色、语气、输出格式与边界 | "你是一位耐心的 Python 助教，回答控制在 100 字以内" |
| `HumanMessage` | 用户输入，真正的问题或指令 | "这段代码为什么报 KeyError？" |
| `AIMessage` | AI 的历史回复（模型输出也是 `AIMessage`） | 多轮对话时把上一轮回答塞回列表以保留上下文 |

一次调用就是把 `[SystemMessage, HumanMessage, AIMessage, HumanMessage, ...]` 按时间顺序传给 `invoke()`。

> **记忆钩子**：模型是金鱼记忆——它没有上一轮的记忆，每次调用都是"失忆重看"。你不把上一轮的 `AIMessage` 塞回列表，它就像第一次见你，根本不记得刚才聊过什么。

## 4. 动手教程

### 步骤 1：安装依赖

```bash
pip install -U "langchain>=0.3" langchain-openai
```

### 步骤 2：配置密钥（不要硬编码进代码）

```bash
export OPENAI_API_KEY="sk-xxx"
# 或在项目根目录建 .env：OPENAI_API_KEY=sk-xxx，并把 .env 写进 .gitignore
# Windows PowerShell：.venv\Scripts\Activate.ps1；$env:OPENAI_API_KEY="sk-xxx"
```

### 步骤 3：初始化模型并发送一条消息

```python
from langchain.chat_models import init_chat_model
from langchain_core.messages import HumanMessage

model = init_chat_model("openai:gpt-4o", temperature=0)
response = model.invoke([HumanMessage(content="用一句话解释什么是 LangChain")])

print(response.content)          # 文本正文
print(type(response).__name__)   # AIMessage
```

### 步骤 4：换成别的厂商（体会"统一"）

```python
claude = init_chat_model("anthropic:claude-sonnet-4-6", temperature=0)  # 需 pip install langchain-anthropic
gemini = init_chat_model("google_genai:gemini-2.5-flash", temperature=0)  # 需 pip install langchain-google-genai + GOOGLE_API_KEY
# 用 VertexAI 则写 "google_vertexai:..."，需 pip install langchain-google-vertexai + GCP 凭据
# 调用方式完全一致：claude.invoke("...")
```

### 步骤 5：组装多轮对话

*（沿用上文步骤 3 中已定义的 `model`）*

```python
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage

messages = [
    SystemMessage("你是 Python 助教，回答不超过 50 字。"),
    HumanMessage("什么是列表推导式？"),
    AIMessage("一种用一行 for 循环生成列表的简洁写法，如 [x*2 for x in range(3)]。"),
    HumanMessage("那字典推导式呢？"),      # 模型能看到上一轮的 AIMessage
]
print(model.invoke(messages).content)
```

### 步骤 6：运行时可配置模型（进阶）

```python
configurable = init_chat_model(temperature=0)     # 不指定模型
configurable.invoke("你好", config={"configurable": {"model": "gpt-4o-mini"}})
configurable.invoke("你好", config={"configurable": {"model": "claude-sonnet-4-6"}})
```

## 5. 完整示例

```python
import os
from langchain.chat_models import init_chat_model
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage

os.environ["OPENAI_API_KEY"] = "sk-xxx"          # 生产请放 .env

model = init_chat_model("openai:gpt-4o", temperature=0)

messages = [
    SystemMessage("你是 Python 助教，只回答 Python 相关问题，否则回复『超出范围』。"),
    HumanMessage("如何读取一个 CSV 文件？"),
]

resp = model.invoke(messages)
print(resp.content)
print(resp.usage_metadata)   # token 用量（AIMessage 的顶层属性；OpenAI 原始字段在 response_metadata["token_usage"]）

# 把模型的回复追加进历史，继续第二轮
messages += [resp, HumanMessage("那怎么写回 CSV？")]
print(model.invoke(messages).content)
```

## 6. 练习

**基础**：写一个函数 `ask(question: str, history: list) -> str`，接收历史消息列表，返回模型回答（内部追加 `AIMessage` 后返回新历史）。

**进阶**：用 `init_chat_model` 分别初始化 `gpt-4o` 和 `gpt-4o-mini`，对同一问题对比输出与 `usage_metadata`，体会成本差异。

**挑战**：实现一个"运行时可配置模型"的 CLI，用 `--model` 参数在不改代码的情况下切换三个厂商。

<details>
<summary>参考答案要点</summary>

- 基础：`history + [HumanMessage(q)]` → `invoke` → 返回 `(resp.content, history + [HumanMessage(q), resp])`。
- 进阶：`usage_metadata` 里有 `input_tokens` / `output_tokens`，两者价格不同。
- 挑战：`config={"configurable": {"model": args.model}}` 即可，注意提前安装对应集成包。
</details>

## 7. 自测清单

- [ ] 能不看文档写出 `init_chat_model("openai:gpt-4o")` 并调用
- [ ] 能说出三种消息各自的作用与典型场景
- [ ] 知道"模型输出的是 `AIMessage`，取文本用 `.content`"
- [ ] 能解释"换模型只改一个字符串"的原因
- [ ] 密钥不出现在代码里

## 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| `ModuleNotFoundError: langchain_openai` | 没装厂商集成包 | `pip install langchain-openai` |
| 多轮对话模型"失忆" | 没把上一轮的 `AIMessage` 追加进列表 | 每次调用后 `messages.append(resp)` |
| `response.content` 是空 | 可能触发了工具调用（见阶段 03） | 打印 `response.tool_calls` 检查 |
| 账单意外增长 | 循环里反复调用 + 用了贵模型 | 阶段 19 会系统解决 |
| 中文输出被截断 | `max_tokens` 太小 | 调大或省略该参数（部分新模型改用 `max_completion_tokens`） |

## 9. 延伸

- `model.with_structured_output(Schema)`：让模型直接输出结构化对象（阶段 02 会用到）
- `.stream()`：打字机效果（阶段 16 系统讲）

## 10. 记忆强化

**口诀**：

> 冒号分厂商，接口都一样；系统定规矩，人来问，AI 答；回复塞回历史里，下轮才不忘。

**5 分钟回顾闪卡**（先默答，再展开核对）：

1. 问：`init_chat_model("anthropic:claude-sonnet-4-6")` 中冒号前后各代表什么？
   <details><summary>点击看答案</summary>冒号前是厂商提供方（如 openai / anthropic / google_vertexai），冒号后是具体模型名。返回的对象与 OpenAI 模型共用同一套 ChatModel 接口，invoke/stream/batch 用法完全一致。</details>
2. 问：多轮对话为什么模型会"失忆"？怎么修？
   <details><summary>点击看答案</summary>因为模型本身无记忆，每次调用都是无状态的。必须把上一轮返回的 `AIMessage` 追加进消息列表，连同新的 `HumanMessage` 一起传给 `invoke()`。</details>
3. 问：三种消息类型 `SystemMessage` / `HumanMessage` / `AIMessage` 各管什么？
   <details><summary>点击看答案</summary>SystemMessage 设定角色、语气、输出格式与边界；HumanMessage 是用户输入；AIMessage 是模型的历史回复（模型输出本身就是 AIMessage）。</details>
4. 问：从 `AIMessage` 里怎么取文本正文和 token 用量？
   <details><summary>点击看答案</summary>正文用 `response.content`；token 用量用顶层属性 `response.usage_metadata`（含 input_tokens / output_tokens / total_tokens）。注意它不在 `response_metadata` 里——后者是 provider 原始信息，OpenAI 下的原始字段名是 `token_usage`。</details>
5. 问：为什么说"换模型只改一个字符串"？
   <details><summary>点击看答案</summary>因为所有厂商返回的都是同一个 ChatModel 接口对象，通用方法（invoke/stream/batch/ainvoke）和通用参数（temperature 等）跨厂商一致，所以业务代码不用动。</details>
6. 问：`response.content` 为空时可能是什么原因？
   <details><summary>点击看答案</summary>很可能是模型触发了工具调用（tool_calls），此时 content 为空、tool_calls 里才有内容。打印 `response.tool_calls` 即可确认（详见阶段 03）。</details>

**费曼任务**：

- 用大白话向一位"完全不懂编程的朋友"讲清【为什么换模型只改一个字符串】，限时 2 分钟（建议录音/对着镜子讲）。哪里卡住、哪里要回头翻书，那个点就是你还没真正懂的点——回到对应小节重看后再讲一遍。

---

上一阶段：[总览与学习路线](00-总览与学习路线.md) ｜ 返回[总览与学习路线](00-总览与学习路线.md) ｜ 下一阶段：[02 阶段02-Prompt与输出解析](02-阶段02-Prompt与输出解析.md)
