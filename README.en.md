[中文](README.md) | **English**

# LangChain / LangGraph: A Systematic Learning Tutorial

This project is a practice-oriented, Chinese-language learning tutorial for LangChain and LangGraph. It covers the full path from model calls, prompts and tools to RAG, agents, evaluation and production deployment. Every stage includes learning objectives, core concepts, examples, exercises and self-checks, along with memory cards and review materials.

## Learning Path

It is recommended to study the stages in order:

1. **Fundamentals & core applications (01–09)**: ChatModel, prompts and output parsing, tools, RAG, LCEL, LangGraph, agents, LangSmith tracing and multi-agent architecture.
2. **Evaluation & productionization (10–19)**: evaluation, context engineering, memory, human approval, MCP, advanced RAG, streaming services, deployment, security and cost management.
3. **Review materials (20–23)**: cheat sheet and interview follow-ups (including a **glossary** and an **old-to-new API migration table**), the core concept map, mnemonics and analogies, and flashcards with a spaced repetition plan.
4. **Supplements (24–27)**: async and concurrency, testing strategies, production monitoring and alerting, and an **end-to-end hands-on project** — insert these into the main path as needed (24 with stages 05/16, 25 with stage 17, 26 with stages 08/17, and 27 after finishing everything else).
5. **Advanced / forward-looking topics (28–32)**: advanced LangGraph (`Send` / retries / caching), voice and real-time agents, browser and computer-use, private deployment and self-hosted inference, and fine-tuning with SLM routing — **take these based on your target role**. You do not need all of them, but at least one should be something you can explain in depth.

Start with the [Overview & Learning Roadmap](00-总览与学习路线.md) for the objectives, estimated time and the 90-day plan. When you feel lost, refer to the [Core Concept Map](21-核心概念地图.md); look up unfamiliar terms in the [Glossary](20-附录-速查与面试追问.md); if code copied from an old tutorial fails, check the [Old-to-New API Migration Table](20-附录-速查与面试追问.md); for review, use the [Flashcards & Spaced Repetition Plan](23-闪卡与间隔重复计划.md) and [`flashcards-anki.csv`](flashcards-anki.csv).

> **Version requirements**: This tutorial is written for **LangChain 1.x** and **LangGraph 1.x** (finalized September 2026), with **Python ≥ 3.10** recommended. The 1.x releases contain breaking changes (reorganized import paths, split packages, and some capabilities moved to `langchain-classic`); see the note at the beginning of the stage-00 overview and the migration table in appendix 20 for details.

## Contents

| Stages | Content |
| --- | --- |
| 01–03 | [ChatModel](01-阶段01-ChatModel.md), [Prompts & Output Parsing](02-阶段02-Prompt与输出解析.md), [Tool System](03-阶段03-Tool工具系统.md) |
| 04–07 | [RAG Pipeline](04-阶段04-RAG流水线.md), [LCEL](05-阶段05-LCEL.md), [LangGraph Basics](06-阶段06-LangGraph基础.md), [create_agent](07-阶段07-create_agent.md) |
| 08–10 | [LangSmith Tracing](08-阶段08-LangSmith追踪.md), [Multi-Agent Architecture](09-阶段09-多代理架构.md), [Evaluation (Evals)](10-阶段10-评测Evals.md) |
| 11–14 | [Context Engineering](11-阶段11-上下文工程.md), [Memory System](12-阶段12-记忆系统.md), [Human-in-the-loop](13-阶段13-Human-in-the-loop.md), [MCP](14-阶段14-MCP.md) |
| 15–19 | [Advanced RAG](15-阶段15-RAG进阶.md), [Streaming & Serving](16-阶段16-流式与服务化.md), [Deployment & Engineering](17-阶段17-部署与工程化.md), [Security & Guardrails](18-阶段18-安全与Guardrails.md), [Cost & Multi-Model Routing](19-阶段19-成本与多模型路由.md) |
| 20–23 | [Appendix: Cheat Sheet & Interview Follow-ups](20-附录-速查与面试追问.md) (glossary + migration table), [Core Concept Map](21-核心概念地图.md), [Mnemonics & Analogies](22-记忆口诀与类比大全.md), [Flashcards & Spaced Repetition Plan](23-闪卡与间隔重复计划.md) |
| 24–27 | [Supplement: Async & Concurrency](24-补充-异步与并发.md), [Supplement: Testing Strategies](25-补充-测试策略.md), [Supplement: Production Monitoring & Alerting](26-补充-生产监控与告警.md), [Hands-On: End-to-End Agent Project](27-实战-端到端Agent项目.md) |
| 28–32 | [Advanced LangGraph](28-补充-LangGraph进阶.md), [Forward Look: Voice & Real-Time Agents](29-前瞻-语音与实时Agent.md), [Forward Look: Browser & Computer-use](30-前瞻-浏览器与ComputerUse.md), [Forward Look: Private Deployment & Self-Hosted Inference](31-前瞻-私有化部署与自托管推理.md), [Forward Look: Fine-tuning & SLM Routing](32-前瞻-模型微调与SLM路由.md) |

## Environment Setup

The examples use Python. Install the dependencies required by each chapter as needed; below is the common environment example from the overview:

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

Add dependencies stage by stage (install only what a chapter needs; each chapter's "Step 1: Install dependencies" has the details):

```bash
pip install -U langchain-classic rank-bm25 langchain-postgres langgraph-checkpoint-postgres
pip install -U langchain-mcp-adapters langgraph-supervisor ragas fastapi uvicorn
pip install -U "langgraph-cli[inmem]"
```

Some examples require an API key from a model provider or a LangSmith configuration. Configure these through environment variables or a local environment file as described in the corresponding chapters; never commit keys to version control.

## Tips for Use

- Read the learning objectives first, then run the examples and complete the exercises and self-checks.
- It is recommended to proceed in order starting from stage 01; stages 04, 07, 10 and 17 serve as practical checkpoints for RAG, agents, evaluation and deployment.
- Turn on tracing and evaluation as early as possible, keep recording issues, and add typical failures to your own evaluation samples.
- Use the 1/3/7/14/30-day review schedule from stage 23 to consolidate what you learn.

For more guidance and the complete roadmap, see the [Overview & Learning Roadmap](00-总览与学习路线.md).
