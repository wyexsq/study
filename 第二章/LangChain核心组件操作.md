# LangChain核心组件操作

## 问题：不同厂商的调用接口不一样，不可能调用一次就要换一个接口

# 模型调用（ChatOpenAI）：统一接口适配不同大模型

在LangChain中，大模型主要分为两类，我们需要根据场景选择使用：

（1）LLM（文本生成模型）：接收一段文本，返回一段文本（一问一答）。适合简单的文本生成、翻译、总结等场景。

（2）ChatModel（对话模型）：接收一系列\&\#34;对话消息\&\#34;（比如用户消息、助手回复）陪你唠嗑，返回一条对话消息。适合多轮对话场景——因为它能更好地理解对话上下文的逻辑。

```python
# 导入必要的模块
from langchain_openai import ChatOpenAI  # OpenAI对话模型的统一接口
from dotenv import load_dotenv
from pydantic import SecretStr
import os

# 加载API密钥（和上一章一样，从.env文件读取）
load_dotenv()

API_KEY = os.getenv("API_KEY")
BASE_URL = os.getenv("BASE_URL")  # 从环境变量读取，未配置时默认为None（使用OpenAI官方地址）

if not API_KEY:
    raise ValueError("未检测到 API_KEY，请检查 .env 文件是否配置正确")

# 1. 初始化对话模型
# 不管是哪个厂商的ChatModel，初始化参数都类似（model、temperature等）
chat_model = ChatOpenAI(
    api_key=SecretStr(API_KEY),  # 使用SecretStr保护API密钥
    base_url=BASE_URL,
    model="qwen-turbo",  # 选择对话模型
    temperature=0.3,        # 随机性：0-1，越小越严谨，越大越有创造力
)

# 2. 构造对话消息
# ChatModel需要接收的是“消息列表”，每个消息有角色（user/assistant/system）和内容
messages = [
    # system消息：给助手设定身份和行为准则，会影响后续所有回复
    {"role": "system", "content": "你是一个耐心的AI学习助手，回复简洁易懂，适合高校学生理解。"},
    # user消息：用户的问题
    {"role": "user", "content": "请用3句话解释什么是LangChain？"}
]

# 3. 调用模型生成结果
# 统一调用方法：invoke()，传入消息列表
result = chat_model.invoke(messages)

# 4. 输出结果
# 结果是一个ChatMessage对象，content属性是回复内容
print("ChatModel回复：")
print(result.content)
```

首先要理解system user assistant

system就是使用模型之前的一个先行的规则，无论后面的user怎么提问，这个都会先行，一个不可违反的规则，**一个约束**

user：**就是提问**，不过他不是什么都可以问，什么都可以问的到，他不能违反system。

assistant：我的理解是助教，模型会讲但是不会记住，靠助教记录，返回，不然每一次提问都是一个新的开始

```python
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv
import os
from pydantic import SecretStr

load_dotenv()

API_KEY = os.getenv("API_KEY")
BASE_URL = os.getenv("BASE_URL")

if not API_KEY:
    raise ValueError("未检测到 API_KEY，请检查 .env 文件是否配置正确")

chat_model = ChatOpenAI(
    api_key=SecretStr(API_KEY),
    base_url=BASE_URL,
    model="qwen-turbo",
    temperature=0.3,
)

# 初始化对话历史（包含 system 设定）
history = [
    {"role": "system", "content": "你是一个耐心的AI学习助手，回复简洁易懂，适合高校学生理解。"}
]

# 第一轮对话
history.append({"role": "user", "content": "请用3句话解释什么是囚徒效应？"})

result = chat_model.invoke(history)
print("【第一轮回复】：")
print(result.content)

# 将模型的回复添加到历史中（assistant 消息）
history.append({"role": "assistant", "content": str(result.content)})

# 第二轮对话
# 追问，模型需要上下文才能理解"它"
history.append({"role": "user", "content": "它的例子有哪些？"})

result = chat_model.invoke(history)
print("\n【第二轮回复】：")
print(result.content)

# 继续记录
history.append({"role": "assistant", "content": str(result.content)})

# 第三轮对话
history.append({"role": "user", "content": "给我一个简单的使用场景"})

result = chat_model.invoke(history)
print("\n【第三轮回复】：")
print(result.content)
```

