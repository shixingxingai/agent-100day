**中文** | [English](README.en.md)

# LangChain / LangGraph 系统学习教程

本项目是一套面向实践的 LangChain 与 LangGraph 中文学习教程，覆盖从模型调用、提示词和工具，到 RAG、Agent、评测与生产部署的完整学习路径。每个阶段包含学习目标、核心概念、示例、练习和自测内容，并配有记忆卡片与复习材料。

## 学习路径

建议按阶段顺序学习：

1. **基础与应用主线（01–09）**：ChatModel、Prompt 与输出解析、工具、RAG、LCEL、LangGraph、Agent、LangSmith 追踪和多代理。
2. **评测与生产化（10–19）**：评测、上下文工程、记忆、人工审批、MCP、进阶 RAG、流式服务、部署、安全和成本治理。
3. **复习资料（20–23）**：速查与面试追问（含**术语表**与**新旧 API 迁移对照表**）、核心概念地图、记忆口诀与类比、闪卡与间隔重复计划。
4. **补充篇（24–27）**：异步与并发、测试策略、生产监控与告警、**端到端实战项目**——按需插入主线（24 配合阶段 05/16、25 配合阶段 17、26 配合阶段 08/17、27 在全部学完后做）。
5. **进阶 / 前瞻篇（28–32）**：LangGraph 进阶（`Send`/重试/缓存）、语音与实时 Agent、浏览器与 Computer-use、私有化部署与自托管推理、模型微调与 SLM 路由——**按目标岗位选修**，不必全做，但至少要有一项能讲透。

从[总览与学习路线](00-总览与学习路线.md)开始，查看各阶段的目标、预计耗时和 90 天学习计划。迷路时参考[核心概念地图](21-核心概念地图.md)；遇到陌生名词查[术语表](20-附录-速查与面试追问.md)；照抄老教程报错先对[新旧 API 迁移对照表](20-附录-速查与面试追问.md)；复习时使用[闪卡与间隔重复计划](23-闪卡与间隔重复计划.md)和 [`flashcards-anki.csv`](flashcards-anki.csv)。

> **版本前提**：本教程基于 **LangChain 1.x** 与 **LangGraph 1.x** 编写（定稿 2026-09），建议 **Python ≥ 3.10**。1.x 有破坏性变更（导入路径重组、组件拆包、部分能力迁至 `langchain-classic`），详见 00 总览开头的说明与附录 20 的迁移对照表。

## 目录

| 阶段 | 内容 |
| --- | --- |
| 01–03 | [ChatModel](01-阶段01-ChatModel.md)、[Prompt 与输出解析](02-阶段02-Prompt与输出解析.md)、[Tool 工具系统](03-阶段03-Tool工具系统.md) |
| 04–07 | [RAG 流水线](04-阶段04-RAG流水线.md)、[LCEL](05-阶段05-LCEL.md)、[LangGraph 基础](06-阶段06-LangGraph基础.md)、[create_agent](07-阶段07-create_agent.md) |
| 08–10 | [LangSmith 追踪](08-阶段08-LangSmith追踪.md)、[多代理架构](09-阶段09-多代理架构.md)、[评测 Evals](10-阶段10-评测Evals.md) |
| 11–14 | [上下文工程](11-阶段11-上下文工程.md)、[记忆系统](12-阶段12-记忆系统.md)、[Human-in-the-loop](13-阶段13-Human-in-the-loop.md)、[MCP](14-阶段14-MCP.md) |
| 15–19 | [RAG 进阶](15-阶段15-RAG进阶.md)、[流式与服务化](16-阶段16-流式与服务化.md)、[部署与工程化](17-阶段17-部署与工程化.md)、[安全与 Guardrails](18-阶段18-安全与Guardrails.md)、[成本与多模型路由](19-阶段19-成本与多模型路由.md) |
| 20–23 | [附录：速查与面试追问](20-附录-速查与面试追问.md)（含术语表 + 迁移对照表）、[核心概念地图](21-核心概念地图.md)、[记忆口诀与类比大全](22-记忆口诀与类比大全.md)、[闪卡与间隔重复计划](23-闪卡与间隔重复计划.md) |
| 24–27 | [补充：异步与并发](24-补充-异步与并发.md)、[补充：测试策略](25-补充-测试策略.md)、[补充：生产监控与告警](26-补充-生产监控与告警.md)、[实战：端到端 Agent 项目](27-实战-端到端Agent项目.md) |
| 28–32 | [进阶：LangGraph 进阶](28-补充-LangGraph进阶.md)、[前瞻：语音与实时 Agent](29-前瞻-语音与实时Agent.md)、[前瞻：浏览器与 Computer-use](30-前瞻-浏览器与ComputerUse.md)、[前瞻：私有化部署与自托管推理](31-前瞻-私有化部署与自托管推理.md)、[前瞻：模型微调与 SLM 路由](32-前瞻-模型微调与SLM路由.md) |

## 环境准备

教程示例使用 Python。按需安装各章节所需依赖；以下是总览中给出的常用环境示例：

```bash
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1

# macOS / Linux
source .venv/bin/activate

pip install -U langchain langchain-openai langchain-community
pip install -U langchain-text-splitters langchain-chroma pypdf
pip install -U langgraph langsmith pydantic python-dotenv
```

按阶段追加依赖（用到哪章装哪个，各章"步骤 1：安装依赖"有说明）：

```bash
pip install -U langchain-classic rank-bm25 langchain-postgres langgraph-checkpoint-postgres
pip install -U langchain-mcp-adapters langgraph-supervisor ragas fastapi uvicorn
pip install -U "langgraph-cli[inmem]"
```

部分示例需要模型提供方的 API 密钥或 LangSmith 配置。请按照相应章节的说明，通过环境变量或本地环境文件配置；不要把密钥提交到版本控制中。

## 使用建议

- 每章先读学习目标，再运行示例并完成练习和自测。
- 建议从阶段 01 开始按顺序推进；阶段 04、07、10、17 可作为 RAG、Agent、评测和部署的实践关卡。
- 学习时尽早接入追踪和评测，持续记录问题并把典型错误加入自己的评测样本。
- 使用第 23 阶段的 1/3/7/14/30 天复习节奏巩固知识。

更多学习说明和完整路线见[总览与学习路线](00-总览与学习路线.md)。
