# Task03｜Memory & Retrieval：从“让 Agent 记住”到“让 Agent 找对信息”

本次任务学习的是 Datawhale `hello-agents` Chapter 8：**记忆与检索（Memory & Retrieval）**。

刚开始看到这一章时，我对 Memory 的理解非常朴素：Agent 能不能“记住以前聊过什么”。但真正做完以后，我发现这一章解决的问题远不只是保存聊天记录。

它实际上在回答一整组系统问题：

> 什么信息值得保存？保存成什么？保存在哪里？什么时候应该重新找到它？如何从几十万甚至几百万条信息中找到相关内容？找到以后怎么判断哪些值得交给 LLM？Memory、RAG 和 Agent 又分别负责什么？

最终，我把这一章理解成两条核心数据路径：

```text
Write Path
新信息
→ Encoding
→ Classification / Parse / Chunk
→ Embedding
→ Storage / Index
→ Consolidation / Forgetting
```

以及：

```text
Read Path
当前问题
→ Route
→ Memory / RAG Retrieval
→ Candidate Pool
→ Reranking
→ Context
→ LLM Generation
```

前者负责“**怎么记**”，后者负责“**怎么想起来**”。

---

## 1. History 不等于 Memory

Task02 中的 Agent 已经有 `_history`，所以我最开始很自然地问：

> 那 History 和 Memory 到底有什么区别？

`History` 更接近当前 Agent 实例中的对话记录：

```text
user
assistant
user
assistant
...
```

它通常是有顺序的、短期的，并且跟当前程序实例绑定。

而 Chapter 8 的 Memory 更像一个真正的信息管理系统。一个 Memory item 可以包含：

```text
content
memory_type
timestamp
importance
metadata
embedding
...
```

它可以被分类、检索、强化、遗忘、整合，部分类型还可以持久化。

因此：

```text
History
= 当前对话记录

Memory
= 可管理、可检索、可能跨会话存在的信息资产
```

这也是我第一次真正意识到：

> “模型记得上一句话”与“系统拥有长期记忆能力”是两件完全不同的事。

---

## 2. 四种 Memory：不是四个数据库，而是四种信息角色

HelloAgents 将 Memory 分成：

```text
Working Memory
Episodic Memory
Semantic Memory
Perceptual Memory
```

### Working Memory

Working Memory 是当前任务正在使用的短期信息。

例如：

```text
当前正在分析第 3 个候选策略
下一步需要检查最大回撤
刚刚 Tool 返回了一组结果
```

它的特点是容量有限、可能有 TTL、主要存在于运行时。

我实际测试了：

```python
MemoryTool(
    user_id="task03_user",
    memory_types=["working"]
)
```

在一个实例中加入 Working Memory 后，新建同一个 `user_id` 的另一个实例，记忆仍然为空。

这说明：

> 相同 `user_id` 并不意味着 Working Memory 自动跨实例共享。

Working Memory 和普通 History 的区别也不在于“是否持久化”，而是前者开始具备结构化属性，例如 importance、TTL、检索等。

---

### Episodic Memory

Episodic Memory 记录的是：

> **发生过什么。**

例如：

```text
昨天修复过一次 PyTorch DLL 错误
某次回测中策略在高波动阶段失效
用户曾经加载过某份研究报告
```

它强调事件、时间和上下文。

可以把它理解为：

```text
Event + Context + Timeline
```

---

### Semantic Memory

Semantic Memory 更像：

> **我知道什么。**

例如：

```text
该策略对高波动 regime 比较敏感
用户长期偏好某种风险水平
Cross-Encoder 是一种 query-document 联合编码模型
```

这里有一个对我非常重要的区分：

> **Episodic 是 evidence，Semantic 是 abstraction。**

例如：

```text
Episode 1：
高波动行情中策略失效

Episode 2：
高波动行情中再次出现大回撤

Episode 3：
类似市场状态下再次失效

↓ consolidation / abstraction

Semantic：
该策略对高波动 regime 敏感
```

