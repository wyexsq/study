# 第三章 LangChain进阶组件实操

目标：构建带记忆、能交互外部系统的智能应用

通过上次的学习，我们不难发现。每一次的调用都是一个“新的人”，所以我们要解决这个问题

解决工具：LangChain的Memory组件。

过程逻辑

> - 存储（Save）：将每一轮的用户输入（HumanMessage）和AI输出（AIMessage）保存到指定存储介质（内存、数据库等）；
> - 提取（Load）：新一轮对话时，从存储介质中提取历史对话，注入到Prompt中供LLM参考。

类别：全量，窗口，摘要。

### 全记忆：

```python
import os
from dotenv import load_dotenv
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.runnables import RunnablePassthrough
from langchain_core.chat_history import BaseChatMessageHistory, InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_openai import ChatOpenAI
from pydantic import SecretStr

# 加载环境变量（确保.env文件中配置了API_KEY）
load_dotenv()
API_KEY = os.getenv("API_KEY")
BASE_URL = os.getenv("BASE_URL")

# 初始化LLM模型
llm = ChatOpenAI(
    api_key=SecretStr(API_KEY), # type: ignore
    base_url=BASE_URL,
    model="qwen-max",  # 替换为你使用的模型名称
    temperature=0.3  # 降低随机性，保证输出稳定
)
# 1. 定义提示词模板（包含历史消息占位符）
"""提示词模板定义了对话的结构和内容，包括系统提示和用户输入。历史消息占位符允许我们在生成回复时包含完整的历史对话，从而使模型能够更好地理解上下文并生成相关的回复。
- system: 定义了对话助手的角色和行为。
- MessagesPlaceholder: 占位符，用于插入历史对话消息。
- human: 定义了用户当前输入的格式。"""
full_memory_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是友好的对话助手，需基于完整的历史对话回答用户问题。"),
    MessagesPlaceholder(variable_name="chat_history"),  # 历史消息占位符
    ("human", "{user_input}")  # 用户当前输入
])

# 2. 构建基础链（提示词 + LLM）
"""基础链将提示词模板和LLM模型连接起来，形成一个可执行的对话链。
- full_memory_prompt：定义了对话的结构和内容，包括系统提示和用户输入。
- llm：负责生成对话回复的语言模型。
通过将提示词模板和LLM模型组合在一起，基础链能够根据用户输入和历史对话生成适当的回复。"""
base_chain = full_memory_prompt | llm

# 3. 会话历史存储（内存模式，生产环境可替换为数据库存储）
full_memory_store = {}

# 4. 定义会话历史获取函数（核心：返回完整历史）
"""get_full_memory_history函数根据session_id获取会话历史，如果不存在则创建新的历史记录。它确保每个会话都有一个独立的历史记录，允许模型在生成回复时访问完整的对话上下文。
- session_id：唯一标识一个会话的字符串。
- full_memory_store：一个字典，用于存储每个会话的历史记录。
- InMemoryChatMessageHistory：一个内存中的聊天消息历史类，用于存储和管理对话历史。"""
def get_full_memory_history(session_id: str) -> BaseChatMessageHistory:
    """根据session_id获取会话历史，不存在则创建新的历史记录"""
    if session_id not in full_memory_store:
        full_memory_store[session_id] = InMemoryChatMessageHistory()
    return full_memory_store[session_id]

# 5. 构建带全量记忆的对话链
full_memory_chain = RunnableWithMessageHistory(
    runnable=base_chain,
    get_session_history=get_full_memory_history,
    input_messages_key="user_input",  # 输入中用户问题的键名
    history_messages_key="chat_history"  # 传入提示词的历史消息键名
)

# 测试多轮对话（指定session_id=user_001，隔离不同用户）
config = {"configurable": {"session_id": "user_001"}}

# 第一轮对话
response1 = full_memory_chain.invoke({"user_input": "我要学习SQL"}, config=config)
print("助手回复1：", response1.content)


# 第二轮对话（验证记忆：询问历史信息）
response2 = full_memory_chain.invoke({"user_input": "你最推荐什么SQL书，只能推荐最好的一本？"}, config=config)
print("助手回复2：", response2.content)


# 查看完整历史记录
print("\n全量记忆的对话历史：")
for msg in get_full_memory_history("user_001").messages:
    print(f"{msg.type}: {msg.content}")
```

