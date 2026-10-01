# 前瞻篇 32：模型微调与 SLM 路由（LoRA / QLoRA / 蒸馏 / 小模型做路由抽取）

> 定位：兑现 00 总览 P2 清单里的"LoRA 微调与 SLM 用于路由分类抽取" ｜ 难度 ★★★ ｜ 预计 7 小时
> **建议学习时机**：学完阶段 10（评测）、19（模型分级）后；如果要做私有化，先学前瞻篇 31（自托管底座）——**微调产出的权重，最终要落在你自己的推理服务上才有意义**。

> **【一句话记住】**：**微调改"怎么做"，RAG 补"知道什么"，提示词只是"怎么说"**——三件事别互相背锅。
>
> **【生活类比】**：**提示词**像给员工发一张便签（临时交代）；**RAG**像给他配一个随时能查的资料柜（补知识）；**微调**像送他去参加三个月培训，把"手感"练进肌肉记忆（改能力）。员工**不知道公司新政策**，你培训多少次也没用（这是 RAG 的活）；员工**知道政策但格式总写不对**，你塞再多资料也没用（这才是微调的活）。

## 1. 学习目标

- [ ] 用一张决策表判断"该改提示词 / 该上 RAG / 该微调"（面试高频，也是最常答错的一题）
- [ ] 说清 LoRA / QLoRA 的核心参数（`r` / `lora_alpha` / `target_modules` / dropout）与取舍
- [ ] 会准备一份合格的微调数据集（格式、规模、质量、切分）
- [ ] 会用 PEFT + TRL 写出一份能跑的 LoRA / QLoRA 训练脚本骨架
- [ ] 知道"微调后怎么证明它真的变好了"（同一份评测集做前后对比）
- [ ] 会用**小模型做路由/分类/抽取**，并用置信度兜底给大模型
- [ ] 知道蒸馏是什么、什么时候用

## 2. 前置知识

- 阶段 02（`with_structured_output`）——SLM 抽取与路由的输出形态
- 阶段 10（评测）——**没有基线就别谈微调**
- 阶段 19（模型分级与成本）——"小模型干小活"是那一章的深化
- 前瞻篇 31（自托管）——微调权重最终要自己部署

## 3. 核心概念

### 3.1 决策表：提示词 / RAG / 微调，到底该动哪个

| 症状 | 根因 | 该动哪一层 |
|------|------|-----------|
| 模型**不知道**你们的内部规定/新数据 | 缺知识 | **RAG**（补资料，随时更新） |
| 模型**知道**该做什么，但输出格式总不对 | 缺格式约束 | 先试**提示词 + 结构化输出**；量大了才考虑微调固化 |
| 模型风格/口吻怎么调都不像品牌话术 | 缺稳定风格 | **微调**（把小样本风格练进权重） |
| 任务很垂直、术语多，通用模型老是理解偏 | 缺领域"手感" | **微调**（领域语料） |
| 任务简单（分类/抽取/路由）却用了大模型，太贵太慢 | 用错了模型 | **SLM 替代**（蒸馏或直接小模型 + 少样本） |
| 上下文塞不进、多轮撑爆 | 上下文工程问题 | **不归微调管**——用阶段 11 的四件套 |

> **记忆钩子**：**"不知道"归 RAG，"不听话"归提示词，"不像"归微调，"太贵"归 SLM。** 四句话就能拆掉 80% 的"要不要微调"的纠结。

### 3.2 微调的三类真实动机（按性价比排序）

1. **格式/风格固化**（性价比最高）：把"必须输出这个 JSON 结构""必须用这种口吻"从"每次靠 prompt 叮嘱"变成"权重里自带的习惯"。**样本需求小**（几百条）、收益直接可测。
2. **SLM 替代大模型**（成本收益最直观）：把"分类/抽取/路由"这类**窄任务**从大模型迁到 1.5B–7B 小模型。延迟与成本降一个量级，**前提是先证明小模型够用**。
3. **领域能力**（最贵、见效最慢）：法律、医疗、金融的术语与推理模式。**样本需求大、标注贵、评测难**，且经常 RAG 就能覆盖大部分需求——**这是最容易过度投入的一类**。

### 3.3 LoRA / QLoRA：改 1% 的参数，拿到 80% 的效果

