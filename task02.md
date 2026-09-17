# Task02｜Chapter 7：构建你的 Agent 框架

## 1. 本次任务目标

Task00 和 Task01 更像是在“从零拼一个 Agent”：

* Task00：第一次跑通 Agent Loop，理解 LLM、Tool、Runtime、History 的关系。
* Task01：实现 ReAct、Plan-and-Solve、Reflection 三种经典 Agent 范式。
* Task02：进一步理解如何把这些零散组件整理成一个可复用的 **Agent Framework**。

这一次最大的变化不是学了一个新的 Agent 算法，而是开始理解：

> 如何把 LLM、Agent、Message、Tool、History、Config、Exception 等组件组织成一套统一框架。

---

# 2. HelloAgents 框架的基本结构

最开始容易把很多东西混在一起，后来逐渐把结构理顺为：

```text
HelloAgents Framework
│
├── Agent
│   └── SimpleAgent
│       └── MySimpleAgent
│           └── basic_agent / travel_agent / ...
│
├── HelloAgentsLLM
│
├── Message
│
├── Config
│
├── Exception
│
└── Tool System
    ├── Tool
    ├── ToolRegistry
    └── CalculatorTool / SearchTool / 自定义 Tool
```

其中最重要的是区分 **类（class）** 和 **实例（instance）**。

例如：

```text
Agent
↓
SimpleAgent
↓
MySimpleAgent
```

这是继承关系，是“模板逐渐具体化”的过程。

而：

```python
basic_agent = MySimpleAgent(...)
```

才真正创建了一个可以运行的 Agent 实例。

所以：

```python
basic_agent.run(...)
```

运行的是 `basic_agent` 这个实例，而不是 `MySimpleAgent` 这个类本身。

---

# 3. Agent、SimpleAgent、MySimpleAgent 与继承

`Agent` 是整个框架的抽象基类。

它规定所有 Agent 都应具备一些共同结构，例如：

```text
name
llm
system_prompt
config
history
run()
```

其中：

* `name`：Agent 名称
* `llm`：使用哪个 LLM Client
* `system_prompt`：Agent 的系统提示词
* `history`：对话历史
* `run()`：统一执行入口

但基类并不规定每个 Agent 具体应该怎么工作。

例如：

```text
SimpleAgent.run()
→ 普通对话

ReActAgent.run()
→ Thought → Action → Observation

ReflectionAgent.run()
→ Execute → Reflect → Refine

PlanAndSolveAgent.run()
→ Plan → Execute
```

也就是说：

> Base Agent 规定“所有 Agent 长什么样”，具体子类决定“这个 Agent 怎么工作”。

`MySimpleAgent(SimpleAgent)` 则是在框架已有的 `SimpleAgent` 基础上继续扩展，例如增加 Tool Calling。

这里还学到了 `super().__init__()`：

```python
super().__init__(...)
```

它的作用是：

> 先让父类完成已有的基础初始化，再添加当前子类自己的功能。

因此 MySimpleAgent 不需要重新实现 name、llm、history 等所有基础能力。

---

# 4. run() 和 invoke() 的区别

这一部分一开始很容易混淆。

## run()

```python
agent.run(...)
```

表示：

> 让一个 Agent 完成一次完整任务。

一次 `run()` 可能涉及：

* 多次 LLM 调用
* 多次 Tool 调用
* 多轮循环
* History 更新
* 最终结果生成

---

## invoke()

```python
llm.invoke(...)
```

表示：

> LLM Client 向模型进行一次完整调用。

所以一次 Agent `run()` 内可以发生多次 `invoke()`。

例如带 Tool 的 Agent：

```text
agent.run()
↓
invoke #1
↓
LLM：我要调用 calculator
↓
执行 calculator
↓
得到 152
↓
invoke #2
↓
LLM 根据工具结果输出最终答案
↓
run() 结束
```

因此可以总结：

> `run()` 是 Agent 层的一次任务；
> `invoke()` 是其中一次模型调用。

---

# 5. LLM Client 的作用

`HelloAgentsLLM` 并不是模型本身。

它更像模型通信层：

```text
Agent
↓
HelloAgentsLLM
↓
OpenRouter
↓
Ling
```

它负责：

* 模型 ID
* API Key
* Base URL
* 请求格式
* 普通调用
* 流式调用
* 错误处理

Agent 不需要直接关心 OpenRouter 请求怎么写，只需要调用：

```python
self.llm.invoke(messages)
```

这体现了框架里的“职责分离”。

---

# 6. Message 与 History

框架用统一的 `Message` 结构保存消息。

例如：

```python
Message(
    role="user",
    content="你好"
)
```

role 可以包括：

```text
user
assistant
system
tool
```

这里需要特别区分：

`user` 和 `assistant` 不是“消息本身”，而是每条 Message 的角色。

例如：

