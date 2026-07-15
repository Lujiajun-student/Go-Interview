# 1. Prompt

Prompt就是提示词。

# 2. RAG

RAG的作用就是将知识存储到知识库，遇到问题时首先根据问题在知识库中检索最相关的知识，将这些知识拼接到问题中输入到agent，就能最大程度保证agent回答的准确性。

首先如果要将一个文档存储到知识库，就会将这个文档进行分片，检索时只会拿出与问题最相关的分片交给agent。

这里包含两个部分。

1. 预处理。在用户提问前，需要先准备好知识库。首先需要准备大量的文档并进行**分片**，再做好**索引**发送到向量库中。
2. 回答。在用户提问后首先进行**召回**，将相关的分片进行**重排**，最后再**生成**回答。

```
分片->索引->召回->重排->生成
```

`分片`。将文档分成多个片段，每个片段分别独立。

`索引`。通过Embedding将片段转化为向量，这样才能保存到向量数据库。

`召回`。搜索用户提问相关的片段。将用户的问题嵌入为向量，根据向量相似度算法，从向量数据库中返回最相似的N个结果。

`重排`。计算召回的所有片段的相似度，从中挑选更相似的结果进行返回。

`生成`。agent根据重拍后的片段和用户问题，最终生成与检索片段相关的答案。

## 1. 分片

分片是因为上下文长度限制，导致agent无法接收过大上下文。但agent又需要保证语义完整性。

因此，分片需要灵活分片。可以使用chunk_size控制分片大小，chunk_overlap保证有重叠，有语义连续。

分块有几种方式，分别是固定分块、语义分块、重叠分块、父子分块等。

1. 固定分块。

```python
# RAG分块策略
from langchain_text_splitters import CharacterTextSplitter

sample_text = (
    "LangChain was created by Harrison Chase in 2022. It provides a framework for developing applications "
    "powered by language models. The Library is known for its modularity and ease of use. "
    "One of its key components is the TextSplitter class, which helps in document chunking. "
)

# 定义分词器
text_splitter = CharacterTextSplitter(
    separator=" ", # 按空格分割
    chunk_size=100, # 定义块大小
    chunk_overlap=20, # 调整重叠比例
    length_function=len,
)

docs = text_splitter.create_documents([sample_text])

for i, doc in enumerate(docs):
    print(f"--- Chunk {i + 1} ---")
    print(len(doc.page_content))
    print(doc.page_content)
```

这里固定分块会导致文本的词组被分割，语义被破坏。

2. 递归字符。按照段落来分割，如`\n`。这是默认的分词器。

```python
# RAG分块策略
from langchain_text_splitters import CharacterTextSplitter

sample_text = (
    "LangChain was created by Harrison Chase in 2022. It provides a framework for developing applications "
    "powered by language models. The Library is known for its modularity and ease of use. "
    "One of its key components is the TextSplitter class, which helps in document chunking. "
)

# 定义分词器
text_splitter = CharacterTextSplitter(
    chunk_size=100, # 定义块大小
    chunk_overlap=20, # 调整重叠比例
)

docs = text_splitter.create_documents([sample_text])

for i, doc in enumerate(docs):
    print(f"--- Chunk {i + 1} ---")
    print(len(doc.page_content))
    print(doc.page_content)
```

这里对分词器不做限制，默认的分隔符就是`\n\n`, `\n`, `" "`, `""`。

3. 按句子分块。

这里需要nltk自然语言处理分句，分句后每次就逐渐叠加句子，直到准备超过chunk大小时，就能作为一个chunk，然后继续拼接其他句子。

4. 按标题层级分块。

像Markdown格式或者word格式，有标题层级，就可以根据层级来进行分割。

5. 代理分块。

让LLM理解文本，根据知识进行分块，每一块能够总结为一对QA。这样分块的质量比较高，但缺乏精准度，且成本高。

6. 语义分块。

让LLM理解文本，根据语义来切割文本。

## 2. 优化

即使使用上面的分块算法，依然会存在检索语义割裂等问题。因此，最常用的方法是使用父文档检索器。将文档切割成多个子文档，如果检索命中其中一个子文档，那么将整个父文档都返回。