注意用from pydantic import SecretStr，转换key的类型，转成 SecretStr

# PromptTemplate：提示词模板

总不能每次都写一套提示词，其实有很多东西可以复用，可复用的叫固定参数，可以通过更改来提高使用效率的叫动态参数

```python
# 导入PromptTemplate
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv
from pydantic import SecretStr
import os

load_dotenv()
API_KEY = os.getenv("API_KEY")
BASE_URL = os.getenv("BASE_URL")  # 从环境变量读取，未配置时默认为None（使用OpenAI官方地址）

if not API_KEY:
    raise ValueError("未检测到 API_KEY，请检查 .env 文件是否配置正确")

chat_model = ChatOpenAI(
    api_key=SecretStr(API_KEY),
    base_url=BASE_URL,
    model="qwen-turbo",  # 选择对话模型
    temperature=0.3,        # 随机性：0-1，越小越严谨，越大越有创造力          # 最大生成 tokens 数，避免生成过长内容
)
# 1. 定义提示词模板
# input_variables：动态参数列表（这里是user_role和subject）
# template：提示词模板字符串，用{参数名}表示动态参数
prompt_template = PromptTemplate(
    input_variables=["user_role", "subject"],
    template="请给{user_role}写一段50字左右的{subject}学习建议，语言简洁实用，分2个小要点。"
)

# 2. 格式化模板（传入具体参数，生成完整提示词）
# 给“高校学生”生成“LangChain”学习建议
formatted_prompt = prompt_template.format(
    user_role="高校学生",
    subject="LangChain"
)
print("格式化后的提示词：")
print(formatted_prompt)

# 3. 调用模型生成结果
result = chat_model.invoke([{"role": "user", "content": formatted_prompt}])

print("\n生成的学习建议：")
print(result.content)
```

思考：既然可以动态调整来决定输出，如果我现在有一种需求我需要一种格式，可以可以也这样解决呢，答案是是可以的：我利用LangChain 的 FewShotPromptTemplate来达到这个目标

```python
# 导入必要的模板类
from langchain_core.prompts import FewShotPromptTemplate, PromptTemplate # FewShotPromptTemplate用于少样本学习，PromptTemplate用于定义示例模板
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv
from pydantic import SecretStr
import os

load_dotenv()
API_KEY = os.getenv("API_KEY")
BASE_URL = os.getenv("BASE_URL")  # 从环境变量读取，未配置时默认为None（使用OpenAI官方地址）

if not API_KEY:
    raise ValueError("未检测到 API_KEY，请检查 .env 文件是否配置正确")

chat_model = ChatOpenAI(
    api_key=SecretStr(API_KEY),
    base_url=BASE_URL,
    model="qwen-turbo",  # 选择对话模型
    temperature=0.3,        # 随机性：0-1，越小越严谨，越大越有创造力
)


# 1. 定义示例（少样本的核心：给模型看的参考案例）
examples = [
    {
        "subject": "Python编程",
        "method": "核心目标：掌握基础语法和常用库；学习步骤：1. 学习变量、函数等基础语法 2. 实操小项目（如计算器） 3. 学习Pandas、Matplotlib库；注意事项：多动手实操，遇到错误及时调试。"
    },
    {
        "subject": "机器学习",
        "method": "核心目标：理解基础算法原理和应用场景；学习步骤：1. 复习数学基础（线性代数、概率） 2. 学习经典算法（线性回归、决策树） 3. 用Scikit-learn实操；注意事项：先理解原理，再动手实现，避免死记硬背。"
    }
]

# 2. 定义示例模板（告诉模型如何解析示例）
example_template = """
学科：{subject}
学习方法：{method}
"""

# 定义示例提示词模板，告诉模型每个示例包含哪些动态参数（subject和method）以及格式
example_prompt = PromptTemplate(
    input_variables=["subject", "method"],
    template=example_template
)

# 3. 定义最终的提示词模板（包含示例和用户需求）
few_shot_prompt = FewShotPromptTemplate(
    examples=examples,                # 传入示例
    example_prompt=example_prompt,    # 示例模板
    suffix="学科：{new_subject}\n学习方法：",  # 详细解释：在示例之后，告诉模型用户要查询的新学科是什么，并要求生成学习方法
    input_variables=["new_subject"]   # 动态参数：用户要查询的新学科
)

# 4. 格式化模板（传入新学科：LangChain）
formatted_prompt = few_shot_prompt.format(new_subject="sql")
print("少样本提示词：")
print(formatted_prompt)

# 5. 调用模型生成结果
result = chat_model.invoke([{"role": "user", "content": formatted_prompt}])

print("\n生成的sql学习方法：")
print(result.content)
```

