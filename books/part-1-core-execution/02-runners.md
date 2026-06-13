# 第二章 Runner：请求是怎样真正跑起来的

上一章我们解决的是：

- Agent 是什么
- 为什么需要 Agent
- Agent 在 ADK Runtime 中处在什么位置

但到这里还差一个最关键的问题没有回答：

**这些 Agent 到底是怎么被真正跑起来的？**

如果说 Agent 是“干活的人”，那么这一章要讲的 `Runner`，就是“把这次活真正派发出去并盯着执行过程的人”。

所以这章可以看成第一章的自然续篇：

- 第一章解决“谁来做事”
- 第二章解决“这次请求如何把这些角色调度起来”

---

## 1. 先从真实场景切入：为什么 Agent 之外还必须有 Runner

假设你已经定义好了一个 `LlmAgent`：

- 它有名字
- 它有 instruction
- 它有 tools
- 它可能还有 sub-agents

现在用户发来一句话：

“帮我查一下 12345 号订单的状态，如果已付款，再告诉我能不能开票。”

这时候只靠 Agent 对象本身，其实还远远不够。

系统还必须回答这些运行时问题：

1. 这句话属于哪个用户、哪个 session？
2. 当前是不是一个新 invocation，还是上次被中断后的恢复执行？
3. 用户消息应该先写进 session，还是先跑 agent？
4. 如果这个 agent 上次转给了 sub-agent，这次应该继续谁？
5. 如果运行过程中不断产生 event，谁来消费这些 event？
6. 哪些 event 应该落盘，哪些只是流式输出？
7. before/after run 的 plugin 在哪里执行？
8. 整次运行结束后，谁来清理资源？

这些事情都不是 Agent 的职责。

所以 ADK 必须有一个专门的调度者：`Runner`。

---

## 2. Runner 到底是什么

先给一个适合零基础读者的定义：

**Runner 是一次请求级别的总调度器。**

这个定义里有三个关键词。

### 2.1 “一次请求级别”

Runner 关注的不是“系统全局配置中心”，也不是“某个 Agent 的内部逻辑”。

它关心的是：

**当一条新的用户请求到来时，这次调用从头到尾怎么被组织起来。**

### 2.2 “总调度器”

Runner 自己不负责思考内容，不负责生成答案，不负责实现工具。

它负责调度这些角色和系统部件：

- Session
- InvocationContext
- Agent 或 Node
- Plugin
- Event persistence
- Live / async execution path

这和 Web 框架里的控制器、调度器、runtime coordinator 有点像，但又更接近 Agent Runtime 自己的 orchestrator。

### 2.3 “不是业务角色”

这一点必须先钉死。

如果第一章的 Agent 是业务角色，那么 Runner 绝对不是。

你可以这样记：

- Agent 负责“做什么”
- Runner 负责“这次怎么跑”

这是全书里最值得反复记住的分层之一。

---

## 3. 先看源码入口：`Runner` 在哪里

关键源码：

Runner  
`src/google/adk/runners.py`

先不要一上来被这个文件的长度吓住。  
它确实很长，但对于学习者来说，并不是每一段都要第一次就看懂。

第一次阅读时，只抓这些入口：

- `class Runner`
- `run`
- `run_async`
- `run_live`
- `_get_or_create_session`
- `_new_invocation_context`
- `_append_user_event`
- `_consume_event_queue`
- `close`

你只要先把这几处之间的关系看顺，Runner 就已经理解一大半了。

---

## 4. 为什么 Runner 推荐接收的是 `App`

先看 `Runner` 的初始化。

Runner  
`src/google/adk/runners.py`

重点位置：

- `__init__`
- `_resolve_app`

如果你观察初始化逻辑，会发现 ADK 虽然允许传：

- `app`
- `agent`
- `node`

但它最终会尽量把入口统一成 `App`。

这说明 ADK 的设计倾向很明确：

**Runner 想调度的不是一个裸 Agent，而是一个完整的应用容器。**

为什么？

因为一次真实运行除了 root agent 之外，往往还需要：

- plugins
- context cache config
- resumability config
- event compaction config

这些能力如果都散落在参数里，Runner 会越来越难维护。  
所以它用 `App` 把运行所需的外围配置打包起来。

对读者来说，应该先建立一个判断：

**生产使用里，Runner 更像是在运行一个 App，而不只是单独运行一个 Agent。**

---

## 5. 先建立总执行链：一条请求进来之后发生什么

在正式拆代码前，我们先给出一条简化后的 Runner 主链路。

当你调用：

Runner  
`src/google/adk/runners.py`

方法：

- `run_async`

大致会发生下面这些步骤：

