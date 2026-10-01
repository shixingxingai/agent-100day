# 前瞻篇 29：语音与实时 Agent（级联三段式 vs 端到端语音）

> 定位：把"打字聊天"升级成"能打电话办事" ｜ 难度 ★★★ ｜ 预计 6 小时
> **建议学习时机**：学完阶段 16（流式与服务化）后。语音本质是"更严格的流式"——延迟预算从"能接受"变成"超过 1s 就明显感觉卡"。

> **【一句话记住】**：语音 agent 的难点从来不是"听懂"，而是**在对方还没说完时就准备好接话、并且在被打断时立刻闭嘴**。
>
> **【生活类比】**：级联三段式（STT→LLM→TTS）像**同声传译的笨办法**——先把整句话抄下来（转写）、再想好怎么答（推理）、再念出来（合成），三个环节串起来，对方能明显感觉到"你在等"。端到端实时模型像**一个真正会聊天的人**——边听边想边说，能听出语气、能被打断、能在对方停顿半秒后就接上。语音交互的体验分水岭，全在这"几百毫秒"里。

## 1. 学习目标

- [ ] 说清级联（STT→LLM→TTS）与端到端实时语音两条路线的取舍
- [ ] 会把语音 agent 的延迟拆成可测量的几段，并知道行业目标值
- [ ] 理解 VAD / 轮次检测（turn detection）与**打断（barge-in）**为什么是语音的核心
- [ ] 能用 Realtime API 的会话事件画出一次完整语音交互
- [ ] 会在语音会话里挂工具（function calling / 远程 MCP）
- [ ] 知道该自建、该用框架（LiveKit Agents / Pipecat）还是该用托管平台

## 2. 前置知识

- 阶段 16（SSE / FastAPI / `astream`）——语音是"全双工版的流式"
- 阶段 03、07（工具与 agent）——语音 agent 仍然是 agent，只是输入输出换成了音频
- 补充篇 24（异步与并发）——实时语音是"长连接 + 高并发"的典型场景

## 3. 核心概念

### 3.1 两条技术路线

| | 级联三段式 | 端到端实时（speech-to-speech） |
|---|---|---|
| 链路 | STT → LLM → TTS，三段串起来 | 单个模型直接"听音频 → 出音频" |
| 延迟 | 三段延迟**相加**，通常 1–3s | 明显更低，可边说边出 |
| 情绪/语气/停顿 | **在转写成文字时就丢了** | 保留在音频里，模型能感知 |
| 打断 | 要自己处理（停 TTS、清缓冲） | 原生支持（VAD + 取消响应） |
| 可控性 | 每段都能换、能看文字日志、好排障 | 中间过程不可见、调试更难 |
| 成本 | 三段分别计费，通常更省 | 音频 token 单价高 |
| 适合 | 对**准确率/合规/可审计**要求高（如电话质检、工单录入） | 对**自然度/延迟**要求高（如客服热线、陪练） |

> **记忆钩子**：级联像"**手写传话**"——每一棒都能查、能改、能留档，但慢；端到端像"**当面聊天**"——快且自然，但事后你只记得聊了什么、记不清每个字。

**实务建议**：**先做级联版把业务跑通**（便宜、可观测、可换模型），把延迟当作一个明确指标去优化；只有当"延迟/自然度"真的成为产品瓶颈时，再评估端到端。

### 3.2 延迟预算：把"感觉卡"拆成可测的数

用户对语音的耐心比打字短得多。行业经验值（**各段的目标区间，不是承诺**）：

| 环节 | 目标 | 说明 |
|------|------|------|
| 采集 + 上行 | 20–50ms | WebRTC 通常比 WebSocket 稳 |
| VAD 判停 | 300–700ms | **这是最大的一块，也是最容易被忽略的** |
| 模型首字/首音频 | 300–800ms | 端到端明显优于级联 |
| 下行 + 播放缓冲 | 50–200ms | 缓冲太大=延迟，太小=卡顿 |

