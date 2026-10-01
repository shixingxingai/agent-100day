# 阶段 10：评测 Evals —— 岗位第一差异化能力

> 定位：JD 中出现率 56%，是"会用框架"与"能交付"的分水岭 ｜ 难度 ★★★ ｜ 预计 8 小时

> **【一句话记住】**：改代码不靠手感靠分数，负样本和阈值是命门。
>
> **【生活类比】**：离线评测是每次改车都上检测线，在线监控是上路后盯仪表盘。RAGAS 四指标像体检分诊：Faithfulness 查"有没有照抄课本胡编"，Response Relevancy 查"有没有答非所问"，Context Precision 查"带的参考资料有没有用"，Context Recall 查"课本有没有给全"——先分诊，再开药。

## 1. 学习目标

- [ ] 说清为什么 agent 必须有评测（非确定性系统的迭代依据）
- [ ] 搭出"数据集 + 目标函数 + 评估器"的离线评测闭环
- [ ] 理解 RAGAS 四指标，能用它定位"是检索错了还是生成错了"
- [ ] 把评测接进 CI，让每次改动都有回归结论（回归：确认改动没把原有能力改坏）

## 2. 前置知识

- 阶段 04（RAG）、07（agent）、08（LangSmith 追踪）

## 3. 核心概念

### 3.1 两层评测

| 层 | 做什么 | 频率 |
|----|--------|------|
| **离线评测** | 一份数据集（输入 + 期望/rubric）+ 一批评估器，改代码就跑一遍 | 每次 PR |
| **在线监控** | 生产 trace 抽样打分：延迟、成本、失败率、用户反馈 | 持续 |

类比：离线评测是单元测试，在线监控是 APM。

> **记忆钩子**：离线评测是"驾校每次练车都模拟考"，在线监控是"真上路后看仪表盘"；只练车不上路会脱节，只上路不模拟考，剐蹭了都不知道是哪次改出来的。

### 3.2 三类评估器

| 类型 | 做法 | 适用 |
|------|------|------|
| 规则/代码 | 断言、正则、JSON Schema、字段比对 | 有确定答案：分类、抽取 |
| 统计指标 | 与参考答案比：精确率、编辑距离、BLEU | 摘要、翻译 |
| LLM-as-judge | 强模型按 rubric 打分（0–1 + 理由） | 开放式问答、有用性、语气 |

LLM-as-judge 要点：**rubric（评分标准）要具体**（"是否给出了可执行的下一步"而不是"好不好"）、**要求输出理由**（便于复核）、**定期用人工标注校准**。

### 3.3 RAGAS 四指标（RAG 必测）

| 指标 | 问的问题 | 低了说明 |
|------|---------|---------|
| **Faithfulness** | 答案是否只基于召回内容？ | 模型在胡编 → 改 prompt / 换模型 |
| **Response Relevancy** | 答案是否回应了问题？ | 答非所问 → 改 prompt / 改写 query |
| **Context Precision** | 召回的内容有用吗？ | 噪声挤占上下文 → 加 Rerank |
| **Context Recall** | 该召回的都召回了吗？ | 漏召回 → 改切分 / 混合检索 |

> **排查口诀**：Faithfulness 低 = 生成的问题；Context Recall 低 = 检索的问题。先分清是哪一类，再动手。

> **记忆钩子**：别让考生替教材背锅——Faithfulness 低是"考生拿着课本还瞎编"（生成/提示词的锅），Context Recall 低是"老师根本没把课本发全"（检索/切分的锅）；分数掉了先查是谁的锅，再改对应的那一端。

> **名称版本坑**：0.2 之前第二个指标叫 `AnswerRelevancy`（Answer），0.2 之后改名 **`ResponseRelevancy`**（Response），指标导入也从"小写单例"改成了"类"。**照抄旧博客会直接 `ImportError`**，详见步骤 7。

## 4. 动手教程

### 步骤 1：安装与配置

```bash
pip install -U langsmith ragas deepeval
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=lsv2_...
# Windows PowerShell：.venv\Scripts\Activate.ps1；$env:LANGSMITH_TRACING="true"；$env:LANGSMITH_API_KEY="lsv2_..."
```

### 步骤 2：建数据集（≥20 条，含负样本）

```python
from langsmith import Client
client = Client()

dataset = client.create_dataset(dataset_name="weather-agent-v1")
client.create_examples(
    inputs=[{"question": "上海明天天气如何，气温换算成华氏度是多少"},
            {"question": "北京昨天的天气？"}],        # 负样本：资料里没有
    outputs=[{"must_contain": "华氏"}, {"must_contain": "不知道"}],
    dataset_id=dataset.id,
)
```