Semantic Memory 因此并不是“一条发生过的事情”，而更接近从多个事件中抽象出来的稳定知识。

---

### Perceptual Memory

Perceptual Memory 对应的是：

> **我感知到了什么。**

例如图片、音频、视频、OCR、图表、传感器输入等。

完整的 Perceptual Memory 可能包含：

```text
原始媒体文件 / 文件引用
+
embedding
+
OCR / transcript / description
+
metadata
```

但这里有一个值得注意的点：HelloAgents Chapter 8 的 demo 所谓“图片/音频记忆”，实际上主要是用文本描述和 metadata 模拟，并没有真正对图像像素或音频波形做 multimodal embedding。

真正的多模态系统更可能是：

```text
Image
→ Vision Encoder / OCR / Caption
→ embedding + description

Audio
→ ASR / Audio Encoder
→ transcript + embedding
```

原始大文件通常也不会直接塞进 Memory DB，而是放在文件系统或对象存储中，Memory 保存它的引用、embedding 和结构化特征。

---

## 3. Memory Lifecycle：记忆不是 Add 完就结束了

这一章的 Memory 生命周期包括：

```text
Encoding
Storage
Retrieval
Consolidation
Forgetting
```

真正让我开始理解 Memory Engineering 的，是后面两个。

### Forgetting

我构造了四条 importance 不同的 Memory：

```text
0.20
0.40
0.70
0.95
```

然后执行 importance threshold = `0.5` 的遗忘策略。

结果：

```text
0.20 → 删除
0.40 → 删除
0.70 → 保留
0.95 → 保留
```

这让我意识到：

> Agent Memory 并不是“存得越多越好”。

如果所有信息永久存在，Memory 会越来越嘈杂，Retrieval 质量也会下降。

---

### Reinforcement

接下来我测试了重复出现的信息。

例如：

```text
用户喜欢无糖咖啡
```

初始 importance：

```text
0.30
```

重复强化后：

```text
0.45
→ 0.60
→ 0.75
→ 0.90
```

而一次性的：

```text
用户今天路过了一家花店
```

仍然保持低 importance。

这证明 Memory 可以基于重复出现的信息强化。

不过也产生了一个重要警告：

> repetition ≠ truth。

错误的信息重复十次也还是错误的。

因此更成熟的 Memory system 应该区分：

```text
importance
confidence
source
last_verified
```

而不是把“被重复过很多次”直接等价成“是真的”。

---

## 4. Memory 分类：关键词规则很容易骗自己

我先实现了一个 rule-based classifier，将输入分类成：

```text
Working
Episodic
Semantic
```

在一个简单测试集上，它达到了：

```text
8 / 8
```

看起来很好。

于是我专门构造了 adversarial examples。

结果：

```text
0 / 7
```

原因很简单：规则只是看关键词，而不是理解命题。

例如：

```text
“当前临时任务是记录‘长期偏好’四个字”
```

里面出现了“长期偏好”，但它描述的仍然是**当前临时任务**。

这让我重新碰到了 Task01 中 Parser 类似的问题：

> surface form ≠ semantics。

随后我使用 LLM classifier，效果明显更好，但也并非绝对正确。

最终做了一个 Hybrid：

```text
高置信度规则
↓
明显情况直接分类

其余情况
↓
LLM semantic judgement

↓
Parser
↓
Validator
↓
Working / Episodic / Semantic
```

并加入 retry 来处理 LLM 返回 `None` 或格式异常的问题。

这个过程最终形成了一个很重要的概念：

> **Memory Policy**

即系统应该明确设计：

```text
什么应该写入？
写入哪种 Memory？
importance 是多少？
是否强化？
什么时候整合？
什么时候删除？
```

---

# 5. Persistence 与 Semantic Retrieval 是两件事

这是本章中我认为最容易混淆的一点。

我首先用 SQLite 做持久化测试。

程序第一次运行：

```text
写入 Memory
```

程序退出。

第二次重新运行：

```text
仍然能够通过 ID 找到之前的数据
```

因此：

> **Persistence = 程序死了，数据没死。**

