今天第一次从零把一个最小 Agent 跑了起来。

在真正动手之前，我对 Agent 的理解其实还比较模糊。我大概知道它和普通 LLM 不一样，会“调用工具”，但“到底是谁决定调用工具”“LLM 有没有真的执行 API”“搜索、爬虫、RAG 和 Agent 又是什么关系”，这些概念其实混在一起。

这一轮配置和调试之后，这些东西终于开始变得具体。

## 1. 我们实际搭建了什么？

这次运行的 Agent 是一个很简单的旅行助手。用户提出：

“查询今天北京的天气，然后根据天气推荐一个合适的旅游景点。”

它背后有三个主要外部能力：

* OpenRouter：提供 LLM，相当于负责理解、判断和规划；
* Open-Meteo：提供实时天气；
* Tavily：提供联网搜索能力。

整体流程大概是：

用户提出问题
→ LLM 判断下一步需要什么信息
→ 输出 Action
→ Python 解析 Action
→ 调用对应 Tool
→ 得到 Observation
→ 把 Observation 再交给 LLM
→ LLM 根据新信息决定下一步
→ 最终 Finish

因此，一个很重要的认识是：

**LLM 并没有真的“亲自调用 API”。**

例如模型输出：

`Action: get_weather(city="北京")`

这只是模型生成的一段文本。真正执行 `get_weather()` 的仍然是 Python 程序。

所以这里至少有三个不同角色：

**LLM：Decision Maker / Planner**

负责判断：

* 用户到底想要什么；
* 当前缺什么信息；
* 下一步应该调用哪个工具；
* 信息是否已经足够，可以结束任务。

**Python Agent Runtime：Executor / Orchestrator**

负责：

* 读取模型输出；
* 解析 Action；
* 找到对应函数；
* 真正执行函数；
* 将结果重新加入上下文；
* 控制整个循环什么时候继续、什么时候结束。

**Tool：Capability**

负责真正干活。

例如：

* `get_weather()` 获取天气；
* `get_attraction()` 调用 Tavily 搜索景点。

这个区分让我终于理解了“Agent 会调用工具”到底是什么意思。

更准确地说，是：

**LLM 决定调用什么，程序负责真的调用。**

---

## 2. Agent 和普通 LLM / RAG 的区别

在学习过程中，我还产生了一个问题：

Google 搜索上面的 AI 总结，是不是也是 LLM 决定调用爬虫，然后爬虫拿资料，最后 LLM 总结？

后来发现这里需要拆开几个概念。

爬虫、搜索检索和 LLM 是不同的组件。

Crawler 负责抓网页；

Search / Retrieval 负责从已有索引或资料库中找到相关内容；

LLM 负责理解、推理和生成答案。

传统 RAG 常见的流程更像：

用户问题
→ 程序固定执行检索
→ 把资料交给 LLM
→ LLM 总结答案

也就是说，流程往往是提前规定好的。

而这次运行的 Agent 不一样：

程序只是告诉模型：

“你现在有这些工具。”

具体什么时候调用哪个工具，则由 LLM 根据当前状态决定。

因此我目前对两者的理解是：

**RAG 更像“程序决定流程，LLM 使用被检索出来的信息”。**

**Agent 更像“程序提供行动空间，LLM 对下一步行动拥有一定决策权”。**

当然，现实中的系统通常会更复杂，可能同时混合规则、检索系统、分类器、LLM 和 Agent，并不是所有决策都必须交给 LLM。

---

## 3. 第一个坑：API 服务不一定可靠

教程原本使用 `wttr.in` 查询天气。

但实际测试时出现：

`ProxyError: Unable to connect to proxy`

而且群里其他同学也反馈这个天气服务近期存在问题。

这里一开始很容易陷入一个错误方向：

“是不是一定要把教程里的 wttr.in 修好？”

后来我们直接把天气 Tool 替换成了 Open-Meteo。

新的实现变成：

城市名
→ Open-Meteo Geocoding API
→ 经纬度
→ Open-Meteo Weather API
→ 当前天气

但对 Agent 来说，它看到的接口依然只是：

`get_weather(city)`

上层逻辑完全不用改变。

这让我第一次非常具体地理解了“封装”。

Agent 并不需要知道 `get_weather()` 内部到底调用的是：

* wttr.in；
* Open-Meteo；
* WeatherAPI；
* 还是未来别的天气供应商。

只要这个函数最终仍然接受一个 city，并返回天气结果，上层 Agent 就可以继续工作。

所以一个 Tool 的底层实现是可以被替换的。

这也意味着：

**Agent 不应该和某一个具体 API 供应商强绑定。**

---

## 4. 第二个坑：API Key 不应该直接写进代码

最开始为了快速测试 OpenRouter，我们直接把：

`sk-or-...`

写进 Python 文件。

这样做可以测试，但不适合正式项目。

如果以后执行：

`git push`

API Key 很可能一起被上传到 GitHub。

所以后来我们使用 `.env`：

`OPENAI_API_KEY=...`

`OPENAI_BASE_URL=...`

`MODEL_NAME=...`

`TAVILY_API_KEY=...`

然后 Python 通过：

`load_dotenv()`

和：

`os.getenv(...)`

读取这些配置。

这让我理解了 `.env` 的作用：