> 也可以在 LangSmith 网页上传 CSV/JSON，或**把线上出问题的 trace 一键加入数据集**——这是最实用的数据源。

### 步骤 3：写目标函数

*（`agent` 是阶段 07 已建好的被测 agent；目标函数就是"喂一条输入，吐一个待测输出"）*

```python
def target(inputs: dict) -> dict:
    r = agent.invoke({"messages": [{"role": "user", "content": inputs["question"]}]})
    return {"answer": r["messages"][-1].content}
# -> 返回如 {"answer": "上海明天 25℃，约 77°F"} 的字典
```

### 步骤 4：写评估器

*（`judge` 是用强模型按 rubric 打分用的模型实例，建议 `temperature=0`）*

```python
def contains_check(run, example) -> bool:                      # 规则型
    return example.outputs["must_contain"] in run.outputs["answer"]
# -> 命中返回 True，否则 False

def helpfulness(run, example) -> dict:                         # LLM-as-judge
    score = judge.invoke([{"role": "user", "content":
        f"按有用性给回答打分（0~1，只输出数字）。\n问题：{example.inputs['question']}\n回答：{run.outputs['answer']}"}]).content
    return {"key": "helpfulness", "score": float(score.strip())}
# -> 返回如 {"key": "helpfulness", "score": 0.8}
```

### 步骤 5：跑评测

```python
from langsmith import evaluate
results = evaluate(target, data="weather-agent-v1", evaluators=[contains_check, helpfulness])
```

### 步骤 6：对比实验（改 prompt 前后的 A/B）

*（`target_v2` 是改用新 prompt 后的目标函数）*

```python
evaluate(target_v2, data="weather-agent-v1",
         evaluators=[contains_check, helpfulness],
         experiment_prefix="v2-stricter-prompt")
```

在 Experiments 页面并排看两版的分数与逐条差异。

### 步骤 7：RAG 场景加 RAGAS

> **前置（RAGAS 0.2+ 必读）**：这些指标都是 **LLM-as-judge + 嵌入相似度**实现的，所以 `evaluate()` 必须显式给两样东西——`llm`（当判官的模型）和 `embeddings`（算相似度用）。**不传就用它的默认模型**，要么因为没配 API key 直接报错，要么跑出一堆无意义的默认分，**而你根本不知道它到底用什么打的分**。另外指标导入在 0.2 之后从"小写单例对象"改成了"类"，`from ragas.metrics import faithfulness` 会 `ImportError`。

```bash
pip install -U ragas datasets        # datasets 用于把 pandas / list 转成 ragas 的 Dataset
```

```python
# eval_rag.py —— 对 RAG 流水线跑 RAGAS
from datasets import Dataset
from ragas import evaluate as ragas_evaluate
from ragas.metrics import (              # 0.2+ 是类；0.1 及更早是同名小写单例
    Faithfulness,                        # 答案是否忠于召回内容  → 低 = 生成端问题
    ResponseRelevancy,                   # 答案是否答到点上      → 低 = 生成端问题
    ContextPrecision,                    # 召回的有用吗          → 低 = 检索排序问题
    ContextRecall,                       # 该召回的召回了吗      → 低 = 检索召回问题
)
from langchain.chat_models import init_chat_model
from langchain_openai import OpenAIEmbeddings

judge_llm = init_chat_model("openai:gpt-4o-mini", temperature=0)  # 判官：便宜够用即可
emb = OpenAIEmbeddings(model="text-embedding-3-small")

# 1) 跑一遍你的 RAG，攒够评测样本（每条一行的 dict）
rows = []
for case in rag_cases:                     # [{"user_input":..., "reference":...}, ...]
    docs = retriever.invoke(case["user_input"])
    rows.append({
        "user_input": case["user_input"],
        "reference": case.get("reference"),            # 只有 Context Recall/需要它
        "response": answer(case["user_input"]),        # 你的 RAG 最终答案
        "retrieved_contexts": [d.page_content for d in docs],
    })

dataset = Dataset.from_list(rows)

# 2) 评测：llm 与 embeddings 是必填项，别省
result = ragas_evaluate(
    dataset,
    metrics=[Faithfulness(llm=judge_llm, embeddings=emb),
             ResponseRelevancy(llm=judge_llm, embeddings=emb),
             ContextPrecision(llm=judge_llm, embeddings=emb),
             ContextRecall(llm=judge_llm, embeddings=emb)],
    llm=judge_llm,                # 也可在 evaluate 层统一给，指标层的优先
    embeddings=emb,
    experiment_name="rag-v1",      # 实验名会同步到 LangSmith，方便并排比历史
)
print(result)                      # -> 示例输出（以实际运行为准）：四个指标各一个 0~1 均值
```

