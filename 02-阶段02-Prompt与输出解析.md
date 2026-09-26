# 阶段 02：Prompt 模板与输出解析

> 定位：让输入可复用、让输出可程序化处理 ｜ 难度 ★☆☆ ｜ 预计 3 小时

> **【一句话记住】**：模板填槽位，解析器收快递，`{{` 转义防误会。
>
> **【生活类比】**：PromptTemplate 像一份填空试卷——`{product}` 是空格，发出去前把空格填好；输出解析器像快递驿站——模型吐出来的是一整袋没分类的文本，驿站按你给的 Pydantic 清单把每件货贴上标签（类型校验）再交给你。`Field(description)` 不是给人看的注释，而是贴给模型的"取件须知"，写清楚模型才不会拿错货。

## 1. 学习目标

- [ ] 用 `PromptTemplate` 做单段文本模板，并正确转义字面花括号
- [ ] 用 `ChatPromptTemplate` 组装多角色消息列表
- [ ] 用 `PydanticOutputParser` 把模型输出解析成带校验的 Python 对象
- [ ] 知道生产环境更推荐 `with_structured_output`，并说清两者差异

## 2. 前置知识

- 阶段 01（ChatModel 与消息类型）
- Python 类型注解基础

## 3. 核心概念

### 3.1 PromptTemplate vs ChatPromptTemplate

| | 输入 | 输出 | 适用 |
|---|---|------|------|
| `PromptTemplate` | 变量字典 | **字符串** | 无角色区分的简单任务：翻译、摘要、文案 |
| `ChatPromptTemplate` | 变量字典 | **消息列表** | 需要 system/human/ai 角色区分的对话 |

模板语法都是 `{变量名}`。`format()` 或 `invoke()` 时替换。

> **转义规则**：模板里若要输出字面 JSON 或花括号，必须写成 `{{ }}`，否则会被当成变量。

> **记忆钩子**：`{x}` 是"要填的空"，`{{x}}` 是"印上去的字"——就像合同里要写"人民币"三个字时，你不会把它当成待填空格。想让大括号原样出现，就把它"双层保护"成 `{{ }}`。

### 3.2 输出解析器的作用

模型输出本质是**文本**。解析器负责把它变成**带类型校验的对象**：

1. 用 Pydantic（Python 的数据校验库）定义目标结构（字段名 + 类型 + `Field(description=...)`）
2. `parser.get_format_instructions()` 生成格式说明，注入 prompt
3. 把模型输出交给 `parser.parse()`（或串进 LCEL 管道）

`Field(description=...)` 不是注释——它会被写进给模型看的 schema（数据结构定义）里，直接影响抽取质量。

> **记忆钩子**：别把 `Field(description="...")` 当成写给同事看的代码注释——它其实是模型"看得见"的试卷题目说明。你写"价格"，模型可能给你带单位的字符串；你写"价格，数字，单位人民币"，它才乖乖吐浮点数。

## 4. 动手教程

### 步骤 1：PromptTemplate 单段模板

```python
from langchain_core.prompts import PromptTemplate

prompt = PromptTemplate(
    template="为{product}写一句广告语，目标用户是{audience}，不超过 20 字。",
    input_variables=["product", "audience"],
)
print(prompt.format(product="智能手表", audience="健身爱好者"))
# -> 为智能手表写一句广告语，目标用户是健身爱好者，不超过 20 字。
```

### 步骤 2：字面花括号必须转义

```python
# 想输出包含花括号的字面文本（如 JSON），必须双写转义。
bad = PromptTemplate(template='输出 JSON：{"price": 100}')     # 单花括号被当成变量占位符
bad.format()
# -> KeyError: '"price"'   模板去找名叫 "price"（含引号）的变量，你没传

good = PromptTemplate(template='输出 JSON：{{"price": 100}}')   # 双层花括号 = 字面花括号
good.format()
# -> 输出 JSON：{"price": 100}
```