**LoRA 一句话**：冻结原模型全部权重，只在注意力层的若干投影矩阵旁边挂一对**小矩阵**（低秩分解，即用两个小矩阵近似一个大矩阵），只训练这一小撮参数。

| 参数 | 作用 | 经验起始值 |
|------|------|-----------|
| `r`（秩） | 小矩阵的"宽度"——越大容量越强、越易过拟合、显存越大 | 8–32（窄任务 8–16 就够） |
| `lora_alpha` | 缩放系数；常设为 `r` 的 1–2 倍 | 16–64 |
| `target_modules` | 把 LoRA 挂在哪些层上 | 注意力投影（`q_proj`/`k_proj`/`v_proj`/`o_proj`） |
| `lora_dropout` | 防过拟合 | 0.05–0.1 |
| 学习率 | 通常比全量微调大得多 | 1e-4 ~ 3e-4 |

**QLoRA 一句话**：把基座模型先**量化到 4bit** 加载（`load_in_4bit`），再在它上面做 LoRA。**结果：单张 24GB 卡就能微调 7B 模型**——这是 QLoRA 真正改变行业的地方。

> **记忆钩子**：LoRA 像**在原书上贴便利贴**——书本身一个字不改（原权重冻结），只加一叠小纸条（低秩矩阵）；换任务时把纸条撕掉换一批就行。QLoRA 像**把原书先做成缩印版**再贴便利贴——更省地方，代价是底本略有损耗。

### 3.4 数据：质量 > 数量，这条没有例外

- **规模**：格式/风格类任务 **几百条**即可（200–1000）；领域能力类通常要**数千条以上**。
- **质量**：100 条干净、一致、人工核对过的样本 > 10000 条爬来的噪声。**标注不一致是微调失败的第一大原因**——同一类输入，标注标准必须唯一。
- **格式**：聊天式用 `messages`（`system`/`user`/`assistant`），指令式用 `instruction`/`input`/`output`；两者都能用，**关键是训练和推理时的模板必须一致**。
- **切分**：留出 **10–20% 验证集**，且必须与训练集**同分布但不重叠**。别用训练集上的 loss 判断好坏。
- **评测集要独立存在**：微调前先在**固定评测集**上测基线（阶段 10 的那套），微调后测同一个集合——**没有前后对比的微调等于没做**。

### 3.5 SLM 路由：让小模型干小活，大模型兜底

阶段 19 讲了"模型分级"的原则，这里把它工程化成一个可部署的模式（SLM：小语言模型，即前面说的 1.5B–7B 档）：

```
用户输入 → 小模型（分类/抽取/路由 + 置信度）→ 置信度够 → 直接处理（便宜、快）
                                            ↘ 置信度不够 → 交给大模型（贵、准）
```

**三个要点**：

1. **小模型只做"窄任务"**：分类到 N 个类别、从文本里抽 `{金额, 日期, 单号}`——**别让它写长文**；
2. **必须输出置信度**（或用 logprob 代理，logprob 是模型对输出给出的对数概率），作为"要不要兜底"的开关；
3. **兜底比例要监控**：如果 80% 都兜底了，说明小模型白训了——**兜底率是这条路线的核心健康指标**。

> **记忆钩子**：SLM 路由像**分诊台**——护士（小模型）按症状分流，拿不准就叫医生（大模型）。**分诊台的错误率可以容忍，但"拿不准却硬分"不行**——所以必须留兜底通道。

### 3.6 蒸馏：用大模型"教"小模型

**思路**：让大模型（或强模型 + 复杂 prompt）对**大量无标注文本**生成结果，人工/规则筛选后作为训练数据，去训一个小模型。常用于：

- 大模型输出的**分类标签** → 训分类小模型；
- 大模型的**结构化抽取结果** → 训抽取小模型；
- 大模型的**推理过程** → 训小模型模仿（更进阶，收益不稳定）。

**红线**：蒸馏数据的质量**完全取决于老师模型 + 筛选规则**。老师错了，学生会**更自信地错**。所以筛选环节（人工抽查、规则校验、一致性检查）不能省。

## 4. 动手教程

### 步骤 1：先建基线（不做这一步，后面全是自嗨）