**怎么用这四个数**（这才是关键，别只看总分）：

| 现象 | 结论 | 改哪里 |
|------|------|--------|
| Faithfulness 低 | 答案在胡编，检索其实召回了 | 改 prompt / 加"资料没写就说不知道" / 换模型 |
| Response Relevancy 低 | 答非所问 | 改写 query、加 few-shot |
| Context Recall 低 | 该召回的没召回 | 改 chunk_size/overlap、加混合检索、换 embedding |
| Context Precision 低 | 召回了一堆噪声挤占上下文 | 加重排（Rerank）、收窄召回数量 |

> **坑**：Context Precision / Recall 需要 `reference`（参考答案）。没有标注参考答案的样本，要么人工标一批，要么**只跑前两个指标**——硬跑会因缺列直接 `ValueError`。另外 `raise_exceptions=False`（默认）会把单条失败吞掉、只体现在低分上，**第一次跑建议设 `raise_exceptions=True`** 好立刻看到是哪条样本炸了。

### 步骤 8：接进 CI

```yaml
# .github/workflows/eval.yml
- name: Run evals
  env:
    LANGSMITH_API_KEY: ${{ secrets.LANGSMITH_API_KEY }}
  run: python evals/run.py --fail-under 0.85     # 低于阈值就让 CI 红
```

## 5. 完整示例

```python
# evals/run.py
from langsmith import Client, evaluate
from langchain.chat_models import init_chat_model
from langchain.agents import create_agent

client = Client()
agent = create_agent(init_chat_model("openai:gpt-4o", temperature=0), tools=[...])
judge = init_chat_model("openai:gpt-4o", temperature=0)     # 判官模型复用，避免每条样本重建

def target(inputs: dict) -> dict:
    r = agent.invoke({"messages": [{"role": "user", "content": inputs["question"]}]})
    return {"answer": r["messages"][-1].content}

def answer_ok(run, example) -> bool:
    """期望内容是否出现在答案里（负样本的期望词就是"不知道"这类拒答表达）"""
    ans, want = run.outputs["answer"], example.outputs["must_contain"]
    return want in ans or any(w in ans for w in ("不知道", "未提及", "无法"))

def groundedness(run, example) -> dict:
    """LLM-as-judge：回答质量打分（0~1）"""
    s = judge.invoke(
        [{"role": "user", "content":
          f"给下面这个回答的质量打分（0~1，只输出数字）。\n"
          f"问题：{example.inputs['question']}\n回答：{run.outputs['answer']}"}]
    ).content
    return {"key": "groundedness", "score": float(s.strip())}

if __name__ == "__main__":
    import sys
    results = evaluate(target, data="weather-agent-v1",
                       evaluators=[answer_ok, groundedness], experiment_prefix="ci")
    # --fail-under：汇总所有评估器得分均值，低于阈值就让 CI 红（退出码 1）
    if "--fail-under" in sys.argv:
        threshold = float(sys.argv[sys.argv.index("--fail-under") + 1])
        scores = [er.score
                  for row in results                                    # ExperimentResults 可迭代
                  for er in row["evaluation_results"]["results"]
                  if er.score is not None]
        avg = sum(scores) / len(scores) if scores else 0.0
        print(f"平均分 {avg:.3f}（阈值 {threshold}）")
        if avg < threshold:
            sys.exit(1)
```

## 6. 练习

**基础**：为阶段 07 的天气 agent 建 20 条数据集（15 条正常 + 5 条应拒答），跑一次评测并记录基线分数。

**进阶**：把 system_prompt 改严格一版，再跑一次，用 Experiments 对比页说明哪类样本变好、哪类变差。

**挑战**：给你的 RAG（阶段 04）加 RAGAS 四指标；若 Context Recall 低，改 chunk 参数后再测，用数据证明改动有效。

<details>
<summary>参考答案要点</summary>