但是 SQLite 能保存数据，并不代表它能理解：

```text
“我最终想做什么项目？”
```

和：

```text
“I want to build a quantitative trading simulation sandbox.”
```

在语义上相关。

所以需要把两个问题拆开：

```text
SQLite
解决：
数据是否还存在？

Embedding + Vector Search
解决：
哪条数据和当前问题语义相关？
```

---

# 6. Embedding：文本为什么可以被“搜语义”

本章使用：

```text
sentence-transformers/all-MiniLM-L6-v2
```

它把一个句子编码成：

```text
384-dimensional vector
```

需要特别说明：

> 384 个维度不是 384 个可以人工解释的“意义”。

它们是模型学出来的 latent distributed representation。

内部大致流程为：

```text
Text
↓
Tokenizer
↓
Tokens / Subwords
↓
Transformer
↓
每个 token 得到 contextualized 384-D vector
↓
Mean Pooling
↓
一个 sentence embedding
```

例如：

```text
"I like coffee."
```

经过 tokenizer 后可能成为：

```text
[CLS]
i
like
coffee
.
[SEP]
[PAD]
...
```

Batch 中为了让句子长度一致，会加入 `[PAD]`。

`attention_mask`：

```text
1 = 有效 token
0 = padding
```

Pooling 时 `[PAD]` 不参与平均。

最终：

```text
[token1 vector]
[token2 vector]
[token3 vector]
...
↓ dimension-wise mean pooling
[sentence vector]
```

---

## 7. Normalize、Dot Product 与 Cosine

之后我继续拆了向量相似度。

向量长度：

$$
||A||=\sqrt{\sum a_i^2}
$$

Dot Product：

$$
A\cdot B=||A||||B||\cos\theta
$$

因此 raw dot product 同时受到：

```text
向量长度
+
方向
```

影响。

Cosine：

$$
\cos\theta=
\frac{A\cdot B}
{||A||||B||}
$$

把长度影响消掉，只比较方向。

Sentence Transformer 常使用：

```python
F.normalize(...)
```

将向量长度归一成：

```text
1
```

因此 normalize 后：

```text
dot product ≈ cosine similarity
```

我实际打印了：

```text
length ≈ 1
dot ≈ 0.6484794
cos ≈ 0.64847946
```

微小差异来自浮点数精度。

同时还有一个非常重要的结论：

> **Similarity score 不是 probability。**

`0.8` 不代表“80% 相关”。

---

# 8. Embedding 主要识别“语义接近”，并不等于逻辑一致

我做了一个专门的 negation attack：

```text
I prefer drinks without sugar.

I hate drinks without sugar.
```

对于：

```text
“What kind of drinks do I prefer?”
```

两句话都获得了很高的 similarity。

同样：

```text
I want to build a quantitative trading simulation sandbox.

I do not want to build a quantitative trading simulation sandbox.
```

也会在 vector space 中彼此靠得很近。

因此：

> Embedding similarity 更接近“主题/语义相关性”，而不是“逻辑立场是否一致”。

Vector Retrieval 能回答：

> 哪些内容值得进一步检查？

但不能独立回答：

> 哪些内容是真的？

---

# 9. Top-K、Threshold、Precision、Recall 与 F1

Retrieval 不能只问：

> 分数最高的是谁？

还需要决定：

```text
取几个？
分数低于多少不要？
```

于是进入：

```text
Top-K
Threshold
```

并进一步引出：

$$
Precision =
\frac{TP}{TP+FP}
$$

回答：

> 捞上来的结果有多干净？

以及：

$$
Recall =
\frac{TP}{TP+FN}
$$

回答：

> 所有真正相关的内容，我捞回来了多少？

F1：

$$
F1=
\frac{2PR}{P+R}
$$

将 Precision 与 Recall 压缩成一个数，但它会丢失 trade-off information。

因此：

> 一个 F1 分数不能替代完整 PR curve。

---

## 10. Retrieval 与 Signal Detection Theory

这一段和 Signal Detection Theory 几乎完全同构：