```python
# baseline.py —— 在任何微调之前，先量出"不微调能拿多少分"
from langchain.chat_models import init_chat_model

model = init_chat_model("openai:gpt-4o-mini", temperature=0)

EVAL_CASES = [
    {"text": "我的包裹三天没动了", "label": "物流"},
    {"text": "申请退款，商品破损", "label": "退款"},
    # ...从阶段 10 的评测集里取 30–50 条
]

hit = 0
for case in EVAL_CASES:
    pred = model.invoke(
        f"把这条客服工单归到「退款/物流/账号/其他」之一，只输出类别名：\n{case['text']}"
    ).content.strip()
    hit += pred == case["label"]

print(f"基线准确率：{hit / len(EVAL_CASES):.1%}")
```

**这条基线数字，是你未来所有决策的锚点**。它同时也回答了最关键的问题：**这个任务真的需要微调吗？**（如果基线已经 95%，微调就是浪费。）

### 步骤 2：准备数据集（JSONL）

```jsonl
{"messages": [{"role": "system", "content": "你是客服工单分类器，只输出类别名。"}, {"role": "user", "content": "我的包裹三天没动了"}, {"role": "assistant", "content": "物流"}]}
{"messages": [{"role": "system", "content": "你是客服工单分类器，只输出类别名。"}, {"role": "user", "content": "申请退款，商品破损"}, {"role": "assistant", "content": "退款"}]}
```

```python
# prepare_data.py —— 切分 + 校验（比训练脚本更重要的 20 行）
import json
import random

rows = [json.loads(line) for line in open("data/tickets.jsonl", encoding="utf-8")]

# ① 一致性校验：同一输入不能有两个不同标签
seen: dict[str, str] = {}
for r in rows:
    key = r["messages"][1]["content"]
    label = r["messages"][-1]["content"]
    if key in seen and seen[key] != label:
        raise ValueError(f"标注冲突：{key!r} → {seen[key]} vs {label}")
    seen[key] = label

# ② 切分
random.seed(42)
random.shuffle(rows)
n_val = max(1, int(len(rows) * 0.15))
val, train = rows[:n_val], rows[n_val:]

for name, data in (("train", train), ("val", val)):
    with open(f"data/{name}.jsonl", "w", encoding="utf-8") as f:
        for r in data:
            f.write(json.dumps(r, ensure_ascii=False) + "\n")

print(f"训练 {len(train)} 条 / 验证 {len(val)} 条 / 类别分布：",
      {lbl: sum(1 for r in rows if r['messages'][-1]['content'] == lbl) for lbl in {r['messages'][-1]['content'] for r in rows}})
```

**先看类别分布**：如果某一类只占 2%，模型很可能永远不预测它——需要补样本或做重采样（把少数类样本多抽一些）。

### 步骤 3：LoRA / QLoRA 训练骨架

```bash
pip install torch transformers peft trl datasets bitsandbytes accelerate
```

```python
# train_lora.py —— QLoRA 微调骨架（单卡可跑 7B）
import torch
from datasets import load_dataset
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from trl import SFTConfig, SFTTrainer

MODEL_ID = "Qwen/Qwen2.5-7B-Instruct"

# ① 4bit 量化加载（QLoRA 的关键；只做 LoRA 不量化就把 bnb 配置去掉）
bnb = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,
)

tokenizer = AutoTokenizer.from_pretrained(MODEL_ID)
model = AutoModelForCausalLM.from_pretrained(
    MODEL_ID, quantization_config=bnb, device_map="auto",
)
model = prepare_model_for_kbit_training(model)          # 让量化模型能接 LoRA 训练

# ② LoRA 配置：窄任务 r 取小一点更稳
lora = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    bias="none", task_type="CAUSAL_LM",
)
model = get_peft_model(model, lora)
model.print_trainable_parameters()                      # 你会看到可训练参数只有 ~1%

# ③ 训练
trainer = SFTTrainer(
    model=model,
    train_dataset=load_dataset("json", data_files="data/train.jsonl", split="train"),
    eval_dataset=load_dataset("json", data_files="data/val.jsonl", split="train"),
    args=SFTConfig(
        output_dir="out/lora-ticket",
        num_train_epochs=3,
        learning_rate=2e-4,
        per_device_train_batch_size=1,
        gradient_accumulation_steps=8,                  # 小显存靠梯度累积凑等效 batch
        logging_steps=10,
        eval_strategy="epoch",
        save_strategy="epoch",
        report_to=[],
    ),
)

trainer.train()
trainer.save_model("out/lora-ticket")                   # 只存 LoRA 适配器（几十 MB）
```