```text
Message 1
role = user
content = "你好"

Message 2
role = assistant
content = "你好！"
```

这就是两条 Message。

因此一次普通问答通常会产生：

```text
user
assistant
```

共两条 History。

测试中：

```text
对话历史: 4 条消息
```

是因为 basic_agent 完成了两轮对话：

```text
user
assistant
user
assistant
```

---

# 7. Config

Config 用来集中管理框架参数，例如：

```text
temperature
max_tokens
debug
max_history_length
```

这样避免参数散落在各处代码中。

同时可以支持通过 `.env` 加载配置。

不过这一次实际使用的 `.env` 主要还是：

```text
LLM_MODEL_ID
LLM_API_KEY
LLM_BASE_URL
LLM_TIMEOUT
TAVILY_API_KEY
```

并没有实际配置 temperature。

因此要区分：

> 框架支持某项 Config，不代表当前项目已经实际使用了该参数。

另外 `.env` 位于项目根目录：

```text
Z:\hello-agents\.env
```

即使运行的是：

```text
code\chapter7\test_simple_agent.py
```

通过 `load_dotenv()` 仍然可以加载根目录环境变量。

---

# 8. Exception

之前调用模型时遇到过：

```text
openai.InternalServerError: 502
```

底层错误来自 OpenRouter 的上游 Provider。

HelloAgents 又把它包装成：

```text
HelloAgentsException:
LLM调用失败...
```

这体现出 Framework 的异常封装：

> 上层 Agent 不需要处理不同 Provider 的各种底层异常，而可以统一处理框架自己的异常。

---

# 9. SimpleAgent 测试

Task02 中跑通了 4 个测试。

## 测试1：基础对话

```text
用户
↓
MySimpleAgent.run()
↓
HelloAgentsLLM.invoke()
↓
Ling
↓
回复
↓
保存 History
```

没有 Tool，也没有 Tool Loop。

---

## 测试2：Tool 增强

注册 CalculatorTool 后：

```text
用户问题
↓
Agent
↓
LLM
↓
LLM 决定调用 calculator
↓
Parser
↓
ToolRegistry
↓
CalculatorTool
↓
152
↓
再次调用 LLM
↓
最终回答
```

这里认识到：

> LLM 并不是直接调用 Python Tool，而只是输出“我要调用哪个 Tool”。

真正执行 Python 的是 Agent Runtime。

---

## 测试3：Streaming

流式输出时发现结果出现：

```text
你好你好
很高兴很高兴
```

检查后发现是流式内容被重复打印。

底层：

```text
HelloAgentsLLM.stream_invoke()
```

已经：

```python
print(content)
yield content
```

上层如果又打印 `chunk`，就会重复。

这个 bug 很适合理解：

> Framework 分层之后，每一层应该只负责自己的职责。

更合理的是：

```text
LLM Client
→ 提供 chunk

Agent / UI
→ 决定如何展示
```

---

## 测试4：动态 Tool 管理

最开始：

```text
basic_agent
没有 calculator
```

然后运行：

```python
basic_agent.add_tool(calculator)
```

之后当前运行中的 `basic_agent` 就拥有了 calculator。

这不是修改源代码，而是改变：

> 当前 Python 进程中这个 Agent 实例的运行时状态。

如果程序结束，再次运行脚本，就会重新创建 Agent，需要重新执行注册逻辑。

---

# 10. Tool 与 ToolRegistry

Tool System 是 Task02 最重要的新增内容之一。

HelloAgents 本身已经提供工具基类：

```python
Tool
```

概念上它就是 Base Tool。

具体工具继承它：

```text
Tool
│
├── CalculatorTool
├── SearchTool
└── TextLengthTool
```

所有 Tool 都遵循统一接口，例如：

```text
name
description
get_parameters()
run()
```

这样 Agent 不需要知道每个工具内部是怎么实现的。

---

## ToolRegistry

ToolRegistry 可以理解为工具注册表 / 工具箱。

它主要负责：

```text
register_tool()
→ 注册工具

get_tools_description()
→ 获取所有工具的名字和说明

get_tool()
→ 根据名字找到工具

execute_tool()
→ 执行工具
```

注册步骤的本质是建立：

```text
工具名字
↓
真正的 Python Tool 对象
```

这样的映射。

例如：

```text
"text_length"
↓
TextLengthTool 实例
```

LLM 只会看到：

```text
text_length：用于统计字符数量
```

然后输出类似：

```text
我要使用 text_length
```

真正找到 Python 对象的是 ToolRegistry。

因此：

> Tool 的 `name` 是给程序匹配用的；
> Tool 的 `description` 是给 LLM 理解用途用的。

如果 LLM 输出的 Tool Name 与 Registry 中登记的不一致，就可能找不到工具。

---

# 11. 自定义 Tool 实验

最后自己实现了一个：

```text
TextLengthTool
```

功能：

> 统计文本字符数。