```text
similarity score ≈ evidence strength
threshold ≈ criterion

TP ≈ hit
FP ≈ false alarm
FN ≈ miss
TN ≈ correct rejection
```

当 threshold 改变时：

```text
threshold t
↓
Precision(t)
Recall(t)
```

于是产生 PR curve。

这里我还纠正了一个很容易混淆的概念：

```text
threshold
= 控制参数

operating point
= 该 threshold 产生的
  (Recall, Precision) 坐标
```

两者不是同一个东西。

在极度 class imbalance 的 Retrieval 场景中，PR 往往比 ROC 更直观，因为即使 FPR 很低，也可能因为 negative 数量巨大而产生大量 false positives。

---

# 11. 为什么需要 Vector Database

最简单的 Vector Search 可以直接：

```text
Query Vector
vs
所有 Memory Vector
```

逐个算 cosine。

我测试了 brute-force search：

```text
1,000 vectors
10,000
100,000
500,000
```

规模增大后，扫描成本明显上升。

第一次还因为：

```python
np.random.randn(...)
```

默认先创建 float64，再 `.astype(float32)`，导致瞬时内存暴涨。

改成：

```python
rng.standard_normal(
    shape,
    dtype=np.float32
)
```

后才顺利运行。

这也是一次很实际的 lesson：

> “最终数组只有多大”与“构造过程瞬间需要多少 RAM”不是一回事。

---

# 12. ANN 与 HNSW

为了避免每次扫描全部向量，Vector DB 通常使用：

```text
ANN
Approximate Nearest Neighbor
```

核心思想：

> 不保证检查所有点，而是用索引快速找到“足够可能”的邻居。

HNSW 是常见方案：

```text
Hierarchical
Navigable
Small
World
```

它是一个**多层图结构**，不是树。

重要特点：

```text
Layer 0
所有节点

Layer 1
较少节点

Layer 2
更少

...
```

进入高层并不是因为节点“更重要”，而通常来自随机/概率式 promotion。

高层负责快速长距离跳转，低层负责精细搜索。

---

## 13. Greedy Search 为什么会掉进 Local Minimum

我构造了一个故意骗人的 graph：

```text
A
├→ B → C → D
└→ X → Y → G
```

目标点是 G。

但是：

```text
A → B
```

看起来越来越接近目标。

走到 D 后，却发现所有邻居都比 D 更远。

Greedy Search 就停在：

```text
D
```

但真正最近的是：

```text
G
```

这证明：

> 有时候想达到 global best，必须暂时走向一个“看起来更差”的方向。

随后引入 candidate pool：

```text
同时保留多个候选路径
```

就可以在 B 路径走不通后回头继续探索 X。

这让我真正理解了 ANN 中 search breadth / budget 的意义，而不是只记一个 `ef_search` 参数名。

---

# 14. SQLite 与 Qdrant 的角色

最终完整 Memory 更接近：

```text
同一个 memory_id
```

在 SQLite 中：

```text
content
user_id
importance
timestamp
metadata
...
```

在 Qdrant 中：

```text
embedding vector
payload
memory_id
```

因此：

```text
MiniLM
= 翻译
Text → Vector

Qdrant
= 找人
哪个 ID 在向量空间最接近？

SQLite
= 查档案
这个 ID 对应的完整记录是什么？
```

我实际完成了：

```text
自然语言 Query
↓
MiniLM
↓
Qdrant
↓
memory_id
↓
SQLite
↓
完整 MemoryItem
```

整个链条。

---

## 15. Metadata Filter：相关不等于允许被检索

Qdrant 还可以先做：

```text
metadata filter
```

然后再 vector ranking。

例如：

```text
importance >= 0.9
```

会先缩小候选集合。

实验中：

```text
sugar-free memory
similarity = 0.6239
importance = 0.80
```

虽然语义非常相关，但因为 filter：

```text
importance >= 0.9
```

它直接被排除。

因此：

> **Filter 决定谁有资格参加比赛；Similarity 决定参赛者之间怎么排序。**