**代码描述程序怎么运行，环境变量保存不同机器、不同用户自己的秘密和配置。**

而仓库里的 `.gitignore` 已经忽略了 `.env`，因此 Key 不会正常进入 Git 提交。

这看起来只是一个很小的配置动作，但其实属于非常基础的工程习惯。

---

## 5. Tavily 返回的一大串数据是什么？

第一次运行 Tavily 时，它返回了一大串 Python 字典：

`query`

`results`

`title`

`url`

`content`

`score`

等字段。

这让我意识到，所谓“搜索 API”并不是直接返回一句漂亮的人类答案。

它实际上返回的是结构化数据。

例如：

* `query`：搜索词；
* `results`：搜索结果列表；
* `title`：网页标题；
* `url`：来源；
* `content`：从页面里提取出来的相关文本；
* `score`：结果与 query 的相关程度。

因此 Tool 的输出本质上是“机器可继续处理的数据”。

最后要不要把它整理成人类能读懂的答案，是另一层事情。

---

## 6. 最有意思的一幕：Agent 第一次就犯错了

完整程序第一次运行时，LLM 输出：

`Action: function_name("get_weather", city="北京")`

这其实是错误的。

真正存在的工具叫：

`get_weather`

而不是：

`function_name`

模型应该是把 System Prompt 里的格式示例：

`function_name(arg_name="arg_value")`

错误地理解成了真实函数调用方式。

Python 随后检查：

`available_tools`

发现没有一个叫 `function_name` 的工具，于是返回：

`Observation: 错误：未定义的工具 'function_name'`

如果这是一个普通的一次性 LLM 请求，这里可能就结束了。

但 Agent 没有结束。

这个错误被加入 `prompt_history`，下一轮又重新交给 LLM。

于是模型看到了：

“刚才调用了不存在的工具。”

然后它重新规划，输出：

`Action: get_weather(city="北京")`

这一次调用成功。

随后流程继续：

天气结果
→ LLM 判断还需要景点信息
→ 调用 `get_attraction()`
→ Tavily 返回搜索结果
→ LLM 判断信息已经足够
→ `Finish[...]`

这个过程让我第一次直观看到了：

**Agent 并不是因为“模型永远不会犯错”才有用。**

恰恰相反，它可以通过：

Action
→ Observation
→ Replan

形成反馈闭环。

模型做出行动；

环境告诉它行动的结果；

模型再根据真实结果调整下一步。

这可能才是所谓 agentic behavior 最直观的部分之一。

---

## 7. prompt_history 原来就是最简单的“记忆”

代码里有：

`prompt_history`

每次 LLM 输出 Thought / Action 后，会 append 进去；

工具执行得到 Observation 后，也会 append 进去。

下一轮再通过：

`"\n".join(prompt_history)`

把前面的历史全部重新发给 LLM。

所以模型第三轮之所以“知道”：

“北京现在晴朗，24.2℃。”

并不是 API 模型神奇地永久记住了上一轮。

而是程序把上一轮内容重新传给了它。

这让我理解了一个之前很容易误解的地方：

**LLM API 本身通常是 stateless 的。**

所谓“模型记得之前发生过什么”，很多时候只是应用层把过去的对话和状态保存下来，并在下一次调用时重新发送。

因此：

`prompt_history`

其实就是这个最小 Agent 里非常原始的一种 memory。

---

## 8. 目前我理解的 Agent 核心

跑完这一轮以后，我会把最小 Agent 理解成四个东西：

**Planner**

LLM 判断下一步干什么。

**Tool Registry**

告诉系统有哪些能力可以使用。

**Executor**

Python 根据 LLM 的 Action 真正执行对应 Tool。

**Memory / State**

保存之前发生过什么，让下一轮决策能够基于过去的 Observation。

然后把它们放进一个循环：

Observe
→ Think
→ Act
→ Observe again

直到任务完成。

我以前看到“Agent”这个词，会觉得它似乎是某种非常复杂的新 AI。

但把代码拆开以后，它其实没有那么神秘。

最小版本就是：

**让 LLM 不只是回答问题，而是能够观察当前状态、选择行动、获取外部反馈，然后继续决策。**

---

## 9. 这一轮最大的收获

我觉得这次最重要的收获不是“终于成功运行了一个旅行助手”。

旅行助手本身没有什么特别的。

真正有价值的是，我开始能区分：

* LLM 和 Agent；
* Action 和真正的函数执行；
* Tool 和模型；
* 搜索和 LLM；
* RAG 和 Agent；
* API 和 API Key；
* 配置和代码；
* 模型记忆和应用层保存的 history；
* Tool 的接口和 Tool 内部的具体实现。

以及最重要的一点：

**Agent 的关键不在于模型一次做对所有事情，而在于它能够在环境反馈之后继续调整行为。**

第一次运行完整程序时，模型甚至第一步就调用错了工具。

但它没有因此彻底失败。

它得到了一个真实 Observation：

“这个工具不存在。”

然后重新规划，第二次做对了。

这个错误反而比一次性完美运行更让我理解 Agent。

因为它把 Agent 和普通“输入一句话、输出一句话”的 LLM 区别直接暴露了出来：

**它不是只生成答案，而是在一个环境中持续采取行动。**