完整流程：

```text
HelloAgents 的 Tool 基类
↓
定义 TextLengthTool
↓
创建 text_length_tool 实例
↓
注册到 ToolRegistry
↓
创建 MySimpleAgent 实例
↓
把 ToolRegistry 交给 Agent
↓
Agent 把工具描述写入 Prompt
↓
Ling 决定调用 text_length
↓
Parser 解析 Tool Name
↓
ToolRegistry 找到 TextLengthTool
↓
Python 执行 len()
↓
返回工具结果
↓
再次 invoke LLM
↓
生成最终答案
```

输入：

```text
Hello Agent
```

工具正确返回：

```text
11 个字符
```

---

# 12. Tool 被调用两次的实验现象

运行时出现：

```text
🔧 检测到 1 个工具调用
🔧 检测到 1 个工具调用
```

说明同一次 `run()` 中发生了两轮 Tool Calling。

原因是 system prompt 写了：

```text
当用户要求统计文本字符数量时，必须使用 text_length 工具。
```

第一次：

```text
LLM → Tool → 得到 11
```

第二次模型重新看到上下文后，仍然判断：

```text
“这是统计字符数的问题，而 Prompt 要求必须使用 Tool”
```

于是再次调用 Tool。

这个实验让我真正理解：

> Agent Loop 中，每轮 LLM 都会重新决定下一步行动。

LLM 并不会天然记住：

> “我上一轮已经调用过这个 Tool，所以这轮不能再调用。”

这种约束需要通过 Prompt、State 或 Loop Engineering 显式设计。

例如可以改为：

```text
如果尚未获得工具结果，则调用 text_length。
如果已经获得工具结果，则直接基于结果回答，不要重复调用。
```

---

# 13. Loop 和 Loop Engineering

这里也重新澄清了一个概念。

例如：

```python
while current_iteration < max_tool_iterations:
```

这个 `while` 本身只是：

> Loop。

Agent Loop 可能是：

```text
LLM
↓
Tool
↓
Observation
↓
再次 LLM
```

而 Loop Engineering 是：

> 对这个循环做设计。

例如：

```text
最大循环次数
停止条件
工具失败怎么办
是否重试
是否重复调用同一个 Tool
成本限制
验证机制
Fallback
```

因此：

> Loop 是结构；
> Loop Engineering 是围绕 Loop 的控制策略。

---

# 14. Framework 和 Codex + Skill 的关系

之前使用 Codex Skill 时，Codex 会自己判断：

```text
用户请求
↓
选择 Skill
↓
执行 Skill
```

因为 Codex 本身已经提供：

```text
Agent Runtime
Routing
Tool System
Loop
State
Execution Harness
```

Skill 更像：

> 给已有 Agent 增加一套任务知识 / SOP / Scripts / Capability。

而 HelloAgents Task02 是反过来：

> 自己理解并搭建 Agent Framework 的基础结构。

目前 Chapter 7 示例默认：

```python
basic_agent.run(...)
```

也就是程序员已经明确决定：

> 使用哪个 Agent。

它还没有实现自动 Agent Routing。

如果以后创建：

```text
travel_agent
coding_agent
finance_agent
```

才可能再增加 Router：

```text
用户
↓
Router
↓
选择哪个 Agent
↓
agent.run()
```

---

# 15. 最终理解

经过 Task02，可以把 Agent Framework 理解为：

```text
用户输入
↓
具体 Agent 实例 .run()
↓
组织 Prompt + History + Tool Description
↓
HelloAgentsLLM.invoke()
↓
模型决定下一步
↓
如果需要 Tool：
    Parser
    ↓
    ToolRegistry
    ↓
    Python Tool
    ↓
    Tool Result
    ↓
    再次 invoke()
↓
最终输出
↓
保存 History
↓
run() 结束
```

最终我理解到：

> Agent Framework 的核心不是某一种“智能算法”，而是把模型、工具、消息、状态、执行循环、配置和异常等组件组织成统一、可复用、可扩展的系统。

Task00 是第一次让 Agent 跑起来；

Task01 是理解 Agent 如何“思考和行动”；

Task02 则是在理解：

> **如何把这些能力做成一个真正可以继续扩展的框架。**

<img width="1346" height="1094" alt="ee884900-9c09-4a8f-9fc5-79de6284bc9a" src="https://github.com/user-attachments/assets/4e7161de-fb7a-4068-a82e-e6c9482bcebc" />

<img width="1346" height="1094" alt="a5ec2ed6-7293-4397-80af-07e49f5d8249" src="https://github.com/user-attachments/assets/e7bc2e16-51c6-4036-ac5f-ccd9823c67ea" />

<img width="1346" height="1094" alt="67d78908-6d0f-41a6-abb4-9835ae9192f2" src="https://github.com/user-attachments/assets/f1151095-321f-416f-b007-623334b23a7f" />