或者让LLM根据分块来生成可能的多个问题，先让用户的问题与这里生成的问题库进行匹配，匹配上就取出对应的分块。这样能够为文档提供更多的语义，提升召回率。

### 1. RAG-Fusion

RAG-Fusion能够很好地提高召回率。

用户的提问不一定准确，可能出现语义或语法上的差异，导致原本应检索出来的文档无法检索。这样，可以让**LLM自动生成多个不同角度的问题，再把所有问题的检索结果融合起来。**

### 2. 假设性文档嵌入

假设性文档嵌入HyDE是用于提升检索效果的技术。思想是**不要直接用用户的问题做向量检索，而是让LLM写假设存在的答案，再根据这些假设答案进行检索。**

这是因为文档通常是以答案的形式存在，问题与答案的相关度不一定高，但假设性答案通常与真实答案的相似度更高。

### 3. 混合检索

通常来说，向量检索在处理专有名词、代码片段、ID编号等必须精确匹配的内容存在问题。这些内容无法通过语义来检索，需要使用其他的检索方法。

混合检索包含**稀疏检索**和**稠密检索**。

稀疏检索用于进行精确匹配，记录每个Token的出现次数，根据出现次数的差异来获取评分，返回最相似的前N个结果。主要的算法是BM25。BM25会考虑关键词出现次数、文档长度、关键词是否稀有。

稠密检索用于语义匹配。会将文本进行嵌入，从向量数据库中检索向量最相近的N个文档。这种方式对关键词不敏感，但能够感知语义，从结果中选取语义最相近的文档。有Qdrant、Chroma、Milvus等。

混合检索会在稀疏检索和稠密检索返回的N个答案中进行**融合**，选取前K个文档来进行返回。

这里的融合有很多种方法。

1. RRF。按照排名融合，不直接比较不同检索器的分数。
2. 加权融合。对BM25分数和向量相似度进行归一化后加权求和。
3. 学习排序。通过机器学习模型，经过训练后进行排序。
4. 重排序。召回这些检索的所有回答，通过更精确的模型进行评分和排序。

# 3. Agent

## 3.1. 规划

规划是Agent理解环境，分析任务并执行策略的前提，当我们面对任务时，需要先考虑任务怎么完成，评估资源是否充足，如何分解任务，任务执行中应怎么反思，执行到什么程度即可完成等。Agent中，最主要的规划能力有两种。

### 3.1.1. 任务分解

这是将复杂任务拆解为具体步骤的能力。

这能够降低任务复杂度，提高完成的概率，并且能够明确执行的路径，便于进行进度追踪等。

主要的实现方法是通过提示词，让LLM首先建立任务的层次结构，为每个子任务分配目标和完成标准。这里又需要使用LLM的推理能力。

#### 3.1.1.1. 思维链

思维链CoT是提示技术，引导LLM逐步输出推理过程，而不只是只给出最终答案。

在提示词中添加“think step by step”即可，LLM会自发将任务在思考中进行拆解，提高复杂任务的完成准确率。主要用于多步数学计算、逻辑推理问题、复杂决策等。

CoT有两种类型，分别是Zero-Shot和Few-Shot。Zero-Shot是让LLM自己进行推理，推理过程可能不可控。而Few-Shot是给出一些具体的推理示例，让LLM能够根据这些示例来对推理过程进行规范。

#### 3.1.1.2. 思维树

思维树ToT是CoT的拓展，在每个节点前生成多个可能的思考分支，选择最有可能的分支来进入下一步。

主要是要求LLM在关键任务节点中生成多个可能的下一步，对每个分支的优劣程度进行评分，可以选择广度优先或者深度优先的方法探索思维树。如果结果不满意，可以进行回溯，选择新的分支来进入下一步。

### 3.1.2. 反思

反思ReAct是让Agent对过去的行为进行反思，从中学习并改进未来的步骤，提升最终结果的质量。ReAct主要分为三个步骤：`思考-行动-观察`。每执行一步，都会观察结果是否符合预期，再思考接下来应该怎么做。好处是每次都能够想到最好的下一步。但缺点就是每次思考的都是局部最优，最终可能会偏离原来的任务。

## 3.2. 工具

工具是一类函数，能够让模型感知并使用，给LLM提供行动的能力。