### 步骤 3：ChatPromptTemplate 多角色

*（沿用上文阶段 01 中已定义的 `model`）*

```python
from langchain_core.prompts import ChatPromptTemplate

chat_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是资深营销文案，只输出广告语本身，不要解释。"),
    ("human",  "为{product}写一句广告语，面向{audience}。"),
])

messages = chat_prompt.invoke({"product": "智能手表", "audience": "健身爱好者"})
print(messages.to_messages())     # [SystemMessage, HumanMessage]
print(model.invoke(messages).content)
```

### 步骤 4：定义 Pydantic 输出结构

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(description="商品名称")
    price: float = Field(description="价格，数字，单位人民币")
    rating: float = Field(description="评分，0 到 5")
```

### 步骤 5：创建解析器并注入格式说明

```python
from langchain_core.output_parsers import PydanticOutputParser

parser = PydanticOutputParser(pydantic_object=Product)

prompt = ChatPromptTemplate.from_messages([
    ("system", "按以下格式输出 JSON：\n{format_instructions}"),
    ("human",  "介绍一款{product}，输出它的信息。"),
]).partial(format_instructions=parser.get_format_instructions())
```

`.partial()` 用于**预先固定**某些变量，之后调用时只需传剩下的。

### 步骤 6：串成链并解析

*（沿用上文步骤 5 已定义的 `prompt` 与 `parser`，以及步骤 3 的 `model`）*

```python
chain = prompt | model | parser          # LCEL 管道（阶段 05 详解）
item: Product = chain.invoke({"product": "智能手表"})
print(item.name, item.price, item.rating)   # 已是对象，可直接取属性
```

### 步骤 7：生产写法 —— with_structured_output

*（沿用上文步骤 4 已定义的 `Product` 与步骤 3 的 `model`）*

```python
structured = model.with_structured_output(Product)
item = structured.invoke("介绍一款智能手表")   # 无需手写 format_instructions
```

## 5. 完整示例

```python
from pydantic import BaseModel, Field
from langchain.chat_models import init_chat_model
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import PydanticOutputParser

class Product(BaseModel):
    name: str = Field(description="商品名称")
    price: float = Field(description="价格，数字，单位人民币")
    rating: float = Field(description="评分，0 到 5")
    features: list[str] = Field(description="三个核心卖点")