> ⚠️ **别把四段直接相加**。上面是**各自压到最好时的下限**，朴素相加 ≈ 670–1750ms；但真实链路里 **VAD 判停与首音频有重叠**（收到"疑似说完"就先预热，不必等判停结束再请求模型），所以全链路通常落在 **1.2–1.5s**。
>
> 要压到这个量级，靠三件事同时发生：① 用**端到端模型**（级联的 ASR + LLM + TTS 串行，首音频压不下来）；② 用**语义轮次检测**替代固定静音时长；③ 首音频与判停**重叠预热**。
>
> 所以 KPI 别定在四段之和上：**1.2s 左右是合格线，压到 1s 内不错，500ms 级才谈得上"像真人"**——而 500ms 基本要求全链路端到端 + 语义轮次检测。

> **记忆钩子**：**VAD 的"判停等待"往往比你想象的贵**——你以为延迟出在模型，其实大半花在"等对方真的说完了没"。把 `silence_duration_ms` 从 800 调到 500，可能比换模型更立竿见影。

### 3.3 VAD 与轮次检测（turn detection）

VAD（Voice Activity Detection）负责判断"现在有没有人在说话"，轮次检测（判断这一轮该谁开口）据此决定"该我接话了"。两种主流策略：

| 策略 | 原理 | 特点 |
|------|------|------|
| **服务端 VAD** | 按音频能量/静音时长判停 | 简单、可预测；但**抢话**（对方只是思考停一下）就会被打断 |
| **语义轮次检测** | 结合语义判断"这句话说完了吗" | 更自然，不会被中途停顿骗；实现更复杂 |

关键参数就是那几个：`threshold`（多大声算说话）、`prefix_padding_ms`（前置补多少音频）、`silence_duration_ms`（静音多久算说完）。

> **记忆钩子**：`silence_duration_ms` 是**灵敏度旋钮**——调大→不容易抢话但显迟钝；调小→反应快但老插嘴。这跟阶段 13 HITL"审批超时"是同一类"阈值拍脑袋必翻车"的问题，**要用真实录音调**。

### 3.4 打断（barge-in）：语音的"生死线"

真人对话里"打断"是常态（barge-in：用户抢话时立刻停掉正在播放的语音）。语音 agent 必须做到两件事：

1. **立刻停嘴**：取消正在生成的响应、清空播放缓冲；
2. **把已播出的部分写进上下文**：否则模型会以为"我还没说过"，下一轮重复一遍。

在 Realtime API 里大致对应：收到用户开始说话 → 发 `response.cancel` 取消生成 → 清 `output_audio_buffer`。

> **记忆钩子**：**不能被打断的语音 agent，用户 3 秒内就会挂电话**。打断处理是"能不能打电话用"的门槛，不是加分项。

### 3.5 传输：WebRTC / WebSocket / SIP

| 传输 | 用在哪 | 注意 |
|------|--------|------|
| **WebRTC** | 浏览器 / App 端 | 抗抖动、抗丢包最好，**首选** |
| **WebSocket** | 服务端编排（自己接电话网关） | 控制力强，但要自己处理网络抖动 |
| **SIP** | 接传统电话网 / 呼叫中心 | 想"给 agent 一个电话号码"就用它 |

### 3.6 成本：音频 token 比文本贵一个量级

语音的账单结构和文本完全不同——**音频按音频 token 计费，单价远高于文本**，而且"用户说话"和"模型说话"都算。以实时语音模型为例（数量级参考，**以官方最新价目为准**）：

- 音频输入 / 输出：约 **$32 / $64** 每百万 token 量级；
- 文本输入 / 输出：约 **$4 / $24** 每百万 token 量级；
- **缓存音频输入**便宜得多（约 $0.4 量级）——**稳定的 system instructions + 长对话前缀**值得利用；
- 便宜的 mini 档模型，音频价格通常只有旗舰的 1/3 左右。

三条省钱纪律：

1. **能级联就级联**：文字环节用便宜文本模型；
2. **控制会话时长**：长连接不等于无限上下文，该 truncate 就 truncate；
3. **简单请求走 mini**：别用旗舰模型接"查余额"这种活。

## 4. 动手教程

### 步骤 1：先做级联版（最省事、最好排障）

```bash
pip install openai
```