1. 校验并准备本次 `RunConfig`
2. 找到或创建 session
3. 判断这是新请求还是恢复请求
4. 创建 `InvocationContext`
5. 把用户消息写进 session
6. 决定这次该跑哪个 agent 或 node
7. 启动执行
8. 持续消费 event queue
9. 把该持久化的 event 写回 session
10. 把 event 流式 yield 给调用方
11. 运行 after-run 清理逻辑

你可以先把它记成一句话：

**Runner 的工作，就是把“用户消息”变成“一串被持久化、可流式消费、可恢复的事件执行过程”。**

这就是它为什么必须存在。

---

## 6. `run` 和 `run_async`：为什么生产要优先理解异步入口

关键源码：

Runner  
`src/google/adk/runners.py`

重点方法：

- `run`
- `run_async`

### 6.1 `run` 只是方便壳

如果你读 `run`，会发现它本质上是在后台线程里启动 `run_async`，再把异步事件转成同步生成器。

也就是说：

**`run` 不是核心执行模型，只是一个方便本地使用的同步包装。**

这对零基础读者非常重要，因为很多人看到 `run()` 就会自然地把它当成主入口。

但从架构上看，真正的核心入口是：

- `run_async`

### 6.2 为什么 `run_async` 才是主线

因为 ADK 的执行天然就是事件流模型，而不是一次性返回值模型。

一条请求运行中，可能不断产生：

- user event
- model event
- function call event
- function response event
- partial event
- final event

所以用异步生成器来承载这条链，才符合系统本质。

这也是你阅读 ADK 时很重要的一个认知切换：

**不要把 Runner 理解成“同步函数 -> 最终结果”，而要理解成“异步事件流调度器”。**

---

## 7. `run_async`：这是理解 Runner 的核心方法

关键源码：

Runner  
`src/google/adk/runners.py`

重点方法：

- `run_async`

第一次读这个方法时，不要纠结所有分支，先抓它在解决的几个核心问题。

### 7.1 它先决定“这次跑的是哪类根对象”

`run_async` 一开始就会判断 root 是什么：

- `LlmAgent`
- `BaseNode`
- 旧的 `BaseAgent` 路径

这说明 Runner 并不是只为单一聊天 agent 服务。  
它同时在兼容：

- 聊天式 root agent
- workflow node runtime
- 部分旧执行路径

所以 `run_async` 看起来分支多，不是因为写乱了，而是因为它站在整个运行时交汇点上。

### 7.2 它还要决定“当前到底该继续哪个 agent”

对零基础读者来说，这里特别值得停一下。

你可能会下意识以为：root agent 不就是固定跑那个 root 吗？

但在真实多 agent 系统里不是这样。

某次 invocation 可能：

- 已经转移给某个 sub-agent
- 正在等待某个 task-mode agent 恢复
- 需要根据历史 function call 找回正确执行者

所以 Runner 需要有一个重要能力：

Runner  
`src/google/adk/runners.py`

相关位置：

- `_find_agent_to_run`

它的职责不是“找一个配置对象”，而是：

**根据 session 历史和当前调用语义，决定本轮真正应该接手执行的是谁。**

这正是多 agent runtime 和普通函数调度的根本区别。

### 7.3 它必须分清“新请求”和“恢复请求”

Runner 还要区分两类运行：

1. fresh invocation
2. resumed invocation

关键位置：

Runner  
`src/google/adk/runners.py`

- `_resolve_invocation_id`
- `_resolve_invocation_id_from_fr`

为什么需要这个能力？

因为在 ADK 里，请求不一定总是“一次说完，一次结束”。

真实系统里可能出现：

- 用户确认后继续
- 鉴权后继续
- 长任务回调后继续
- task-mode agent 等待补充信息后继续

所以 Runner 必须能把这次消息正确接回之前的 invocation，而不是每次都重新开始。

这一步如果做不好，整个可恢复执行能力就会失效。

---

## 8. `_get_or_create_session`：为什么 Runner 天生站在会话语义上

关键源码：

Runner  
`src/google/adk/runners.py`

重点方法：

- `_get_or_create_session`

第一章讲 Agent 时我们已经强调过：ADK 不是临时脚本系统，而是会话型运行时。

Runner 在这里把这种设计落到了实处。

它不是先跑 agent 再想会话怎么处理，而是一开始就先解决：

- 这条请求属于哪个 session
- 找不到 session 时怎么办
- 是否允许自动创建 session

这说明在 ADK 的设计里：

**Session 不是执行后的附属物，而是执行前提。**

这对生产系统非常重要，因为：

- 历史上下文依赖 session
- 状态恢复依赖 session
- event persistence 依赖 session

如果把 session 处理晚了，很多运行时语义都无从谈起。

---