工具能够提供Agent查阅新知识、新数据，或者操作数据库等能力。而使用工具需要编写skills，skills给出能够调用的工具的方法、描述、输入输出规范，并且能够规范agent的执行过程，编排agent的任务流程。

## 3.3. 记忆

智能体需要记忆来感知环境。

主要分为长期记忆和短期记忆两种类型。

1. 短期记忆指的是当前的上下文。当前会话的上下文会保留先前的问答，在给出下一次问题时会将先前的上下文拼接上去。
2. 长期记忆指的是可持久化存储的知识，通过向量数据库来保存。将对话历史、用户偏好、任务结果等存储到数据库，通过语义检索来召回相关的记忆。

## 3.4. 思考模式

### 3.4.1. ReAct

反思ReAct是让Agent对过去的行为进行反思，从中学习并改进未来的步骤，提升最终结果的质量。ReAct主要分为三个步骤：`思考-行动-观察`。每执行一步，都会观察结果是否符合预期，再思考接下来应该怎么做。好处是每次都能够想到最好的下一步。但缺点就是每次思考的都是局部最优，最终可能会偏离原来的任务。

### 3.4.2. Plan-and-Execute

Plan-and-Execute是一种分阶段的处理框架，将规划和执行分开。首先让一个LLM做规划，将任务分成多个简单的小任务，让其他LLM分别处理这些小任务，最后的规划LLM对这些小任务的处理结果进行处理，查看哪些小任务完成了。规划LLM能够根据小任务的结果来修改接下来的步骤，能够重新规划，确保最终的结果不会偏离原来的目标。

### 3.4.3. Reflection



### 3.4.4. Self-Ask

Self-Ask与ReAct、Plan-and-Execution不同级，Self-Ask是Agent内部的思考方式。它强制**agent面对复杂问题时，将复杂问题拆解为一系列可执行的子问题并依次回答，不是尝试直接生成最终答案。

1. 问题分解。agent获取用户指令后，先思考为了回答这个问题，首先需要知道什么。
2. 生成子问题。agent生成这个大问题，需要解决的哪些小问题。
3. 中间回答。按照顺序，逐步回答这些小问题。
4. 回答了一个小问题后，会根据需求继续追问问题，直到认为收集到足够信息。
5. 生成回答。基于所有的小问题和答案，推理并给出最终回答。

这个和ReAct不同，ReAct是思考需要调用什么工具，观察调用结果，Self-Ask是拆解复杂问题，每回答一个问题就更新接下来的问题。

与Plan-and-Execution也不同，PE是拆解成小任务后，交给其他agent或工具来解决，最终收集到所有的结果来汇总生成回答。Self-Ask是根据复杂问题，给出这个问题需要提前了解什么，生成对应的小问题，优先解决小问题后，根据所有小问题的答案来生成最终回答。

## 3.5. 实战

这是让agent自己实现一个小项目，自主放到某个文件夹。

下面就是通过提示词，让大模型输出指定格式的回答，根据回答判断是调用工具，还是最终回答。

```python
prompt_template = """
你需要解决一个问题。为此，你需要将问题分解为多个步骤。对于每个步骤，首先使用<thought>来思考需要做什么，然后通过<action>来调用工具，或者使用<final_answer>来展示最终答案。其中，调用工具时，会通过<observation>来展示工具或环境返回的结果，可以通过结果考虑是否能够生成最终答案，还是需要继续调用工具。
所有步骤请严格使用以下XML标签格式输出：
 - <question>用户问题
 - <thought> 思考
 - <action> 采取的工具操作
 - <observation> 工具或环境返回的结果
 - <final_answer> 最终答案

例子1：
<question>埃菲尔铁塔有多高？</question>
<thought>我需要找到埃菲尔铁塔的高度。可以使用搜索工具。</thought>
<action>get_height("埃菲尔铁塔")</action>
<observation>埃菲尔铁塔的高度约为330米。</observation>
<thought>搜索结果显示了高度。我已经得到答案了。</thought>
<final_answer>埃菲尔铁塔的高度约为330米。</final_answer>
"""
```

这样，可以通过action来调用工具、创建文件、修改代码等。最终能够实现这里的效果。

# 4. LangChain

LangChain能够解决LLM的缺点。