parser = PydanticOutputParser(pydantic_object=Product)
model = init_chat_model("openai:gpt-4o", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是商品信息抽取助手。只输出 JSON，不要额外文字。\n{format_instructions}"),
    ("human",  "抽取商品信息：{product}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | model | parser
item = chain.invoke({"product": "Apple Watch Series 10"})
print(type(item).__name__, item.price, item.features)
```

## 6. 练习

**基础**：写一个 `PromptTemplate`，含 `{lang}` 和 `{code}` 两个变量，让模型把代码翻译成指定语言并解释。

**进阶**：把阶段 02 的 `Product` 扩展为嵌套结构：`Product` 里含 `reviews: list[Review]`，`Review` 含 `user`、`score`、`comment`。用 `PydanticOutputParser` 解析出来。

**挑战**：同一个抽取任务，分别用 `PydanticOutputParser` 和 `with_structured_output` 各跑 10 次，统计解析成功率与平均 token 消耗，写成对比结论。

<details>
<summary>参考答案要点</summary>

- 基础：注意 `{code}` 里若含花括号需转义；`input_variables` 要写全。
- 进阶：嵌套模型一样能生成 JSON Schema；失败时先检查 `Field(description)` 是否清楚。
- 挑战：`with_structured_output` 由 API 侧约束输出，成功率明显更高且无需 `format_instructions`（省 token）；`PydanticOutputParser` 兼容不支持结构化输出的模型。
</details>

## 7. 自测清单

- [ ] 能独立写出含两个变量的 `PromptTemplate` 并 `format`
- [ ] 能说出 `.partial()` 的用途
- [ ] 知道字面花括号要写成 `{{ }}`
- [ ] 能定义 Pydantic 模型并接入 `PydanticOutputParser`
- [ ] 忘记注入 `format_instructions` 会怎样？能答上来（输出自由文本 → 解析失败）

## 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| `KeyError: 'xxx'` / 模板不替换 | 变量名拼写不一致或漏写 `input_variables` | 核对拼写，或用 `PromptTemplate.from_template` 自动推断 |
| 解析报 `OutputParserException` | 忘了注入 `format_instructions` | `.partial(format_instructions=parser.get_format_instructions())` |
| 模型输出带着 ```json 围栏 | 提示词没强调"只输出 JSON" | system 里加"不要 Markdown 围栏"，或换 `with_structured_output` |
| 字段类型错（`"12.5"` 字符串） | 模型没理解类型 | 在 `Field(description=...)` 里写明"数字类型，不要带单位" |
| JSON 里的大括号被当变量 | 未转义 | 写成 `{{ }}` |

## 9. 延伸

- `JsonOutputParser`：只要 dict，不要 Pydantic 校验时的轻量选择
- `OutputFixingParser`：解析失败时让模型自动修复（多一次调用，成本换稳定性）。1.x 起该类位于 `langchain_classic.output_parsers`

## 10. 记忆强化

**口诀**：

> 模板填槽位，消息分角色；花括号要字面，双层才安全；格式说明忘注入，解析一定报错；生产首选结构化输出。

**5 分钟回顾闪卡**（先默答，再展开核对）：

1. 问：`PromptTemplate` 和 `ChatPromptTemplate` 的输出分别是什么？
   <details><summary>点击看答案</summary>PromptTemplate 输出一个字符串，适合无角色区分的翻译/摘要/文案；ChatPromptTemplate 输出消息列表（SystemMessage + HumanMessage 等），适合需要角色区分的对话。</details>
2. 问：模板里要输出字面的 `{"price": ...}` 该怎么写？
   <details><summary>点击看答案</summary>字面花括号要双层写成 `{{"price": "{product}"}}`——键名的花括号是字面（双写转义），变量槽位 `{product}` 单写。若单写键名花括号，会被当成变量占位符而报错。</details>
3. 问：`Field(description=...)` 是写给谁看的？
   <details><summary>点击看答案</summary>不是给程序员看的注释，它会被写进给模型的 JSON Schema 里，直接影响模型抽取字段的质量。描述越清楚，输出越准。</details>
4. 问：忘记注入 `format_instructions` 会发生什么？
   <details><summary>点击看答案</summary>模型不知道要按什么格式输出，会吐自由文本，PydanticOutputParser.parse() 时抛 OutputParserException。要靠 `.partial(format_instructions=parser.get_format_instructions())` 注入。</details>
5. 问：`.partial()` 的作用是什么？
   <details><summary>点击看答案</summary>预先固定模板里的某些变量（如 format_instructions），之后 invoke 时只需传剩下的变量，避免每次重复传。</details>
6. 问：生产环境为什么更推荐 `with_structured_output` 而不是 `PydanticOutputParser`？
   <details><summary>点击看答案</summary>with_structured_output 由 API 侧原生约束输出格式，成功率更高，且无需手写 format_instructions 省 token；PydanticOutputParser 则兼容不支持结构化输出的模型。</details>

**费曼任务**：

- 用大白话向一位"完全不懂编程的朋友"讲清【模板填槽 + 解析器收快递】的完整流程，限时 2 分钟（建议录音/对着镜子讲）。哪里卡住、哪里要回头翻书，那个点就是你还没真正懂的点——回到对应小节重看后再讲一遍。

---

上一阶段：[01 阶段01-ChatModel](01-阶段01-ChatModel.md) ｜ 返回[总览与学习路线](00-总览与学习路线.md) ｜ 下一阶段：[03 阶段03-Tool工具系统](03-阶段03-Tool工具系统.md)