```python
# cascade_voice.py —— STT → LLM → TTS 三段式最小实现
from openai import OpenAI
from langchain.chat_models import init_chat_model

oai = OpenAI()
model = init_chat_model("openai:gpt-4o-mini", temperature=0)

# ① STT：音频 → 文字
with open("question.wav", "rb") as f:
    text = oai.audio.transcriptions.create(model="gpt-transcribe", file=f).text

# ② LLM：文字 → 文字（这一段就是前面 19 章学的 agent，可以挂工具、RAG、记忆）
reply = model.invoke(text).content

# ③ TTS：文字 → 音频（流式写文件，避免长回答等整段合成）
with oai.audio.speech.with_streaming_response.create(
    model="gpt-4o-mini-tts", voice="alloy", input=reply,
) as resp:
    resp.stream_to_file("reply.mp3")

print("用户：", text)
print("助手：", reply)
```

> **要点**：第 ② 步里的 `model` 可以整条换成阶段 07 的 `create_agent(...)`——**级联架构的最大好处就是"语音"被隔离成前后两段，中间的 agent 逻辑完全复用**。

### 步骤 2：转端到端实时（会话不过是一个"事件循环"）

实时语音不是"发一个请求等一个响应"，而是一条**长连接，双方持续互推事件**。形状如下（示意，实际事件名以官方为准）：

```python
# realtime_voice.py —— 端到端语音会话骨架
import asyncio
import json
import os

import websockets          # pip install websockets，≥12 用 additional_headers

API_KEY = os.environ["OPENAI_API_KEY"]
URL = "wss://api.openai.com/v1/realtime?model=gpt-realtime-2.1"     # 旗舰；省钱可用 -mini 档


async def voice_agent():
    headers = {"Authorization": f"Bearer {API_KEY}"}
    async with websockets.connect(URL, additional_headers=headers) as ws:
        # ① 开场先配置会话：输出模态、音频格式、轮次检测、指令
        await ws.send(json.dumps({
            "type": "session.update",
            "session": {
                "type": "realtime",
                "output_modalities": ["audio"],
                "instructions": "你是客服，语气温和，回答控制在两句以内。",
                "audio": {
                    "input": {
                        "format": {"type": "audio/pcm", "rate": 24000},
                        "turn_detection": {
                            "type": "server_vad",
                            "threshold": 0.5,
                            "prefix_padding_ms": 300,
                            "silence_duration_ms": 500,      # ← 最该调的旋钮
                        },
                    },
                    "output": {
                        "format": {"type": "audio/pcm", "rate": 24000},
                        "voice": "alloy",
                    },
                },
            },
        }))

        # ② 麦克风音频持续推上去（每帧一个事件）
        # await ws.send(json.dumps({"type": "input_audio_buffer.append", "audio": b64_chunk}))

        # ③ 消费服务端事件
        async for raw in ws:
            ev = json.loads(raw)
            t = ev["type"]
            if t == "response.output_audio.delta":
                play_audio(ev["delta"])                    # 播放音频分片
            elif t == "response.output_audio_transcript.delta":
                print(ev["delta"], end="", flush=True)     # 同步显示转写文本
            elif t == "input_audio_buffer.speech_started":
                await ws.send(json.dumps({"type": "response.cancel"}))   # 用户插话 → 取消生成
                clear_playback_buffer()                                   # 立刻停嘴
            elif t == "response.done":
                pass


def play_audio(b64_delta: str) -> None: ...
def clear_playback_buffer() -> None: ...


if __name__ == "__main__":
    asyncio.run(voice_agent())
```

要点：**客户端事件**（`input_audio_buffer.append/commit/clear`、`conversation.item.create`、`response.create`、`response.cancel`、`session.update`）与**服务端事件**（`response.output_audio.delta`、`response.output_audio_transcript.delta`、`input_audio_buffer.speech_started`…）大致就是这套。

> **易错点**：老教程里的 `session.voice` / `input_audio_format` / `response.audio.delta` 是**早期 beta 形状**，已随协议调整（音频配置挪进 `session.audio.*`，事件名改为 `response.output_audio.*`）。**照抄旧博客必挂**——以官方文档为准。

### 步骤 3：调轮次检测（最影响体感的一步）

```json
"turn_detection": {
    "type": "server_vad",
    "threshold": 0.5,             // 环境吵就调高，否则空调声都能触发
    "prefix_padding_ms": 300,     // 别把开头的第一个字吃掉
    "silence_duration_ms": 500    // ← 抢话 vs 迟钝的分界线
}
```