窗口记忆：只保留n轮对话记录

```python
# 1. 定义提示词模板（与全量记忆通用，可复用）
window_memory_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是友好的对话助手，需基于最近的对话历史回答用户问题。"),
    MessagesPlaceholder(variable_name="chat_history"),
    ("human", "{user_input}")
])

# 2. 构建基础链
window_base_chain = window_memory_prompt | llm

# 3. 会话历史存储
window_memory_store = {}
WINDOW_SIZE = 2  # 保留最近2轮对话（即最近4条消息：用户-助手-用户-助手）

# 4. 定义带窗口限制的会话历史获取函数
def get_window_memory_history(session_id: str) -> BaseChatMessageHistory:
    """获取会话历史，仅保留最近WINDOW_SIZE轮对话"""
    if session_id not in window_memory_store:
        window_memory_store[session_id] = InMemoryChatMessageHistory()

    # 获取完整历史，截取最近WINDOW_SIZE轮（每轮2条消息）
    history = window_memory_store[session_id]
    if len(history.messages) > 2 * WINDOW_SIZE:
        # 截取后WINDOW_SIZE轮消息（保留最新的）
        history.messages = history.messages[-2 * WINDOW_SIZE:]
    return history

# 5. 构建带窗口记忆的对话链
window_memory_chain = RunnableWithMessageHistory(
    runnable=window_base_chain,
    get_session_history=get_window_memory_history,
    input_messages_key="user_input",
    history_messages_key="chat_history"
)

# 测试多轮对话（session_id=user_002，与全量记忆会话隔离）
config = {"configurable": {"session_id": "user_002"}}

# 模拟5轮对话，验证窗口记忆的截断效果
inputs = [
    "我叫小红",
    "我喜欢画画",
    "我来自上海",
    "我是一名学生",
    "我刚才说我来自哪里？",  # 第5轮：询问第3轮的信息，验证窗口截断
    "我叫什么名字？"  # 第6轮：询问第1轮的信息，验证窗口记忆
]

for i, user_input in enumerate(inputs, 1):
    response = window_memory_chain.invoke({"user_input": user_input}, config=config) # type: ignore
    print(f"\n第{i}轮 - 助手回复：", response.content)

# 查看窗口记忆的最终历史（仅保留最近2轮）
print("\n窗口记忆的最终对话历史（最近2轮）：")
for msg in get_window_memory_history("user_002").messages:
    print(f"{msg.type}: {msg.content}")
```

摘要记忆：

```python
# 1. 定义摘要生成提示词（用于压缩对话历史）
summary_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是对话摘要助手，需简洁总结以下对话的核心信息（包含用户身份、偏好、关键问题等），不超过50字。"),
    ("human", "对话历史：{chat_history_text}\n请生成摘要：")
])

# 2. 构建摘要生成链（输入完整历史文本，输出摘要）
summary_chain = summary_prompt | llm

# 3. 定义对话记忆提示词（注入摘要而非完整历史）
summary_memory_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是友好的对话助手，需基于对话摘要回答用户问题，摘要包含核心上下文信息。"),
    ("system", "对话摘要：{chat_summary}"),  # 注入摘要
    ("human", "{user_input}")
])

# 4. 构建基础对话链（提示词 + LLM）
summary_base_chain = (
    RunnablePassthrough.assign(
        chat_summary=lambda x: summary_chain.invoke(
            {
                "chat_history_text": "\n".join(
                    [f"{msg.type}: {msg.content}" for msg in x["chat_history"]]
                )
            }
        ).content
    )
    | summary_memory_prompt
    | llm
)

# 5. 会话历史存储（保存完整历史用于生成摘要）
summary_memory_store = {}

# 6. 定义会话历史获取函数
def get_summary_memory_history(session_id: str) -> BaseChatMessageHistory:
    if session_id not in summary_memory_store:
        summary_memory_store[session_id] = InMemoryChatMessageHistory()
    return summary_memory_store[session_id]

# 7. 构建带摘要记忆的对话链
summary_memory_chain = RunnableWithMessageHistory(
    runnable=summary_base_chain,
    get_session_history=get_summary_memory_history,
    input_messages_key="user_input",
    history_messages_key="chat_history"  # 传入完整历史用于生成摘要
)
```