## 工程化实践：少样本提示模板的高效管理

刚刚那个代码，就是上面那个，我们直接将例子写入，这里有一个问题，在这里，我们只写了，一个类型例子，但是实际时候可能会有很多 **例子**，甚至需要动态示例选择、批量示例管理、性能优化。

这个过程中面临的痛点：

- 示例维护困难：将示例存储在文件（JSON/CSV）中，批量加载与更新

- 不同场景需要不同示例：按输入参数匹配示例类型，实现场景化示例分发

- 例子太多：使用ExampleSelector动态筛选相关示例，避免冗余

### 使用ExampleSelector的方法：

LangChain提供ExampleSelector组件，能根据输入的动态参数（如主题难度、输入长度）筛选最相关的示例，减少提示词体积，提升模型响应效率。以下是“按主题难度匹配示例”的工程化案例： 运行下面脚本的时候，需要创建一个名为 learning\_method\_examples\.json 的文件，内容如下：

```json
[
{"subject": "Python编程（入门）", "difficulty": "easy", "method": "核心目标：掌握基础语法；学习步骤：1.变量与数据类型 2.条件语句；注意事项：边学边练"},
{"subject": "Python编程（进阶）", "difficulty": "hard", "method": "核心目标：掌握面向对象与库开发；学习步骤：1.类与对象 2.模块开发；注意事项：参与开源项目"},
{"subject": "机器学习（入门）", "difficulty": "easy", "method": "核心目标：理解基础概念；学习步骤：1.数据预处理 2.简单模型；注意事项：用Excel辅助理解"},
{"subject": "机器学习（进阶）", "difficulty": "hard", "method": "核心目标：掌握模型优化；学习步骤：1.特征工程 2.超参数调优；注意事项：研读论文复现实验"}
]
```