也因此，importance 并不适合永远作为 hard filter。

像：

```text
user_id
memory_type
time range
```

更自然地适合 hard filtering。

---

# 16. Memory 与 RAG 不是同一个东西

这是这一章另一个非常重要的边界。

Memory：

```text
关于用户、Agent、自身历史的信息
```

RAG Knowledge Base：

```text
外部资料、文档、知识库
```

例如：

```text
“我之前学过 Cross-Encoder”
→ Memory

“Cross-Encoder 的技术原理是什么”
→ RAG
```

RAG 本身是：

```text
Retrieval
+
Augmented Context
+
Generation
```

Memory 只是**可能成为 Retrieval source 之一**。

因此可以存在：

```text
RAG over documents
RAG over memory
RAG over database
...
```

但：

> **Memory ≠ RAG。**

---

# 17. Document RAG：Parse → Chunk → Embed → Retrieve → Generate

完整 Document RAG：

```text
Document
↓
Parse
↓
Text / Markdown
↓
Chunk
↓
Embedding
↓
Vector Store
↓
Retrieval
↓
Context
↓
LLM
```

这里也终于统一了我之前频繁见到的 `parse`：

> Parse 的本质，是把当前表示转换成下一阶段程序可以明确操作的结构。

因此：

```text
PDF parser
LLM output parser
JSON parser
HTML parser
Python parser
```

本质上都是同一个抽象概念。

---

# 18. OCR 不等于 Chunking，也不天然消耗 LLM Token

对于扫描 PDF：

```text
Pixels
↓
OCR
↓
Text
```

OCR 解决：

> 图片里写了什么？

而 RAG Chunking 解决：

> 哪些文字应该作为同一个检索单元？

这是完全不同的步骤。

本地 OCR 本身通常不需要 LLM token。

如果：

```text
OCR whole PDF
↓
Local chunk
↓
Local embedding
↓
只把 Top-K text 给 LLM
```

通常会比把整份 PDF 直接塞给 multimodal LLM 更节省模型输入。

---

# 19. Chunking：切得越小并不一定越好

我做了三个 chunk size 实验。

小 Chunk：

```text
archive access code is cobalt-seven and
```

Similarity 最高：

```text
0.7956
```

但句子是不完整的。

Medium Chunk：

```text
The archive system stores...
The archive access code is cobalt-seven...
```

分数稍低：

```text
0.7444
```

但信息完整。

Large Chunk：

```text
整篇多主题内容
```

只有：

```text
0.6195
```

因为混入了更多 semantic noise。

因此 Retrieval 质量不能只看 similarity。

还要看：

```text
Retrieval relevance
Context completeness
Context efficiency
```

最高 cosine 并不一定等于最好的 Context。

---

## 20. Chunk Overlap

我还故意把一句关键内容切断：

```text
Chunk 0:
The archive access code is

Chunk 1:
cobalt-seven and must be entered...
```

结果：

```text
Chunk 0
像问题，但没有答案

Chunk 1
有答案，但丢了语境
```

加入 overlap 后：

```text
The archive access code is cobalt-seven...
```

完整关系重新出现在同一个 Chunk 中，similarity 大幅提高。

因此：

> overlap 的作用是降低重要语义关系被 chunk boundary 切断的概率。

但 overlap 太大也会增加：

```text
重复 embedding
重复存储
重复 retrieval
重复 LLM context
```

所以它也是 trade-off。

---

# 21. HNSW adjacency ≠ Document adjacency

一个特别容易产生的错觉是：

> “Chunk 17 和 Chunk 18 都在 Qdrant 里，所以检索到 17 后应该顺便能找到 18。”

不一定。

HNSW 中的邻接关系是：

```text
vector-space navigation
```

不是：

```text
original document order
```

如果需要文档邻居扩展，应该显式保存：

```text
doc_id
chunk_index
page
section
```

然后在检索到 chunk 17 后由程序获取：

```text
16
17
18
```

这叫 neighbor/context expansion。

---

# 22. RAG 并不能自动消灭 Hallucination