### 摘要记忆

上面的例子最大的问题是：如果要长期对话，积累的数据量会非常大，所以，我们会想到，，不是所有信息都需要保存，我们可以保存关键部分，这个思想，反映的就是摘要记忆

```python
# 1. 定义摘要生成提示词（用于压缩对话历史）
summary_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是对话摘要助手，需简洁总结以下对话的核心信息（包含用户身份、偏好、关键问题等），不超过50字。"),
    ("human", "对话历史：{chat_history_text}\n请生成摘要：")
])

# 2. 构建摘要生成链（输入完整历史文本，输出摘要）
summary_chain = summary_prompt | llm

# 3. 定义对话记忆提示词（注入摘要而非完整历史）
summary_memory_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是友好的对话助手，需基于对话摘要回答用户问题，摘要包含核心上下文信息。"),
    ("system", "对话摘要：{chat_summary}"),  # 注入摘要
    ("human", "{user_input}")
])

# 4. 构建基础对话链（提示词 + LLM）
summary_base_chain = (
    RunnablePassthrough.assign(
        chat_summary=lambda x: summary_chain.invoke(
            {
                "chat_history_text": "\n".join(
                    [f"{msg.type}: {msg.content}" for msg in x["chat_history"]]
                )
            }
        ).content
    )
    | summary_memory_prompt
    | llm
)

# 5. 会话历史存储（保存完整历史用于生成摘要）
summary_memory_store = {}

# 6. 定义会话历史获取函数
def get_summary_memory_history(session_id: str) -> BaseChatMessageHistory:
    if session_id not in summary_memory_store:
        summary_memory_store[session_id] = InMemoryChatMessageHistory()
    return summary_memory_store[session_id]

# 7. 构建带摘要记忆的对话链
summary_memory_chain = RunnableWithMessageHistory(
    runnable=summary_base_chain,
    get_session_history=get_summary_memory_history,
    input_messages_key="user_input",
    history_messages_key="chat_history"  # 传入完整历史用于生成摘要
)
```

但是，我们不难发现，我们生成摘要是利用LLM。这里就有一个问题，这肯定是增加了token的消耗，所以我们会想，每次总结，是不是太频繁了，如果我们把几轮对话收集，然后再总结是不是更好。我觉得是可以的。然后是全量记忆，这个也有大量的消耗，你想想，每次大量的输入，消耗也在不断增加。窗口记忆就不用说了，就是定期忘记一些东西。

优化：我们看看记忆的存储，其实工程化我们要用数据库，这样更好

### TOOL

搞了这么多，我们发现，总是在聊天，没意思，怎么可以像opencode等等，更强大呢。这个时候吧，我们就要借助TOOL

工作逻辑 ：思考→行动→反馈

要实现工具调用，需要三个核心组件配合，就像一个团队：

- Tool（工具）：具体的“干活工具”，比如查询天气的工具、读文件的工具。每个工具都有明确的名称和描述（AI就是通过描述知道该用哪个工具的）；

- Toolkit（工具包）：把相关的工具打包在一起，比如“文件操作工具包”里包含读文件、写文件、列目录三个工具；