```python
# 导入工程化所需组件
from langchain_core.prompts import FewShotPromptTemplate, PromptTemplate
from langchain_core.example_selectors import BaseExampleSelector
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv
import os
import json
from typing import Dict, List
from pydantic import SecretStr

# 1. 环境初始化（工程化标准操作：环境变量管理密钥）
load_dotenv()
API_KEY = os.getenv("API_KEY")
BASE_URL = os.getenv("BASE_URL")  # 从环境变量读取，未配置时默认为None（使用OpenAI官方地址）

if not API_KEY:
    raise ValueError("未检测到 API_KEY，请检查 .env 文件是否配置正确")

chat_model = ChatOpenAI(
    api_key=SecretStr(API_KEY),
    base_url=BASE_URL,
    model="qwen-turbo",  # 选择对话模型
    temperature=0.3,

)

# 2. 工程化示例管理：从JSON文件加载示例（避免硬编码，便于维护）
with open("learning_method_examples.json", "r", encoding="utf-8") as f:
    examples = json.load(f)  # 从JSON中直接提取示例数据列表
# 示例文件格式参考（learning_method_examples.json）前面内容

# 3. 方案A：ExampleSelector按长度筛选示例（控制提示词总长度）
# example_selector = LengthBasedExampleSelector(
#     examples=examples,
#     example_prompt=PromptTemplate(
#         input_variables=["subject", "difficulty", "method"],
#         template="学科：{subject}\n难度：{difficulty}\n学习方法：{method}\n"
#     ),
#     max_length=150,  # 控制示例总长度，避免提示词过长
#     get_text_length=lambda x: len(x)  # 长度计算函数
# )

# 4. 方案B（推荐）：自定义ExampleSelector按难度筛选示例
# 当需要根据用户输入的特征（如难度）精准匹配示例时，可自定义ExampleSelector
class DifficultyExampleSelector(BaseExampleSelector):
    """根据用户输入的 difficulty 字段筛选样本"""
    def __init__(self, examples: List[Dict[str, str]]):
        self.examples = examples

    def add_example(self, example: Dict[str, str]) -> None:  # 添加新示例的方法，允许动态添加示例到选择器中
        self.examples.append(example)

    def select_examples(self, input_variables: Dict[str, str]) -> List[Dict]:
        # 获取用户输入的难度等级，如果没有提供则默认为 'easy'
        target_difficulty = input_variables.get("difficulty", "easy") # 解释代码：从用户输入的动态参数中获取“difficulty”字段的值，如果用户没有提供这个字段，则默认使用“easy”作为难度等级。这样可以确保在用户没有明确指定难度时，系统仍然能够正常工作，并且默认提供入门级的示例。
        """
        详细解释：
        1. input_variables是用户输入的动态参数字典，包含了用户在提示词中传入的所有参数（如new_subject、difficulty等）。
        2. 通过input_variables.get("difficulty", "easy")方法尝试获取用户输入的“difficulty”参数，如果用户没有提供这个参数，则默认使用“easy”作为难度等级。这种设计可以确保在用户没有明确指定难度时，系统仍然能够正常工作，并且默认提供入门级的示例。
        3. 最后，函数返回一个列表，包含了所有示例中“difficulty”字段与用户输入的难度等级匹配的示例。这些示例将被用来构建最终的提示词，确保模型看到的示例与用户需求高度相关，从而提高生成结果的质量和针对性。
        """
        return [ex for ex in self.examples if ex.get("difficulty") == target_difficulty] # 过滤出匹配难度的所有示例


# 本案例使用方案B（按难度筛选），如需使用方案A（按长度筛选），取消方案A的注释并注释掉方案B即可
example_selector = DifficultyExampleSelector(examples=examples)

# 5. 构建工程化少样本模板
few_shot_prompt = FewShotPromptTemplate(
    example_selector=example_selector,  # 详细解释：将自定义的ExampleSelector传入FewShotPromptTemplate，替换掉固定的examples参数。这样，在每次格式化提示词时，模板会根据用户输入的动态参数（如难度）调用example_selector.select_examples()方法，从而动态选择出与用户需求最相关的示例。这种设计使得提示词更加灵活和智能，能够根据不同用户的需求提供定制化的示例，从而提高模型生成结果的质量和针对性。
    example_prompt=PromptTemplate(
        input_variables=["subject", "difficulty", "method"],
        template="学科：{subject}\n难度：{difficulty}\n学习方法：{method}\n"
    ),
    example_separator="\n", # 控制examples示例之间的分隔方式
    prefix="少样本提示：", # 控制示例前的固定文本，作用是引导模型理解接下来是一些示例内容，增强提示词的结构化和可读性
    suffix="参考以上示例，回答：\n学科：{new_subject}\n难度：{difficulty}\n学习方法：",
    input_variables=["new_subject", "difficulty"]  # 新增难度参数
)

# 6. 动态生成不同难度的提示词
# 场景1：生成入门级LangChain学习方法
formatted_prompt_easy = few_shot_prompt.format(
    new_subject="LangChain",
    difficulty="easy"
)
print("入门级少样本提示词：")
print(formatted_prompt_easy)
result_easy = chat_model.invoke([{"role": "user", "content": formatted_prompt_easy}])
print("\n入门级学习方法：")
print(result_easy.content)

# 场景2：生成进阶级LangChain学习方法
formatted_prompt_hard = few_shot_prompt.format(
    new_subject="LangChain",
    difficulty="hard"
)
print("\n进阶级少样本提示词：")
print(formatted_prompt_hard)
result_hard = chat_model.invoke([{"role": "user", "content": formatted_prompt_hard}])
print("\n进阶级学习方法：")
print(result_hard.content)
"""
逻辑说明：
1. 定义了一个自定义的ExampleSelector（DifficultyExampleSelector），它根据用户
输入的“difficulty”字段筛选示例。这样，用户在请求学习方法时，可以指定需要“easy”还是“hard”的示例，系统会自动选择与之匹配的示例来构建提示词。
2. 在构建FewShotPromptTemplate时，将这个自定义的ExampleSelector传入
3. 在格式化提示词时，传入用户需求的“new_subject”和“difficulty”参数，模板会自动调用ExampleSelector筛选出与用户指定难度匹配的示例，从而生成针对不同难度的提示词。这种设计使得提示词更加灵活和智能，能够根据不同用户的需求提供定制化的示例，从而提高模型生成结果的质量和针对性。
4. 最后，分别生成了入门级和进阶级的提示词，并调用模型生成对应难度的学习方法。通过这种方式，用户可以根据自己的需求选择适合的难度等级，从而获得更有针对性的学习建议。
"""
```