调参方法：**录 20 段真实对话**，每段标"哪里真的说完了"，然后扫 `silence_duration_ms ∈ {300, 500, 800}`，统计"被抢话次数"与"感知延迟"，取平衡点。**别问感觉，用数据。**

### 步骤 4：打断处理（写不对就白做）

```python
# 三条底线，缺一不可
# 1) 检测到用户在说话 → 取消当前响应
await ws.send(json.dumps({"type": "response.cancel"}))
# 2) 清空播放缓冲（否则会"嘴上停了、扬声器里还在念"）
clear_playback_buffer()
# 3) 把"已经播出去的部分"记进对话历史，避免下一轮重复回答
append_assistant_partial_transcript(already_spoken_text)
```

第 3 条最常被漏——漏了会出现"你说到一半被打断，然后它下一轮**从头再讲一遍**"的诡异体验。

### 步骤 5：把工具挂进语音会话

语音 agent 一样能调工具（查订单、改预约、发短信）。流程与阶段 03 的 tool_calls 循环同构，只是包在事件里：

```json
{
  "tools": [{
    "type": "function",
    "name": "query_order",
    "description": "按订单号查询状态",
    "parameters": {
      "type": "object",
      "properties": {"order_id": {"type": "string"}},
      "required": ["order_id"]
    }
  }],
  "tool_choice": "auto"
}
```

> ② 收到模型请求参数 → **你自己执行工具**：监听 `response.output_item.done`，筛出 `item.type == "function_call"`，参数在 `item.arguments`（是 JSON 字符串，要 `json.loads`）与 `item.call_id`。
> ③ 把结果塞回会话 → 让模型继续生成：`conversation.item.create` 带 `{"type": "function_call_output", "call_id": call_id, "output": result_json}`，然后 `response.create`。

> **事件名版本坑**：早期协议是 `response.function_call_arguments.done`，GA 之后统一走 `response.output_item.done`（一次事件里同时给出 `function_call` 与 `function_call_output` 两类 item）。**照抄旧博客的事件名会永远等不到回调**——以官方 Realtime 文档为准。

> 较新的实时模型还支持**远程 MCP**——把 MCP server 直接挂进语音会话，省掉自己搬运 `function_call_output` 的胶水（回顾阶段 14）。

### 步骤 6：选型——自建 / 框架 / 平台

| 方案 | 代表 | 适合 |
|------|------|------|
| **自建** | 自己写 WebRTC/WebSocket + 事件循环 | 要完全可控、有基础设施团队；成本最透明 |
| **开源框架** | LiveKit Agents、Pipecat | **推荐起点**：VAD、打断、轮次、多模态编排都封装好了 |
| **托管平台** | Vapi 等语音 agent 平台 | 想几天上线、接受按分钟计费与供应商绑定 |

> **成本经验值**：月通话量不大时，托管平台的"按分钟"通常比自建便宜（省人力）；量上来之后（数万分钟/月起）自建栈才在账单上反超。**先按你的月通话量算一遍，别凭直觉选。**

## 5. 完整示例

一个"语音查订单"的级联版，把 STT、agent、TTS、工具串起来（**能跑通的最小闭环**）：

```python
# voice_order.py
import os

from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain_core.tools import tool
from openai import OpenAI

oai = OpenAI(api_key=os.environ["OPENAI_API_KEY"])


@tool
def query_order(order_id: str) -> str:
    """按订单号查询订单状态。"""
    fake_db = {"A1001": "已发货，预计明天送达", "B2002": "待付款"}
    return fake_db.get(order_id, "没有这个订单号")


agent = create_agent(
    init_chat_model("openai:gpt-4o-mini", temperature=0),
    tools=[query_order],
    system_prompt="你是语音客服。回答必须口语化、不超过两句话，适合朗读；不要输出 Markdown。",
)


def handle_turn(audio_path: str) -> str:
    """一次完整语音回合：音频进 → 音频出，返回转写文本便于观测。"""
    # ① STT
    with open(audio_path, "rb") as f:
        heard = oai.audio.transcriptions.create(model="gpt-transcribe", file=f).text
    # ② Agent（复用前面所有章节的能力）
    reply = agent.invoke({"messages": [("user", heard)]})["messages"][-1].content
    # ③ TTS
    with oai.audio.speech.with_streaming_response.create(
        model="gpt-4o-mini-tts", voice="alloy", input=reply,
    ) as resp:
        resp.stream_to_file("reply.mp3")
    return f"用户：{heard}\n助手：{reply}"


if __name__ == "__main__":
    print(handle_turn("question.wav"))
```