> **版本提示**：TRL 在 1.x 前后有参数改名（如序列长度上限由 `max_seq_length` 改为 `max_length`）。**照抄任何博客前先确认你自己装的版本**——这跟阶段 04/15 里"导入路径随版本变"是同一类问题。

### 步骤 4：训练后必须做同一份评测

```python
# eval_after.py —— 用步骤 1 的同一份评测集，测微调后的模型
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer

base = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-7B-Instruct")
model = PeftModel.from_pretrained(base, "out/lora-ticket").merge_and_unload()   # 合并适配器
tok = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-7B-Instruct")

# 评测要跟基线口径完全一致：贪心解码（do_sample=False），否则分数差异里混着采样噪声
def classify(text: str) -> str:
    msgs = [{"role": "system", "content": "你是客服工单分类器，只输出类别名。"},
            {"role": "user", "content": text}]
    ids = tok.apply_chat_template(msgs, add_generation_prompt=True, return_tensors="pt")
    ids = ids.to(model.device)          # 若加载时用了 device_map="auto"，输入张量仍在 CPU，必须搬过去
    out = model.generate(ids, max_new_tokens=8, do_sample=False)
    return tok.decode(out[0][ids.shape[-1]:], skip_special_tokens=True).strip()
```

> **两个必踩的坑**：`ids.to(model.device)` —— 用 `device_map="auto"` 加载时**权重在 GPU 而输入张量还在 CPU**，不搬就抛 `Expected all tensors to be on the same device`；`do_sample=False` —— 微调前后两次评测如果一个采样一个贪心，分数差异里混着随机性，**你根本分不清是微调有效还是运气好**。

**判据**（三者缺一不可）：

1. 同评测集准确率**是否真的高于基线**（哪怕只高 2 个点也要看到）；
2. **是否出现"灾难性偏移"**——把原来会做的题做坏了（**只看目标指标会漏掉这个**）；
3. 延迟/成本是否降到能覆盖训练成本（**算回本周期**）。

### 步骤 5：合并权重 + 部署（接前瞻篇 31）

```python
# 合并成完整权重，才能被 vLLM / SGLang 直接加载
# 注意：步骤 4 的 model 已 merge_and_unload() 过，返回的是已合并权重的 base model（不再是 PeftModel），
# 这里从原始 base + 适配器重新合并，保证本步骤可独立运行：
base = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-7B-Instruct")
merged = PeftModel.from_pretrained(base, "out/lora-ticket").merge_and_unload()
merged.save_pretrained("out/merged-ticket")
tok = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-7B-Instruct")
tok.save_pretrained("out/merged-ticket")
```

```bash
# 用你自己的权重起服务（与前一篇的 vLLM 完全一致，只是把 --model 换成本地路径）
vllm serve /models/merged-ticket \
  --max-model-len 4096 \
  --enable-auto-tool-choice --tool-call-parser hermes
```

> **注意**：vLLM **不能直接吃 LoRA 适配器目录**，要用 `merge_and_unload()` 合并后的**完整权重**（或使用 vLLM 的 LoRA 加载能力，需额外配置）。合并后模型体积 = 原模型大小，**别指望省存储**。

### 步骤 6：SLM 路由（不用微调也可能够用，先试这个）

**顺序很重要**：先用"小模型 + 少样本 + 结构化输出"试；不行**再**微调。很多时候第 6 步就够，根本不用走到前面几步。

```python
# router.py —— 小模型做路由 + 置信度兜底
import os

from langchain.chat_models import init_chat_model
from pydantic import BaseModel, Field


class Route(BaseModel):
    category: str = Field(description="类别：退款 / 物流 / 账号 / 其他")
    confidence: float = Field(description="0~1 的置信度；不确定就给低分")


small = init_chat_model(
    "openai:Qwen2.5-1.5B-Instruct",
    base_url=os.environ.get("INFERENCE_BASE_URL", "http://localhost:8000/v1"),
    api_key="EMPTY", temperature=0,
).with_structured_output(Route)

big = init_chat_model("openai:gpt-4o-mini", temperature=0)


def route(text: str) -> tuple[str, str]:
    """返回 (类别, 由谁判定)。conf 低于阈值就交给大模型兜底。"""
    r = small.invoke(f"给这条工单分类：{text}")
    if r.confidence >= 0.7:
        return r.category, "slm"
    return big.invoke(f"把工单归到 退款/物流/账号/其他 之一，只输出类别名：{text}").content.strip(), "llm"
```