**代码理解有点困难，差不多知道怎么回事，过一段时间再试着理解一次**

# 输出解析：让输出更可控

什么是解析层，我的理解就是给大模型输出加工

大模型喜欢输出 **自由文本** ，但是，在使用的时候，这些数据往往不能直接使用，需要用正则表达式的重新处理，这肯定增加了不必要的工作，所以我们要用解析层给他加工一下。

|场景需求|无解析器（原始输出）|有解析器（结构化输出）|
|---|---|---|
|提取商品评论关键词|“这个手机续航好、拍照清晰，但系统有点卡”|\[\&\#34;续航好\&\#34;, \&\#34;拍照清晰\&\#34;, \&\#34;系统卡顿\&\#34;\]|
|生成用户信息|“用户叫张三，25岁，手机号138xxxx1234”|\{\&\#34;name\&\#34;:\&\#34;张三\&\#34;,\&\#34;age\&\#34;:25,\&\#34;phone\&\#34;:\&\#34;138xxxx1234\&\#34;\}|
|分析订单状态|“订单号A123已发货，预计3天到；订单B456还在待付款”|\[\{\&\#34;orderId\&\#34;:\&\#34;A123\&\#34;,\&\#34;status\&\#34;:\&\#34;已发货\&\#34;,\&\#34;estimate\&\#34;:\&\#34;3天\&\#34;\},\{\&\#34;orderId\&\#34;:\&\#34;B456\&\#34;,\&\#34;status\&\#34;:\&\#34;待付款\&\#34;\}\]|

## 案例：JsonOutputParser

适用场景：快速搭建 Demo，需要简单的键值对结构（无需复杂类型校验），配置简单，无需定义数据模型。核心是自动引导模型输出 JSON 格式，并转化为 Python 字典。

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv
import os

# 1. 环境与模型初始化（省略，同方案1）
load_dotenv()
API_KEY = os.getenv("API_KEY")
BASE_URL = os.getenv("BASE_URL")  # 从环境变量读取，未配置时默认为None（使用OpenAI官方地址）
llm = ChatOpenAI(
    api_key=API_KEY,
    base_url=BASE_URL,
    model="deepseek-chat",
    temperature=0.3
)

# 2. 创建 JSON 解析器（无需额外配置，默认引导模型输出 JSON）
parser = JsonOutputParser()

# 3. 构建提示模板（无需手动嵌入格式指令，解析器自动关联）
prompt = PromptTemplate(
    template="请介绍1个LangChain开发工具，输出工具名和核心功能。{format_instructions}",
    input_variables=[],
    partial_variables={"format_instructions": parser.get_format_instructions()}
)

# 4. 链式调用（LangChain ≥1.0.0 推荐方式，自动完成提示+调用+解析）
chain = prompt | llm | parser
result = chain.invoke({})  # 无输入参数，传入空字典

print("解析后的JSON（Python字典）：")
print(result)
print("获取单个字段：", result.get('tool_name', None))  # 可直接用于业务逻辑
```

> （注：文档部分内容可能由 AI 生成）