第一次完整 RAG：

```text
Query
↓
正确检索到 quant sandbox
↓
LLM
```

最终模型虽然回答了正确项目，却自己补了一句：

> PyTorch DLL 问题“可能和项目依赖有关”。

Context 并没有提供这个因果关系。

因此：

> **Retrieval correct ≠ Generation grounded。**

后来加强 prompt：

```text
Only use facts explicitly stated in context.
```

面对：

> “Why do I want to build this project?”

虽然检索结果非常相关，但 Context 并没有提供“为什么”。

模型才正确回答：

```text
I don't know based on the provided context.
```

这让我得到一个非常重要的二分：

```text
Retrieval relevance
≠
Answer sufficiency
```

资料与问题“相关”，不代表资料“足够回答问题”。

---

# 23. MQE：换几个搜法提高 Recall

MQE：

```text
Multi-Query Expansion
```

原始问题：

```text
How can I stop a neural network
from memorizing the training set?
```

普通 retrieval 第一名居然是：

```text
Batch normalization
```

真正直接相关的：

```text
Dropout + weight decay
```

甚至没进 Top-3。

LLM 将 Query 改写成：

```text
How to prevent neural network overfitting
and improve generalization?
```

以后：

```text
Dropout + weight decay
```

立刻变成第一名。

因此：

> MQE 不是让 Retriever 本身变聪明，而是给 Retriever 多几个搜索入口。

它主要改善 Recall。

---

# 24. HyDE：先假装知道答案，再拿假答案去找真资料

HyDE：

```text
Hypothetical Document Embeddings
```

流程非常反直觉：

```text
Question
↓
LLM 先生成一段“可能的答案”
↓
Embedding hypothetical answer
↓
拿它去搜索真实 Document
```

原 Query 下：

```text
Dropout + weight decay
score = 0.3214
rank = 4
```

HyDE 后：

```text
score = 0.7080
rank = 1
```

因为：

```text
Question language
“How can I...?”

↓ HyDE

Document-like language
“regularization / dropout / weight decay...”
```

Hypothetical Document 和真实知识库文档在表达形态上更接近。

但这个假答案：

> 不应该直接作为最终事实交给用户。

它只是一个：

```text
search probe
```

---

# 25. Reranking：召回来，不代表排得对

Vector Retrieval 常采用 Bi-Encoder：

```text
Query
→ vector A

Document
→ vector B

cosine(A, B)
```

好处是 document embedding 可以预计算，非常快。

Cross-Encoder 不一样：

```text
[CLS]
Query
[SEP]
Document
[SEP]

↓ same Transformer

Query tokens ↔ Document tokens
通过 self-attention 直接互动

↓
一个 relevance score
```

它更精细，但代价更大。

因此常见架构：

```text
百万 chunks
↓
Fast Retriever
↓
Top 50

↓
Cross-Encoder
↓
Top 5
```

Retriever 偏：

```text
Recall
```

Reranker 偏：

```text
Precision
```

---

## 26. 更复杂的模型并不保证一定更正确

我的 Cross-Encoder 实验也出现了一个很有意思的失败。

原 Query：

```text
stop a neural network
from memorizing the training set
```

Cross-Encoder 竟然把：

```text
Dropout + weight decay
```

排到最后。

换成更规范的：

```text
How can I prevent neural network overfitting?
```

结果明显改善。

于是出现：

```text
Query Normalization
```

即：

```text
用户自然表达

↓ rewrite

Canonical Query
```

例如：

```text
stop memorizing training set

↓

prevent neural network overfitting
```

`Canonical Query` 就是系统选择的一个更标准、更稳定的代表性表达。

而：

> robust / 鲁棒性

在这里指：

```text
用户只是换一种说法
↓
模型结果是否仍然稳定
```

如果同义改写就让模型排名剧烈变化，则模型对 wording 不够 robust。

---

# 27. Advanced Retrieval 不是“模块越多越好”

最终我把：

```text
Original Query
MQE
HyDE
Vector Retrieval
Candidate Pool
Cross-Encoder
```