**要监控的指标**：`由谁判定` 的比例。**兜底率长期高于 30%** 就说明小模型该重训或该换——这正是步骤 1–5 存在的意义。

### 步骤 7：蒸馏造数据（没有标注数据时的正路）

```python
# distill.py —— 用强模型造标注，再筛一遍
import json
from langchain.chat_models import init_chat_model

teacher = init_chat_model("openai:gpt-4o", temperature=0)

raw_texts = [line.strip() for line in open("data/unlabeled_tickets.txt", encoding="utf-8") if line.strip()]

out = []
for text in raw_texts:
    label = teacher.invoke(
        f"把这条工单归到「退款/物流/账号/其他」之一，只输出类别名：\n{text}"
    ).content.strip()
    if label not in {"退款", "物流", "账号", "其他"}:
        continue                                        # ① 规则校验：非法标签直接丢
    out.append({"messages": [
        {"role": "system", "content": "你是客服工单分类器，只输出类别名。"},
        {"role": "user", "content": text},
        {"role": "assistant", "content": label},
    ]})

with open("data/tickets_distilled.jsonl", "w", encoding="utf-8") as f:
    for r in out:
        f.write(json.dumps(r, ensure_ascii=False) + "\n")

print(f"可用样本 {len(out)} / 原始 {len(raw_texts)}")   # ② 看丢弃率：太高说明提示或规则有问题
```

**③ 人工抽查 50–100 条**——这一步绝不能省。蒸馏的本质是"把老师模型的水平（和它的错误）复制给学生"。

## 5. 完整示例

"客服工单自动分流"的完整落地方案（不训练也能上线版）：

```python
# ticket_triage.py
import os

from langchain.chat_models import init_chat_model
from pydantic import BaseModel, Field

INFERENCE_BASE_URL = os.environ.get("INFERENCE_BASE_URL", "http://localhost:8000/v1")

class Route(BaseModel):
    category: str = Field(description="退款 / 物流 / 账号 / 其他")
    confidence: float = Field(description="0~1 置信度")


small = init_chat_model("openai:Qwen2.5-1.5B-Instruct", base_url=INFERENCE_BASE_URL,
                        api_key="EMPTY", temperature=0).with_structured_output(Route)
big = init_chat_model("openai:gpt-4o-mini", temperature=0)

ESCALATE = ("账号",)          # 涉及账号安全的类别一律交人工/大模型复核
CONF_THRESHOLD = 0.7


def triage(text: str) -> dict:
    r = small.invoke(f"给这条工单分类：{text}")
    if r.confidence >= CONF_THRESHOLD and r.category not in ESCALATE:
        return {"category": r.category, "by": "slm", "confidence": r.confidence}
    answer = big.invoke(
        f"把工单归到 退款/物流/账号/其他 之一，只输出类别名：\n{text}"
    ).content.strip()
    return {"category": answer, "by": "llm", "confidence": r.confidence}


if __name__ == "__main__":
    stats = {"slm": 0, "llm": 0}
    for t in ["包裹三天没动", "商品破损要退款", "忘了密码登不上", "想问下发票怎么开"]:
        res = triage(t)
        stats[res["by"]] += 1
        print(f"{t} → {res['category']}（{res['by']}, conf={res['confidence']:.2f})")
    print("兜底率：", stats["llm"] / sum(stats.values()))
```

**这个例子的三个设计点**：

1. **`ESCALATE` 白名单**：账号类业务风险高，**就算置信度高也不交给小模型**——分级不是"一律以小博大"，而是"按风险分级"；
2. **输出 `by` 字段**：让"兜底率"变成可监控指标（上线后接到阶段 26 的看板）；
3. **失败模式是可控的**：小模型判断不了就退到大模型，**不会因为路由错误而把工单扔掉**。