## 9. `_new_invocation_context`：Runner 把运行世界真正搭起来的时刻

关键源码：

Runner  
`src/google/adk/runners.py`

重点方法：

- `_new_invocation_context`
- `_new_invocation_context_for_live`

这一段是 Runner 和第一章 `InvocationContext` 之间真正接起来的地方。

第一章我们讲过：

- `InvocationContext` 是一次请求的总运行时

而在这一章，你终于能看到：

**是谁把它建出来的。**

Runner 在这里把这些东西统一装进 invocation context：

- artifact service
- session service
- memory service
- credential service
- plugin manager
- context cache config
- invocation id
- current session
- user content
- run config

这一步的意义非常大：

它说明 `InvocationContext` 不是某个 agent 自己拼的，也不是某个 tool 临时造的，而是 **Runner 统一创建并下发的运行时总对象**。

这恰恰说明 Runner 的“总调度器”角色。

---

## 10. `_handle_new_message` 与 `_append_user_event`：用户消息不是直接塞给 Agent

这也是很多初学者一开始会误判的地方。

他们容易以为：

`new_message -> 直接交给 agent`

但 Runner 的实际做法更严谨：

关键源码：

Runner  
`src/google/adk/runners.py`

相关位置：

- `_handle_new_message`
- `_append_user_event`
- `_append_new_message_to_session`

它做的事情大致是：

1. 先让 plugin 有机会处理用户消息
2. 再把消息追加到 session
3. 必要时附带状态变更
4. 再把这条用户消息纳入后续执行历史

这意味着：

**用户消息不是裸输入，而是系统历史的一部分。**

从这个 moment 开始，这条消息就不只是“当前参数”，而是变成了后续所有 agent/flow 可见的事实记录。

这也是为什么 ADK 很适合做：

- 多轮系统
- 审计系统
- 恢复系统

因为它不是临时传参，而是在一开始就把输入纳入事实流。

---

## 11. `_run_node_async`：Runner 真正把执行拉起来的地方

关键源码：

Runner  
`src/google/adk/runners.py`

重点方法：

- `_run_node_async`

这个方法值得特别重视，因为它几乎就是 Runner 的主战场。

### 11.1 它干了什么

你可以把它粗略理解成：

1. 拿到 session
2. 解析 resume 输入
3. 创建 invocation context
4. 处理用户消息
5. 决定 root node / root agent
6. 启动后台执行任务
7. 开始消费 event queue

它不是在“做业务逻辑”，而是在把运行时舞台搭完整。

### 11.2 为什么这里是 Node 而不只是 Agent

这里再一次体现 ADK 的一个重要设计取向：

**Runner 正在逐渐统一 agent runtime 和 workflow runtime。**

所以这里的执行单位常常是 node，而不只是老式的“直接跑一个 agent”。

这也解释了为什么你会在 Runner 里看到不少 workflow 相关逻辑。

---

## 12. `_consume_event_queue`：Runner 的核心不是 return，而是消费事件

如果这一章只记住一个实现点，我建议记这个。

关键源码：

Runner  
`src/google/adk/runners.py`

重点方法：

- `_consume_event_queue`

第一章我们已经提过：

Agent 和 Runner 的关系不是“函数返回值”关系，而是“生产事件 / 消费事件”关系。

这里就是这个思想真正落地的地方。

### 12.1 它在做什么

当 agent 或 node 执行时，事件会被放进 invocation context 的 `_event_queue`。  
Runner 的主循环则不断从这个队列里取事件出来，然后做这些事：

1. 合并 run-level metadata
2. 执行 `on_event` 插件回调
3. 决定输出 event 形态
4. 判断是否应该 append 到 session
5. 最终把 event yield 给调用方

这一步非常关键，因为它说明：

**真正决定 event 如何从“内部运行信号”变成“对外可见事实”的，是 Runner。**

### 12.2 为什么这件事不能放在 Agent 里做

因为 Agent 只是事件生产者之一。  
而以下这些语义需要一个更高层的统一视角：

- 当前 invocation 的插件策略
- session 持久化策略
- partial event 与 final event 的区别
- live 模式下的转写和 tool call 顺序

这些都不应该让某个单独 agent 决定。

所以 `_consume_event_queue` 才是 Runner 作为“运行时总调度器”的核心证据。

---

## 13. `_wrap_with_plugins`：为什么 Runner 是插件生命周期的总入口

关键源码：

Runner  
`src/google/adk/runners.py`

重点方法：

- `_wrap_with_plugins`

这个方法告诉你一件很重要的事：

插件不是在执行链之外“顺手挂一下”，而是由 Runner 统一包裹整个执行过程。

它主要负责三件事：

1. before-run callback
2. on-event callback
3. after-run callback