**这个例子要你注意两件事**：

1. **system_prompt 里写了"口语化、不要 Markdown"**——语音场景里模型输出会被**读出来**，`**加粗**`、列表符号、"（见下表）"全是噪声。这是文本 agent 转语音 agent 最容易忘的一步。
2. **TTS 用流式写文件**——长回答别等整段合成完再播，否则延迟直接翻倍。

## 6. 练习

**基础**：把上面例子的 STT 换成对同一段录音的两次调用，比较"先转写再回答"（级联）与"把音频直接给多模态模型"（阶段 01 的 `input_audio` 块）两条路径的**延迟与答案质量**差异。

**进阶**：实现打断：用一个"播放器"线程模拟音频播放，收到 `speech_started` 时立刻停播并打印"已打断，已播长度 = N 毫秒"，验证不会出现"嘴上停了还在念"。

**挑战**：为你的语音 agent 设计一份**延迟拆解表**（采集/VAD/首音频/播放各占多少毫秒），在真实录音上测出三段数字，并回答：如果把 `silence_duration_ms` 从 800 降到 400，总感知延迟下降多少、抢话率上升多少？

<details>
<summary>参考答案要点</summary>

- 基础：级联路径 = `transcriptions.create()` 的往返 + LLM 往返 + TTS 合成；多模态单次路径少了"转写成文字再交给另一个模型"的一跳，**首响通常更快**，但可观测性差（拿不到文字中间态）、成本结构不同（音频 token 贵）。要点：**选型取决于你更需要"延迟"还是"可审计"**，两者可混合（关键环节用级联留档）。
- 进阶：核心是"取消 + 清缓冲 + 记录已播部分"。用 `response.cancel` 取消生成、清播放缓冲停止出声，再把"已播出的转写"追加进历史。**最容易漏的是第三步**，漏了下一轮会从头重复。
- 挑战：典型结果——VAD 判停从 800→400ms 会省下约 300–400ms 感知延迟（占大头），但抢话率明显上升（对方思考停顿被当成说完）。这份表的价值在于：**它把"感觉慢"变成了一个可以优化、可以回归的数字**，这正是补充篇 26（生产监控）在语音场景的落地。
</details>

## 7. 自测清单

- [ ] 能说出级联与端到端各自的取舍，并知道"先级联后实时"的落地顺序
- [ ] 能把语音延迟拆成采集 / VAD / 首音频 / 播放四段
- [ ] 知道 `silence_duration_ms` 是灵敏度旋钮，且要用真实录音调
- [ ] 能说清打断处理的三个必要动作（取消、清缓冲、记已播部分）
- [ ] 知道 WebRTC / WebSocket / SIP 各自的适用场景
- [ ] 知道音频 token 比文本贵一个量级，并说出三条省钱纪律
- [ ] 知道语音场景要把 system_prompt 写成"适合朗读"的口语体

## 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 照抄旧博客的实时 API 代码报错 | 用的是早期 beta 的字段/事件名 | 音频配置用 `session.audio.*`，事件名用 `response.output_audio.*`，以官方文档为准 |
| 感觉"很慢"但说不出慢在哪 | 没做延迟拆解 | 分段计时；**先查 VAD 判停**，它常是最大一块 |
| 老被抢话 | `silence_duration_ms` 太小 / 用了服务端 VAD | 调大判停时长，或改语义轮次检测 |
| 被打断后重复回答 | 没把"已播出的部分"写进上下文 | 记录 partial transcript 再追加 |
| "嘴上停了、扬声器还在念" | 没清播放缓冲 | 打断时清 `output_audio_buffer` |
| 模型把 Markdown 念出来 | system_prompt 没约束输出形式 | 明确"口语化、不要 Markdown、不超过 N 句" |
| 空调声触发应答 | VAD `threshold` 太低且没降噪 | 调高 threshold、加降噪/回声消除 |
| 账单远超预期 | 长连接不断、音频 token 单价高 | 会话超时回收、简单请求走 mini 档、善用缓存音频输入 |
| 升级模型后变得"老插嘴" | VAD 参数是针对旧模型调的 | **换模型要重新校准 VAD / 打断参数** |