- 基础：用 `client.create_dataset` / `create_examples` 建 15 条正常 + 5 条应拒答（`outputs={"should_refuse": True}`），跑 `evaluate(target, data=..., evaluators=[...])`，把第一次的 mean score 记为基线。**先有基线，才能判断后续改动是变好还是变坏。**
- 进阶：同一数据集、换 `experiment_prefix` 再跑一次，在 Experiments 对比页逐条 diff。典型现象：严格版**拒答类样本变好**（5 条全过），但**正常样本的 helpfulness 下降**（过度拒答）。要点：单看总均分没有意义，要**按样本类型**看得失。
- 挑战：`Faithfulness`（答案是否忠于检索资料）、`ResponseRelevancy`（是否答到点上）、`ContextPrecision` / `ContextRecall`（召回的是不是有用、该召回的召没召回）。Context Recall 低说明**检索环节漏召回**，改 chunk_size/overlap 或加混合检索后重测，用两次分数对比证明改动有效。要点：**`llm` 与 `embeddings` 必须显式传**，否则 RAGAS 用自己的默认模型打分，你不知道分数是谁给的。
</details>

## 7. 自测清单

- [ ] 能说出离线评测与在线监控的区别
- [ ] 能独立建数据集、写目标函数与评估器并跑通
- [ ] 能解释 RAGAS 四个指标各自在测什么
- [ ] 知道"Faithfulness 低 ≠ 检索有问题"
- [ ] 能把评测接进 CI 并设置阈值

## 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 评测分数虚高 | 数据集太简单/全是正样本 | 加负样本、边界样本 |
| LLM-as-judge 不稳定 | rubric 模糊、温度不为 0 | rubric 写具体 + `temperature=0` + 多跑取均值 |
| 改了 prompt 分数没变 | 评估器测错了东西 | 检查评估器是否真的对齐业务目标 |
| 只测一次就下结论 | 非确定性 | 每条样本跑 3 次取平均 |
| 数据集建完就扔 | 没把线上 bug 沉淀进去 | 建立"线上问题 → 入数据集"的流程 |
| 评测很慢 | 样本多 + judge 用贵模型 | 抽样 + judge 用小模型 + 并行 |

## 9. 延伸

- `deepeval` / `promptfoo`：另一套评测栈，适合 prompt 回归
- 阶段 17：把评测放进 CI/CD
- 阶段 19：把成本指标也纳入评测

## 10. 记忆强化

**口诀**：

> 离线跑数据，在线看监控；
> 规则测确定，judge 测开放；
> faithfulness 低怪生成，recall 低怪检索；
> 负样本必加，阈值卡 CI。

**5 分钟回顾闪卡**（先默答，再展开核对）：

1. 问：离线评测和在线监控的分工是什么？
   <details><summary>点击看答案</summary>离线评测 = 一份数据集 + 评估器，改代码就跑一遍（每次 PR），像单元测试；在线监控 = 生产 trace 抽样打分，看延迟/成本/失败率/反馈，持续进行，像 APM。</details>
2. 问：三类评估器分别适合什么？
   <details><summary>点击看答案</summary>规则/代码型适合有确定答案的（分类、抽取）；统计指标（精确率、编辑距离、BLEU）适合摘要/翻译；LLM-as-judge 按 rubric 打分，适合开放式问答、有用性、语气。</details>
3. 问：RAGAS 的 Faithfulness 低了该改哪一端？
   <details><summary>点击看答案</summary>Faithfulness 低是"答案没基于召回内容、模型在胡编"，属生成端问题——改 prompt 或换模型，不要去动检索。</details>
4. 问：Context Recall 低说明什么？
   <details><summary>点击看答案</summary>该召回的资料没召到，属检索端问题——改切分策略、embedding 或检索器，而不是改生成提示词。</details>
5. 问：LLM-as-judge 打分不稳定怎么办？
   <details><summary>点击看答案</summary>rubric 写具体、要求输出理由、judge 用 `temperature=0`、每条样本多跑几次取均值，定期用人工标注校准。</details>
6. 问：数据集为什么一定要加负样本？
   <details><summary>点击看答案</summary>全是正样本会让分数虚高，无法发现"该拒答时却硬答"的问题；负样本（如资料里根本没有的问题）专门验证拒答能力。</details>
7. 问：怎么把评测接进 CI？
   <details><summary>点击看答案</summary>在 workflow 里跑 `evaluate(...)` 并设 `--fail-under 0.85` 阈值，低于阈值 CI 就红，拦住回归。</details>

**费曼任务**：

- 用大白话向一位"完全不懂编程的朋友"讲清【RAGAS 四指标如何区分"检索错了还是生成错了"】，限时 2 分钟（建议录音/对着镜子讲）。哪里卡住、哪里要回头翻书，那个点就是你还没真正懂的点——回到对应小节重看后再讲一遍。

---

上一阶段：[09 多代理架构](09-阶段09-多代理架构.md) ｜ 返回[总览与学习路线](00-总览与学习路线.md) ｜ 下一阶段：[11 上下文工程](11-阶段11-上下文工程.md)