这意味着插件和执行主链并不是分离的，而是被 Runner 有意识地纳入生命周期。

所以从认知上说：

- Agent 定义业务角色
- Runner 定义请求级运行生命周期
- Plugin 则嵌在这个生命周期里

这三者的边界必须分清楚。

---

## 14. `run_live`：为什么 ADK 不只考虑文本聊天

关键源码：

Runner  
`src/google/adk/runners.py`

重点方法：

- `run_live`

这一章不需要深入讲 live API 细节，但读者至少要先知道：

ADK 的 Runner 不是只为传统文本问答设计的。

它还考虑了：

- 实时音频输入
- 实时音频输出
- transcription
- live session resumption

这说明 Runner 在架构上并不是“聊天机器人驱动器”，而是更通用的 Agent Runtime coordinator。

所以你后面看到它处理：

- live request queue
- buffered events
- transcription ordering

不要惊讶，这些都属于它的职责范围。

---

## 15. `close`：为什么资源清理也是 Runner 的职责

关键源码：

Runner  
`src/google/adk/runners.py`

重点方法：

- `close`

初学者常常忽略这一点：  
一个运行时系统不是“执行完就自然结束”。

真实系统里还要考虑：

- toolset 关闭
- plugin 关闭
- session service flush

所以 `close` 的存在说明：

**Runner 管的不只是开始执行，也管结束清理。**

这是生产级系统和 demo 代码之间非常典型的差异。

---

## 16. Agent 和 Runner 的关系，到这里应该怎么理解

现在可以把第一章和这一章合在一起，得到一个更完整的认知模型。

### 16.1 Agent 负责什么

Agent 负责：

- 定义业务角色
- 决定自己的能力边界
- 通过 flow 和 tool 执行具体智能行为

### 16.2 Runner 负责什么

Runner 负责：

- 建立 session 语义
- 建立 invocation 语义
- 选定当前该跑谁
- 组织 event 流
- 持久化运行事实
- 包裹插件生命周期
- 负责资源收尾

### 16.3 一句话总结它们的关系

**Agent 是执行主体，Runner 是请求级调度者。**

或者更口语一点：

- Agent 负责“干活”
- Runner 负责“开工、盯流程、记流水、收尾”

这就是二者最核心的边界。

---

## 17. 零基础读者应该怎么读 `runners.py`

`runners.py` 很长，最怕一上来从第一行读到最后一行。

更好的阅读顺序是：

### 第一步：只看入口方法

先读：

Runner  
`src/google/adk/runners.py`

- `__init__`
- `run`
- `run_async`
- `run_live`

目的不是读细节，而是先知道这个类对外暴露什么。

### 第二步：只追主链

再读：

- `_get_or_create_session`
- `_new_invocation_context`
- `_handle_new_message`
- `_run_node_async`
- `_consume_event_queue`

你只要先把这 5 段接成一条线，Runner 的主干就已经清楚了。

### 第三步：再补特殊语义

之后再看：

- `_resolve_invocation_id`
- `_find_agent_to_run`
- `_wrap_with_plugins`
- `close`

这些是 Runner 的高级控制逻辑。

### 第四步：最后再看 live 分支

最后再补：

- `run_live`
- live 相关转写与缓冲逻辑

否则一开始就看 live 分支，很容易把主线打散。

---

## 18. 这一章读完后，你应该能回答什么

如果这一章真的读进去了，你应该已经能回答下面这些问题：

1. 为什么 ADK 不能只有 Agent，没有 Runner？
2. 为什么 Runner 先处理 session，而不是先跑 Agent？
3. 为什么 `run_async` 比 `run` 更接近系统本质？
4. 为什么 Runner 要消费 event queue，而不是等 Agent return？
5. Agent 和 Runner 的边界到底在哪里？

如果这些问题你都能回答，说明你已经真正进入 ADK Runtime 的世界了，而不再只是停留在“会创建一个 agent”的层面。

---

## 19. 本章总结：先记住这五句话

### 第一句

**Runner 是一次请求级别的总调度器。**

### 第二句

**Runner 不负责思考内容，它负责把这次运行组织起来。**

### 第三句

**Agent 产出事件，Runner 消费事件并决定如何持久化与返回。**

### 第四句

**Session 和 InvocationContext 都由 Runner 统一建立，不由 Agent 自己拼装。**

### 第五句

**理解 Runner 的关键，不是背方法名，而是看懂它如何把“用户消息”变成“一串事件驱动的执行过程”。**

下一章我们会进入 `sessions`。  
如果说这一章讲的是“运行如何被拉起来”，那么下一章要讲的就是：

**这些运行中的状态、历史和事件，究竟被保存在哪里。**
