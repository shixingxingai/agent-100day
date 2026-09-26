# 阶段 10：评测 Evals —— 岗位第一差异化能力

> 定位：JD 中出现率 56%，是"会用框架"与"能交付"的分水岭 ｜ 难度 ★★★ ｜ 预计 8 小时

> **【一句话记住】**：改代码不靠手感靠分数，负样本和阈值是命门。
>
> **【生活类比】**：离线评测是每次改车都上检测线，在线监控是上路后盯仪表盘。RAGAS 三指标像体检分诊：Faithfulness 查"有没有照抄课本胡编"，Answer Relevancy 查"有没有答非所问"，Context Recall 查"课本有没有给全"——先分诊科再开药。

## 1. 学习目标

- [ ] 说清为什么 agent 必须有评测（非确定性系统的迭代依据）
- [ ] 搭出"数据集 + 目标函数 + 评估器"的离线评测闭环
- [ ] 理解 RAGAS 三元组，能用它定位"是检索错了还是生成错了"
- [ ] 把评测接进 CI，让每次改动都有回归结论

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

LLM-as-judge 要点：**rubric 要具体**（"是否给出了可执行的下一步"而不是"好不好"）、**要求输出理由**（便于复核）、**定期用人工标注校准**。

### 3.3 RAGAS 三元组（RAG 必测）

| 指标 | 问的问题 | 低了说明 |
|------|---------|---------|
| **Faithfulness** | 答案是否只基于召回内容？ | 模型在胡编 → 改 prompt / 换模型 |
| **Answer Relevancy** | 答案是否回应了问题？ | 答非所问 → 改 prompt / 改写 query |
| **Context Precision / Recall** | 召回的有用吗？该召回的召到了吗？ | 检索不行 → 改切分 / 检索策略 |

> **排查口诀**：Faithfulness 低 = 生成的问题；Context Recall 低 = 检索的问题。先分清是哪一类，再动手。

> **记忆钩子**：别让考生替教材背锅——Faithfulness 低是"考生拿着课本还瞎编"（生成/提示词的锅），Context Recall 低是"老师根本没把课本发全"（检索/切分的锅）；分数掉了先查是谁的锅，再改对应的那一端。

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

*（`judge_chain` 是用强模型按 rubric 配好的打分链，建议 `temperature=0`）*

```python
def contains_check(run, example) -> bool:                      # 规则型
    return example.outputs["must_contain"] in run.outputs["answer"]
# -> 命中返回 True，否则 False

def helpfulness(run, example) -> dict:                         # LLM-as-judge
    score = judge_chain.invoke({"question": example.inputs["question"],
                                "answer": run.outputs["answer"]})
    return {"key": "helpfulness", "score": float(score)}
# -> 返回如 {"key": "helpfulness", "score": 0.8}
```

### 步骤 5：跑评测

```python
from langsmith import evaluate
results = evaluate(target, data="weather-agent-v1", evaluators=[contains_check, helpfulness])
```

### 步骤 6：对比实验（改 prompt 前后的 A/B）

```python
evaluate(target_v2, data="weather-agent-v1",
         evaluators=[contains_check, helpfulness],
         experiment_prefix="v2-stricter-prompt")
```

在 Experiments 页面并排看两版的分数与逐条差异。

### 步骤 7：RAG 场景加 RAGAS

```python
from ragas import evaluate as ragas_evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

# 需要：question / answer / contexts / ground_truth 四列
```

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

def target(inputs: dict) -> dict:
    r = agent.invoke({"messages": [{"role": "user", "content": inputs["question"]}]})
    return {"answer": r["messages"][-1].content}

def refusal_ok(run, example) -> bool:
    """负样本必须拒答；正样本必须给出实质内容"""
    ans = run.outputs["answer"]
    if example.outputs.get("should_refuse"):
        return "不知道" in ans or "无法" in ans
    return len(ans.strip()) > 10

def groundedness(run, example) -> dict:
    """LLM-as-judge：答案是否只基于给定资料"""
    s = judge.invoke({"context": example.inputs.get("context", ""), "answer": run.outputs["answer"]})
    return {"key": "groundedness", "score": float(s)}

if __name__ == "__main__":
    evaluate(target, data="weather-agent-v1", evaluators=[refusal_ok, groundedness],
             experiment_prefix="ci")
```

## 6. 练习

**基础**：为阶段 07 的天气 agent 建 20 条数据集（15 条正常 + 5 条应拒答），跑一次评测并记录基线分数。

**进阶**：把 system_prompt 改严格一版，再跑一次，用 Experiments 对比页说明哪类样本变好、哪类变差。

**挑战**：给你的 RAG（阶段 04）加 RAGAS 三指标；若 Context Recall 低，改 chunk 参数后再测，用数据证明改动有效。

## 7. 自测清单

- [ ] 能说出离线评测与在线监控的区别
- [ ] 能独立建数据集、写目标函数与评估器并跑通
- [ ] 能解释 RAGAS 三个指标各自在测什么
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

- 用大白话向一位"完全不懂编程的朋友"讲清【RAGAS 三指标如何区分"检索错了还是生成错了"】，限时 2 分钟（建议录音/对着镜子讲）。哪里卡住、哪里要回头翻书，那个点就是你还没真正懂的点——回到对应小节重看后再讲一遍。

---

上一阶段：[09 多代理架构](09-阶段09-多代理架构.md) ｜ 返回[总览与学习路线](00-总览与学习路线.md) ｜ 下一阶段：[11 上下文工程](11-阶段11-上下文工程.md)