* LLM存储的知识是静态的，LangChain能够提供tools，给LLM检索的能力，获取最新信息。
* LangChain能够存储上下文或者用户偏好，让交互个性化。
* LangChain能够让Agent调用工具，对外部环境进行操作。
* 拆解任务，LangChain能够将用户的复杂需求进行拆解，分成多步来执行。

## 4.1. 记忆存储方式

agent需要实现记忆，也就是通过某种方法来保存上下文。

### 4.1.1. State

调用LangChain时，数据是从一个节点流向另一个节点的。也就是agent节点经过工具节点后，又会回到agent节点上。这里的agent会维护一个状态state，数据流出节点时，会同步更新这个状态，每次调用agent都会带上这个状态，保证agent知道当前任务的执行情况。

### 4.1.2. checkpoint

使用checkpoint能够将agent的state保存到内存中，agent会自动从存储的位置读取相关的记忆，这能够保证多轮对话时，能够指定不同的checkpoint，赋予agent不同的记忆。

```python
# 记忆通过Checkpoint来实现。
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware
from langchain_core.messages import HumanMessage
from langgraph.checkpoint.memory import InMemorySaver
from dotenv import load_dotenv

load_dotenv()

agent = create_agent(
    "deepseek-v4-flash",
    checkpointer=InMemorySaver(), # 指定内存保存器来存储记忆
    middleware=[
        SummarizationMiddleware(
            model="deepseek-v4-flash",
            trigger=("tokens", 20),
            keep=("messages", 20),
        ),
    ],
)

# 设置thread_id，作为会话标识
config = {"configurable": {"thread_id": "thread_1"}}

response = agent.invoke(
    {"messages": [HumanMessage(content="你好，我叫虎哥，我最喜欢猫。")]},
    config
)

print(response)

# 第二次会话时，再调用该thread_id即可
response = agent.invoke(
    {"messages": [HumanMessage(content="你好，你还记得我叫什么名字吗？")]},
    config
)
# 这里能够根据checkpoint找到会话记录作为记忆。
print(response)

```

这里使用InMemorySaver，用来保存agent的checkpoint。那么调用agent时，需要指定对应的thread_id。调用agent时，会根据传入的thread_id，读取存储对应的state，输入到agent。

默认情况下，InMemorySaver影响的是内存。但经过配置，可以保存到Redis或者数据库。

### 4.1.3. ConversationBufferMemory

ConversationBufferMemory能够保存完整的历史对话，不做压缩和筛选，每次调用时，都会将历史对话拼接到提示词，再输入到agent。

这个属于老式的记忆方法，

```python
# ConversationBufferMemory
from langchain_community.memory import ConversationBufferMemory
from langchain import ConversationChain

chat = OpenAI(model="deepseek-v4-flash")

memory = ConversationBufferMemory(memory_key="chat_history")
conversation = ConversationChain(llm=chat, memory=memory, verbose=False)
conversation.predict(input="你好，我是一个程序员。")
conversation.predict(input="你叫什么名字？")
```

> 这里ConversationBufferMemory似乎废弃，无法找到。

### 4.1.4. ConversationBufferWindowMemory

这个与ConversationBufferMemory类似，而ConversationBufferMemory保存的是完整的上下文，但ConversationBufferWindowMemory保存的是近K轮对话，确保上下文不会太长。

```python
from langchain.memory import ConversationBufferWindowMemory
memory = ConversationBufferWindowMemory(k=5)
```

### 4.1.5. ConversationTokenBufferMemory

这是用于指定上下文的Token数，系统会保留不超过限制的对话内容。

```python
from langchain.memory import ConversationTokenBufferMemory
memory = ConversationTokenBufferMemory(llm=chat, max_token_limit=100)
conversation = ConversationChain(llm=chat, memory=memory, verbose=False)

conversation.predict(input="你好，我是一个程序员。")
conversation.predict(input="你叫什么名字？")
```

### 4.1.6. ConversationSummaryBufferMemory

这是用于指定上下文的Token数，如果超过限制，就会将先前的上下文总结成摘要，同时尽量保留关键信息。

## 4.2. 文本嵌入模型

文本嵌入模型最主要的作用，是将文本转化为固定长度的向量表示。通过向量可以捕获语义，语义相似的文本的向量在空间中距离较近。