全部串起来。

结果发生了一个非常真实的 pipeline failure：

```text
MQE
帮我找到 Dropout

HyDE
也帮我找到 Dropout

↓

Candidate Pool
正确资料已经在里面

↓

Cross-Encoder
重新用自己不擅长的原始 wording 精排

↓

Dropout 又被排出去
```

因此：

> **后置模块完全可能把前置模块辛苦修好的结果重新搞坏。**

这个结论和 Task01 的 Reflection 非常像：

```text
Reviewer
也可能把正确答案改坏
```

Agent / RAG engineering 不是：

> 模块越多，系统必然越强。

而是：

> 每增加一个模块，同时增加一个潜在增益点和一个潜在 failure point。

---

# 28. Memory + RAG + Agent：真正开始成为系统

最后，我做了一个 mini Agent Router。

问题可能被分流为：

```text
MEMORY
RAG
BOTH
DIRECT
```

例如：

```text
“What final project do I want to build?”
→ MEMORY

“How does a Cross-Encoder work?”
→ RAG

“结合我的最终项目，reranking 可以放在哪里？”
→ BOTH
```

这里我发现四选一分类也有问题，因为：

```text
Memory need
RAG need
```

其实不是互斥的。

于是改成：

```text
MEMORY = YES / NO
RAG = YES / NO
```

再由程序组合：

```text
YES + NO → MEMORY
NO + YES → RAG
YES + YES → BOTH
NO + NO → DIRECT
```

这是一种更合理的 multi-label routing。

---

# 29. Router、Retriever、Reranker、Generator 都会错

这个 Agent 实验最终让我看到了一条非常统一的规律。

Router 会错：

```text
本该 BOTH
却判断成 MEMORY
```

Retriever 会错：

```text
相关文档没进入候选
```

Reranker 会错：

```text
正确候选被重新排后
```

Generator 也会错：

```text
Context 没写
模型却自己补出很多事实
```

因此：

> Agent Engineering 不是把一串“聪明模型”连起来。

而是：

```text
明确每个模块职责
+
明确每个模块可能怎么错
+
控制错误如何传播
```

---

# 30. Parametric Knowledge、RAG Knowledge 与 Web Search

最后一个非常重要的区分来自 LLM 自己“开始自由发挥”。

Context 只有：

```text
quantitative trading simulation sandbox

reranking =
candidate pool → expensive reranker

Qdrant = vector database
```

但模型自己写出了：

```text
Sharpe ratio
Sortino ratio
maximum drawdown
portfolio construction
momentum
volatility
...
```

这些并没有来自我们的 RAG。

它也没有联网。

而是来自：

```text
Parametric Knowledge
```

也就是模型训练阶段学进参数里的知识。

因此：

```text
RAG Knowledge
= 本次运行时从外部知识库检索进 Context

Parametric Knowledge
= 模型训练时已经学会的知识

Web Search
= 本次运行时真正访问互联网取得的信息
```

这三者必须区分。

---

# 31. Grounding Policy：给了 Context，不代表模型只会使用 Context

我最后加入了两种回答模式。

STRICT：

```text
只允许使用 Context 中明确支持的事实
```

ASSISTED：

```text
Context 作为事实基础
+
允许模型使用自己的参数知识做分析和建议
```

但实验中甚至 STRICT prompt 仍然挡不住 LLM 自己补出：

```text
Sharpe ratio
Sortino ratio
portfolio construction
...
```

因此又得到一个非常现实的结论：

> **Prompt 是行为指导，不是权限边界。**

真正需要严格 Grounding 的系统还需要：

```text
retrieval
↓
generation
↓
claim validation
↓
citation / source checking
↓
unsupported claim rejection
```

而不是只写一句：

```text
Use ONLY the context.
```

---

# 32. 最终形成的 Chapter 8 心智模型

这一章最终可以压缩成两个系统。

Memory System：

```text
Information
↓
Encoding
↓
Memory Policy
↓
Working / Episodic / Semantic / Perceptual
↓
Storage
↓
Reinforcement / Consolidation / Forgetting
↓
Retrieval
```