- Agent（智能体）：团队的“指挥官”，负责协调LLM和工具——判断要不要调用工具、用哪个工具、怎么处理工具返回的结果。

  ```python
  from langchain.agents import create_agent
  from langchain_core.tools import tool
  from langchain_openai import ChatOpenAI
  from dotenv import load_dotenv
  import os
  
  # ======================
  # 1. 环境
  # ======================
  load_dotenv()
  API_KEY = os.getenv("API_KEY")
  BASE_URL = os.getenv("BASE_URL")
  llm = ChatOpenAI(
      api_key=API_KEY,
      base_url=BASE_URL,
      model="deepseek-chat",
      temperature=0.3,
  )
  
  # ======================
  # 2. 工具
  # ======================
  @tool
  def weather_query(city: str) -> str:
      """查询指定城市天气"""
      weather_data = {
          "北京": "北京今日天气：晴，-2~8℃",
          "上海": "上海今日天气：多云，5~12℃",
          "广州": "广州今日天气：小雨，18~25℃",
      }
      return weather_data.get(city, f"暂无 {city} 数据")
  
  tools = [weather_query]
  
  # ======================
  # 3. 创建 Agent（开启 debug）
  # ======================
  agent = create_agent(
      model=llm,
      tools=tools,
      debug=True,  # 打开过程打印
  )
  
  # ======================
  # 4. 运行
  # ======================
  response = agent.invoke({
      "messages": [
          {"role": "user", "content": "北京今天的天气怎么样？"}
      ]
  })
  
  print("\n最终回答：")
  print(response["messages"][-1].content)
  
  ```

  LangChain提供了@tool装饰器，这样可以让我们能快速创建自己的工具

创建要求：

要让AI能正确使用你的工具，必须满足三个要求：

- 写清楚文档字符串（docstring）：告诉AI这个工具能做什么、需要什么参数；
- 明确参数类型：用类型注解（比如int、str）指定参数类型，AI才知道该传什么类型的值；
- 返回结果清晰：工具执行后的返回值要简洁明了，方便AI整理成回答。

```python
from langchain.agents import create_agent
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field
from dotenv import load_dotenv
import os

# ======================
# 1. 环境变量
# ======================
load_dotenv()
API_KEY = os.getenv("API_KEY")
BASE_URL = os.getenv("BASE_URL")

llm = ChatOpenAI(
    api_key=API_KEY,
    base_url=BASE_URL,
    model="deepseek-chat",
    temperature=0.3,
)

# ======================
# 2. 参数模型
# ======================
class TemperatureConvertInput(BaseModel):
    temperature: float = Field(description="需要转换的温度值，例如37.0")
    from_unit: str = Field(description="原始温度单位，只能是celsius或fahrenheit")

# ======================
# 3. 工具
# ======================
@tool(args_schema=TemperatureConvertInput)
def temperature_converter(temperature: float, from_unit: str) -> str:
    """温度单位转换工具"""
    
    if from_unit not in ["celsius", "fahrenheit"]:
        return f"错误：单位'{from_unit}'不合法"

    if from_unit == "celsius":
        fahrenheit = temperature * 9/5 + 32
        return f"{temperature}摄氏度 = {fahrenheit:.2f}华氏度"
    else:
        celsius = (temperature - 32) * 5/9
        return f"{temperature}华氏度 = {celsius:.2f}摄氏度"


tools = [temperature_converter]

system_prompt = """
你是一名专业温度转换助手，只能使用temperature_converter工具完成计算。
"""

# ======================
# 4. 创建 Agent
# ======================
agent = create_agent(
    model=llm,
    tools=tools,
    system_prompt=system_prompt,
    debug=True
)

# ======================
# 5. 运行
# ======================
if __name__ == "__main__":

    query = "将37摄氏度转换为华氏度"

    response = agent.invoke({
        "messages": [{"role": "user", "content": query}]
    })

    print("\n最终结果：")
    print(response["messages"][-1].content)

```

## 实践：

创建一个，功能：和我聊天，利用摘要记忆存储我日常生活，要求有时间，然后要.md格式