## 6. 练习

**基础**：用步骤 1 的基线脚本，在你的业务文本上量出"不微调"的准确率，并记录三个数字：准确率、平均延迟、单次成本。

**进阶**：用步骤 7 的蒸馏脚本，让强模型给 500 条未标注文本打标签，人工抽查 50 条，统计"老师正确率"，再决定这批数据能不能用来训学生。

**挑战**：完整跑一遍步骤 2→4：用 300 条样本做一次 QLoRA 微调，用**同一份评测集**对比基线。

写出结论报告，必须包含：① 准确率变化；② 是否出现"原来会的题变不会"（灾难性偏移）；③ 回本周期估算（省下的推理成本多久能覆盖训练与运维成本）。如果结论是"不该微调"，**这也是一份合格的报告**。

<details>
<summary>参考答案要点</summary>

- 基础：典型结论是"简单分类任务上，`gpt-4o-mini` + 清晰提示就能拿到 85–95%"——**这个数字直接决定后面该不该微调**。把延迟与成本也记下来，它们是后面算回本周期的分母。
- 进阶：抽查重点看**类别边界**（"物流" vs "退款" 的模糊句子）。若老师正确率 < 90%，先修提示或修标注规则，**别拿它去教学生**——学生会更自信地继承错误。
- 挑战：合格报告的判据是**三条都答**，而不是"准确率涨了就成功"。常见结果：准确率涨 2–5 个点，但若基线已很高，微调的边际收益抵不过训练与运维投入——此时正确决策是**用 SLM 路由 + 兜底**（步骤 6），把成本降下来，而不训模型。要点：**"不微调"是常见且正确的结论**。
</details>

## 7. 自测清单

- [ ] 能用一句话区分"该 RAG / 该提示 / 该微调 / 该换 SLM"
- [ ] 说得出微调的三类动机，以及哪一类性价比最高、哪一类最容易过度投入
- [ ] 能解释 LoRA 与 QLoRA 的区别，并说出 `r` 调大调小的影响
- [ ] 知道数据"质量 > 数量"，且标注一致性是首要问题
- [ ] 知道微调前必须建基线、微调后必须用**同一份评测集**对比
- [ ] 会用"小模型 + 置信度 + 大模型兜底"做路由，并知道要监控兜底率
- [ ] 知道蒸馏的产出质量完全取决于老师模型与筛选环节

## 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 微调后模型变"傻"了 | 数据脏/量太少/训练过久（过拟合） | 查标注一致性、加验证集、减 epoch、降 `r` |
| 微调后**其他能力**也退化了 | 灾难性偏移（只训了单任务） | 混入一部分通用数据；评估时**必须测非目标任务** |
| loss 很低但上线很烂 | 用训练集 loss 当判据 | 只看**独立评测集**的指标 |
| 输出格式还是不对 | 训练模板与推理模板不一致 | 训练/推理用**同一个 chat template** |
| 不知道微调有没有用 | 没有基线 | 微调前先测基线（步骤 1） |
| 换了新知识，微调完还是不知道 | 想用微调补知识 | 那是 RAG 的活；微调改能力不改事实 |
| 小模型路由兜底率奇高 | 小模型其实不擅长这个任务 | 先试少样本 + 结构化输出；仍不行再微调，或换个 7B 档 |
| LoRA 适配器直接丢给 vLLM 加载失败 | vLLM 默认读完整权重 | `merge_and_unload()` 合并后再部署 |
| 照抄博客训练脚本报参数错 | TRL/transformers 版本差异 | 先确认版本，必要时查该版本的文档 |
| 蒸馏数据训出来的模型错得更自信 | 老师模型本身就错，筛选环节缺失 | 规则校验 + 人工抽查，老师正确率不达标就别训 |

## 9. 延伸

- 阶段 10：评测（微调的前置与验收，没有它就没有基线）
- 阶段 19：模型分级（SLM 路由是它的工程化落地）
- 前瞻篇 31：自托管（微调权重最终的部署形态）
- 工具链：PEFT（LoRA）、TRL（SFTTrainer）、bitsandbytes（QLoRA）、Unsloth（提速）
- 补充篇 26：把"兜底率 / 小模型准确率 / 单次成本"做成线上指标

