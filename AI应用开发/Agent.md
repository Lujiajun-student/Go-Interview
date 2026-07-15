# 1. 什么是Agent

Agent时能自主决策和行动的AI。

Agent = LLM + Tools + Memory + Planning。

# 2. ReAct

ReAct就是思考、行动、观察。

Agent首先思考问题，接着调用工具，观察工具返回结果，思考是否需要继续调用，不需要就返回。

# 3. Function Calling的原理

Function Calling就是让LLM调用外部函数。

首先需要定义工具的Schema，创建Agent时注册工具，Agent返回function_call时解析，执行工具，得到结果，然后再次调用Agent。

# 4. 如何防止Agent陷入死循环

可以在代码中规定最大循环次数，达到上限就强制停止。

或者可以工具去重，如果agent重复调用同一个工具，代码中可以拒绝执行。

也可以引入超时机制，超过时间就强制停止。

# 5. 多Agent怎么协作

首先，用户给出一个需求。多Agent中，需要有一个协调者或规划者，需要能够将需求分解成多个小任务，将小任务交给其他Agent来执行。

# 6. 设计过什么Agent

代码Agent。能够输出指定格式的代码。还有检索Agent，能够搜索网上的论文，并保存到本地，还有学术Agent，能够解析论文，阅读里面的方法论，写出伪代码。等。

# 7. Agent的memory怎么设计

短期记忆用的是State，通过LangGraph的State来保存记忆。在AgentState里，会保存当前的对话历史，用户问题，下一步计划，最终输出等。所以每一次交互的结果都会存到state中，agent每次调用时都会携带这个state来保持记忆。