```python
# =========================
# 1. 基础依赖
# =========================
import os
import re
from datetime import datetime
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

# =========================
# 2. 环境变量 & 模型
# =========================
load_dotenv()
API_KEY = os.getenv("API_KEY")
BASE_URL = os.getenv("BASE_URL")

llm = ChatOpenAI(
    api_key=API_KEY,
    base_url=BASE_URL,
    model="deepseek-chat",
    temperature=0.5,
)


# =========================
# 3. 摘要记忆管理器 + Markdown 存储
# =========================
class DiaryMemory:
    """使用 LLM 生成摘要，记录日常生活，并导出为 Markdown 文件"""

    def __init__(self, storage_file="diary.md"):
        self.storage_file = storage_file
        self.summary = ""  # 当前的生活摘要
        self.recent_history = []  # 最近 N 条带时间戳的对话
        self.recent_limit = 6  # 保留最近 6 条消息（3 轮对话）

        # 初始化 Markdown 文件（如果不存在则创建）
        if not os.path.exists(storage_file):
            with open(storage_file, "w", encoding="utf-8") as f:
                f.write(f"# 📔 我的生活日记\n\n> 自动记录时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n\n---\n\n")
        else:
            # 如果文件已存在，尝试加载已有摘要（可选）
            self._load_existing_summary()

    def _load_existing_summary(self):
        """从 Markdown 文件中读取已有的「生活摘要」部分（简单实现）"""
        try:
            with open(self.storage_file, "r", encoding="utf-8") as f:
                content = f.read()
                # 匹配最新一次「## 📌 生活摘要」区块中的内容
                match = re.search(r"## 📌 生活摘要\n\n(.*?)\n\n---", content, re.DOTALL)
                if match:
                    self.summary = match.group(1).strip()
                    print(f"[系统] 已加载已有摘要：{self.summary[:50]}...")
        except Exception:
            pass  # 无历史摘要则忽略

    def _write_to_md(self, user_msg: str, assistant_msg: str):
        """将一次完整对话追加到 Markdown 文件（带时间戳）"""
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        with open(self.storage_file, "a", encoding="utf-8") as f:
            f.write(f"### 🕒 {timestamp}\n")
            f.write(f"**你：** {user_msg}\n\n")
            f.write(f"**助手：** {assistant_msg}\n\n")
            f.write("---\n\n")

    def _update_summary(self, new_user_msg: str, new_assistant_msg: str):
        """调用 LLM 更新生活摘要 (基于旧摘要 + 新对话内容)"""
        new_dialog = f"用户说：{new_user_msg}\n助手回应：{new_assistant_msg}"
        prompt = ChatPromptTemplate.from_messages([
            ("system", "你是一个善于提炼的私人日记助手。请根据「已有生活摘要」和「新的对话内容」，更新摘要。"
                       "摘要应重点记录用户的日常生活信息（如：去了哪里、做了什么、心情如何、计划等），"
                       "每条信息尽量包含时间点。保持语言简洁但信息完整。\n"
                       "如果用户没有提供新信息，摘要保持不变。\n"
                       "只输出更新后的摘要文本，不要输出其他内容。"),
            ("human", f"已有摘要：\n{self.summary if self.summary else '（无）'}\n\n"
                      f"新对话内容：\n{new_dialog}\n\n"
                      f"请输出更新后的摘要：")
        ])
        chain = prompt | llm
        try:
            new_summary = chain.invoke({}).content.strip()
            if new_summary and new_summary != self.summary:
                self.summary = new_summary
                # 将摘要同步保存到 Markdown 文件（覆盖旧摘要区）
                self._save_summary_to_md()
        except Exception as e:
            print(f"[摘要更新失败] {e}")

    def _save_summary_to_md(self):
        """将当前摘要写入 Markdown 文件（维护一个单独的摘要章节）"""
        try:
            with open(self.storage_file, "r", encoding="utf-8") as f:
                content = f.read()

            # 构造新的摘要区块
            new_summary_block = f"## 📌 生活摘要\n\n{self.summary}\n\n---\n\n"

            # 如果文件中已有摘要区块，替换它；否则插入到开头
            if "## 📌 生活摘要" in content:
                # 替换从 "## 📌 生活摘要" 到下一个 "---" 之间的内容
                pattern = r"(## 📌 生活摘要\n\n).*?(\n\n---\n\n)"
                content = re.sub(pattern, r"\1" + self.summary + r"\2", content, flags=re.DOTALL)
            else:
                # 没有摘要区块，则在标题后插入
                parts = content.split("\n\n", 1)
                if len(parts) == 2:
                    content = parts[0] + "\n\n" + new_summary_block + parts[1]
                else:
                    content = new_summary_block + content

            with open(self.storage_file, "w", encoding="utf-8") as f:
                f.write(content)
        except Exception as e:
            print(f"[保存摘要失败] {e}")

    def add_dialog(self, user_input: str, assistant_output: str):
        """添加一轮对话：写入 md 文件、更新摘要、维护最近历史"""
        # 1. 写入 Markdown（完整对话记录）
        self._write_to_md(user_input, assistant_output)

        # 2. 更新生活摘要（使用 LLM）
        self._update_summary(user_input, assistant_output)

        # 3. 维护最近历史（带时间戳，用于对话上下文）
        timestamp = datetime.now().strftime("%m-%d %H:%M")
        self.recent_history.append(f"{timestamp} 用户：{user_input}")
        self.recent_history.append(f"{timestamp} 助手：{assistant_output}")
        if len(self.recent_history) > self.recent_limit:
            self.recent_history = self.recent_history[-self.recent_limit:]

    def get_context_for_llm(self) -> str:
        """构建给 LLM 的完整上下文：摘要 + 最近历史"""
        context = ""
        if self.summary:
            context += f"【生活摘要】截止目前，你了解到关于用户日常生活的重要信息：\n{self.summary}\n\n"
        if self.recent_history:
            context += "【最近对话】\n" + "\n".join(self.recent_history) + "\n\n"
        else:
            context += "【提示】这是你们的第一次对话。\n\n"
        return context


# =========================
# 4. 聊天主程序
# =========================
def chat_loop():
    print("===== 📔 生活日记聊天助手（摘要记忆 + Markdown 存储）=====")
    print("我会记住你生活中的重要事情（时间、事件、心情等），并写入 diary.md")
    print("输入 q 退出\n")

    memory = DiaryMemory("diary.md")
    # 系统提示词
    system_prompt_template = (
        "你是一个温暖、善于倾听的生活伙伴，名叫小忆。"
        "你的核心任务是：通过与用户聊天，了解并记住用户的日常生活信息（时间、地点、活动、感受、计划等）。"
        "每次回答时，你可以基于已有的「生活摘要」和「最近对话」来与用户自然交流。"
        "如果用户提到新生活信息，务必在回答中体现你对它的关注（例如：“哦，你今天去了公园，听起来很开心！”）。"
        "回答要简洁、亲切，保持在 2~3 句话内，除非用户需要详细帮助。"
        "\n\n"
        "接下来是已知的用户生活摘要和最近对话：\n{context}\n"
        "请根据以上信息回复用户最新的消息：\n{user_message}"
    )

    while True:
        user_input = input("你：").strip()
        if user_input.lower() in ["q", "quit", "退出"]:
            print("助手：你的日记已保存在 diary.md，再见 👋")
            break

        # 构建上下文（摘要 + 最近对话）
        context = memory.get_context_for_llm()

        # 调用 LLM 回复
        prompt_text = system_prompt_template.format(context=context, user_message=user_input)
        response = llm.invoke(prompt_text).content

        print(f"\n助手：{response}\n")

        # 将本轮对话存入记忆（更新摘要、写入 md）
        memory.add_dialog(user_input, response)


if __name__ == "__main__":
    chat_loop()
```