## 10. 记忆强化

**口诀**：

> 不知道归 RAG，不听话归提示，不像归微调，太贵归 SLM；
> 先建基线再动手，同一评测比前后；
> 冻结原权重、只挂小矩阵，LoRA 只训百分之一；
> 四比特加载加 LoRA，单卡也能训七 B；
> 质量胜过数量，标注不一致全白费。

**5 分钟回顾闪卡**（先默答，再展开核对）：

1. 问：怎么判断一个需求该用 RAG、改提示词，还是微调？
   <details><summary>点击看答案</summary>看"缺什么"：**缺知识**（不知道你们的规定/新数据）→ RAG；**缺格式/风格约束**（知道但写不对）→ 先提示词 + 结构化输出，量大了再微调固化；**缺领域手感**（术语与推理模式对不上）→ 微调。一句话：**"不知道"归 RAG，"不听话"归提示，"不像"归微调**。</details>
2. 问：LoRA 和 QLoRA 的区别是什么？QLoRA 为什么重要？
   <details><summary>点击看答案</summary>LoRA 冻结原权重，只在注意力投影旁挂低秩小矩阵，只训这一小撮参数（约 1%）；**QLoRA = 把基座先 4bit 量化加载 + LoRA 训练**。QLoRA 重要在于它把"微调 7B"的硬件门槛降到了一张 24GB 消费级卡。</details>
3. 问：`r`（秩）调大调小分别意味着什么？
   <details><summary>点击看答案</summary>`r` 决定低秩矩阵的容量：调大 → 表达能力更强、显存与训练成本更高、**更容易过拟合**；调小 → 更稳、更省，但可能学不动复杂任务。窄任务（分类/抽取/风格）通常 8–16 就够，不必一上来就 64。</details>
4. 问：微调数据"质量 > 数量"具体指什么？
   <details><summary>点击看答案</summary>指三件事：①**标注一致性**——同一输入不能有两个标签（标注标准必须唯一，这是微调失败第一大原因）；②**规模够用即可**——格式/风格类几百条就能起效，领域类才需要数千条；③**100 条干净核对过的样本胜过 10000 条爬来的噪声**。</details>
5. 问：微调后怎么证明它真的变好了？
   <details><summary>点击看答案</summary>三条：①用**微调前建立的同一份评测集**测前后对比（哪怕只涨 2 个点也要看到）；②检查**有没有灾难性偏移**（原来会的题被做坏——只看目标指标会漏掉）；③算**回本周期**（省下的推理成本多久覆盖训练与运维成本）。缺任何一条都不能算"变好"。</details>
6. 问：SLM 路由为什么一定要有兜底？要监控什么？
   <details><summary>点击看答案</summary>因为小模型在窄任务上够用、但边界情况会判错，而路由错会导致工单被错误处理。做法是让 SLM 输出**置信度**，低于阈值就交给大模型。要监控的核心指标是**兜底率**——长期高于 30% 说明小模型该重训或该换；另外高风险类别（如账号）应**强制走大模型/人工**，而不是只看置信度。</details>
7. 问：蒸馏的数据能直接用吗？红线在哪？
   <details><summary>点击看答案</summary>不能直接用。蒸馏 = 让强模型给未标注数据打标签，再拿去训小模型。**红线是筛选环节不能省**：规则校验（丢掉非法标签）+ 人工抽查（老师正确率不达标就别训）。老师错了，学生会**更自信地错**——你只是把老师的错误固化进了权重。</details>

**费曼任务**：

- 用大白话向一位"完全不懂技术的同事"讲清【为什么"模型答得不对"有时该查资料库（RAG）、有时该改提示词、有时才该花钱微调】，限时 2 分钟（建议录音/对着镜子讲）。哪里卡住、哪里要回头翻书，那个点就是你还没真正懂的点——回到对应小节重看后再讲一遍。

---

上一阶段：[31 私有化部署与自托管推理](31-前瞻-私有化部署与自托管推理.md) ｜ 返回[总览与学习路线](00-总览与学习路线.md) ｜ 下一阶段：[20 附录-速查与面试追问](20-附录-速查与面试追问.md) ｜ 前瞻篇导航：[28 LangGraph 进阶](28-补充-LangGraph进阶.md)