RAG System：

```text
Document
↓
Parse / OCR
↓
Chunk
↓
Embedding
↓
Vector Store

User Query
↓
Query Transformation
MQE / HyDE / Normalization
↓
Retriever
↓
Candidate Pool
↓
Reranker
↓
Context
↓
LLM
```

Agent 再站在上面：

```text
                Memory
                   ↑
                   │
User → Agent → Router
                   │
          ┌────────┴────────┐
          ↓                 ↓
        Memory             RAG
          ↓                 ↓
          └──── Context ────┘
                   ↓
                  LLM
                   ↓
                Action
```

---

# 33. 对最终项目的意义

我的最终课程项目计划是：

> **Quantitative Trading Simulation Sandbox**

Chapter 8 对它的价值已经非常清楚。

Memory 可以负责：

```text
之前测试过什么
用户风险偏好
策略历史修改
过去的异常事件
长期形成的策略经验
```

RAG 可以负责：

```text
交易所 API 文档
研究报告
策略说明
风险规则
论文
外部知识库
```

Structured Data Source 则负责：

```text
价格
成交量
订单
position
PnL
drawdown
```

Agent 决定：

```text
当前任务需要：
Memory？
RAG？
Market Data？
Calculator？
Jev？
```

Python 则负责：

```text
确定性计算
风险限制
模拟撮合
账户状态
PnL
```

最终结构逐渐变成：

```text
Market Data
     ↓
Feature Engine
     ↓
Market State
     ↓
Agent / Decision Layer
 ├─ Memory
 ├─ RAG
 ├─ Tools
 └─ Jev
     ↓
Python Risk Rules
     ↓
Virtual Exchange
     ↓
Position / PnL / Logs
```

Chapter 8 因此不是单独给 Agent 加了一个“记忆功能”。

它真正补上的，是：

> **Agent 如何拥有可持续的状态、外部知识和可检索的信息环境。**

---

# Task03 Conclusion

完成这一章以后，我对 Agent 的理解从：

```text
LLM + Tools
```

进一步变成：

```text
Agent
=
LLM
+ Tools
+ Runtime
+ State
+ Memory
+ External Knowledge
+ Retrieval
+ Routing
+ Verification / Grounding
```

如果 Task01 的核心是：

> **让 LLM 进入 Action Loop。**

Task02 的核心是：

> **把这个 Loop 组织成 Framework。**

那么 Task03 的核心就是：

> **让这个 Framework 开始拥有时间、状态和知识。**

<img width="1347" height="1095" alt="1c360301-4aab-4318-b2b8-f6eba72a09cf" src="https://github.com/user-attachments/assets/29ed27c3-ee27-4312-b660-8909a5a66724" />

<img width="1347" height="1095" alt="6a1a7d5e-bde1-4d5b-9d2f-3e6af0387c15" src="https://github.com/user-attachments/assets/79f6f3db-8046-4970-a257-a2ce57a0d11e" />

<img width="1347" height="1095" alt="89a2cf9a-7ef2-4fcc-aa58-40b8490e0642" src="https://github.com/user-attachments/assets/c61c65f6-ca68-4916-810a-4a7d2069d994" />

<img width="1347" height="1095" alt="08cd44f1-5192-431c-88b4-fc715b53412f" src="https://github.com/user-attachments/assets/441e7e42-1c7f-4680-9811-0390ad602aeb" />

<img width="1347" height="1095" alt="db1100cb-cf65-472b-b435-36cfb1038333" src="https://github.com/user-attachments/assets/a188fce0-9fa3-401d-befe-da00cdf225a3" />

<img width="1347" height="1095" alt="482590fc-8d24-440a-ae5d-748b7cdbe020" src="https://github.com/user-attachments/assets/e595ef75-d920-4ef6-943c-94d1c0a8913f" />

[task03_memory_retrieval_knowledge_map.html](https://github.com/user-attachments/files/32602163/task03_memory_retrieval_knowledge_map.html)