## 9. 延伸

- 官方文档：Realtime API（会话事件与工具调用）、Speech-to-Text、Text-to-Speech
- 开源语音 agent 框架：LiveKit Agents、Pipecat
- 阶段 16：流式与服务化（SSE 是"半双工流"，语音是"全双工流"，对比着看）
- 阶段 14：MCP（较新的实时模型支持远程 MCP，语音里直接挂工具集）
- 补充篇 26：把"感知延迟 / 抢话率 / 通话成本"做成线上看板与告警

## 10. 记忆强化

**口诀**：

> 三段串起来慢，端到端最像人；
> 延迟拆四段别相加，先查 VAD 判停；
> 判停是旋钮，抢话迟钝两头挑；
> 打断三件事：取消、清缓冲、记已播；
> 音频 token 贵，能级联就级联。

**5 分钟回顾闪卡**（先默答，再展开核对）：

1. 问：级联（STT→LLM→TTS）和端到端实时语音，各自最大的优势和代价是什么？
   <details><summary>点击看答案</summary>级联：**可观测、可替换、可审计、通常更便宜**，代价是延迟相加、转写时丢掉语气/停顿信息、打断要自己处理。端到端：**延迟低、自然、原生支持打断**，代价是中间过程不可见（难排障）、音频 token 贵。</details>
2. 问：语音 agent 的延迟通常最大的一块在哪？为什么容易被忽略？
   <details><summary>点击看答案</summary>常在 **VAD 判停**这一段——"等对方真的说完"要花 300–700ms 甚至更多。它容易被忽略，因为大家第一反应都去怀疑模型慢，而这段是"采集侧"的参数问题。</details>
3. 问：`silence_duration_ms` 调大调小分别会怎样？
   <details><summary>点击看答案</summary>调大：不容易抢话，但反应显得迟钝；调小：反应快，但对方思考时的短暂停顿就会被当成"说完了"而抢话。**没有通用最优值**，要用真实录音扫参数取平衡点。</details>
4. 问：打断处理必须做哪三件事？少做哪件最要命？
   <details><summary>点击看答案</summary>①取消正在生成的响应（`response.cancel`）；②清空播放缓冲（否则嘴上停了扬声器还在念）；③把"已播出的部分"写进对话上下文。**第③件最容易被漏**——漏了下一轮会从头把话再讲一遍。</details>
5. 问：语音场景为什么要在 system_prompt 里写"不要 Markdown、不超过两句"？
   <details><summary>点击看答案</summary>因为模型的输出会被 **TTS 直接读出来**。`**加粗**`、列表符号、"见下表"这类文本表达在语音里全是噪声。这是从文本 agent 迁到语音 agent 最容易忘的一步。</details>
6. 问：把语音 agent 做上线，三条省钱纪律是什么？
   <details><summary>点击看答案</summary>①能级联就级联（文字环节用便宜的文本模型）；②控制会话时长（长连接不等于无限上下文，该截断就截断）；③简单请求走 mini 档模型。另外要善用**缓存音频输入**（稳定 instructions + 长前缀便宜得多）。</details>

**费曼任务**：

- 用大白话向一位"完全不懂编程的朋友"讲清【为什么"能被打断、打断后不重复"比"答案准不准"更决定语音助手好不好用】，限时 2 分钟（建议录音/对着镜子讲）。哪里卡住、哪里要回头翻书，那个点就是你还没真正懂的点——回到对应小节重看后再讲一遍。

---

上一阶段：[16 流式与服务化](16-阶段16-流式与服务化.md) ｜ 返回[总览与学习路线](00-总览与学习路线.md) ｜ 下一阶段：[30 浏览器与 Computer-use Agent](30-前瞻-浏览器与ComputerUse.md) ｜ 前瞻篇导航：[31 私有化部署与自托管推理](31-前瞻-私有化部署与自托管推理.md)
