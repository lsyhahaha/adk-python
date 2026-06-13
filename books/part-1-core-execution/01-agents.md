# 第一章 Agents：先建立脑子里的运行模型

这是全书第一章，所以我们先不急着钻实现细节。

如果读者一上来就看 `BaseAgent`、`LlmAgent`、`InvocationContext` 这些类名，很容易产生一种错觉：ADK 只是又一套“把大模型包起来”的 Python 框架。这个理解是不够的。

`agents` 这一层真正解决的问题是：

**如何把一个“会思考、会决策、会调用能力、会和其他角色协作”的执行主体，放进一套可运行、可恢复、可观测的运行时里。**

这一章的目标不是背 API，而是帮你先建立四个最重要的认知：

1. Agent 到底是什么。
2. 为什么 ADK 必须先有 Agent，而不是先有 Tool。
3. Agent 在整个 ADK Runtime 里处在什么位置。
4. 如果你要从源码学 ADK，第一步该看哪里。

---

## 1. 从一个真实场景开始：为什么“聊天函数”不够

假设你要做一个企业内部智能助手。

业务方的要求通常不会是“接个模型，返回一段话”这么简单，而是类似下面这样：

- 用户可以问订单、发票、退款、知识库、流程制度等问题。
- 助手必要时要去查数据库、搜知识库、调用内部 HTTP API。
- 有些任务不能一次回答完，要分步骤完成。
- 有些任务需要多个角色协作，例如：
  - 一个角色负责检索资料
  - 一个角色负责总结
  - 一个角色负责审校输出
- 如果中途需要人工确认、授权登录、长任务等待，系统下次还能接着跑。

这时候，如果你的系统只有一个“万能聊天函数”，代码很快会变成下面这种形态：

- 一坨 prompt 拼接逻辑
- 一坨工具调用逻辑
- 一坨状态保存逻辑
- 一坨多角色分发逻辑
- 一坨异常恢复逻辑

这种系统一开始也许能跑，但几乎不可能维护成生产级。

所以 ADK 的第一步不是“先做模型封装”，而是先抽象出 **Agent**。

Agent 的意义在这里就出现了：

**把一个业务角色定义成独立的执行主体。**

这个主体不只是会说话，它还可以：

- 持有自己的目标和行为约束
- 访问工具
- 把任务交给别的 agent
- 在运行时读写状态
- 被 Runner 调度
- 被事件系统记录

换句话说，Agent 是 ADK 里“真正干活的人”。

---

## 2. Agent 到底是什么

先给一个面向零基础读者的定义：

**Agent 是一个被 ADK 运行时调度的业务角色。**

这个定义里有两个关键词。

### 2.1 它是“业务角色”

一个 Agent 不是一段随便调用的函数，也不是一个普通的数据类。

它代表的是某个明确的角色，例如：

- 售后助手
- 知识检索助手
- 研究助手
- 代码分析助手
- 审稿助手

这些角色的核心不在于“它用哪个模型”，而在于：

- 它负责解决什么问题
- 它能调用什么能力
- 它和其他角色如何分工

这也是为什么在源码里，`BaseAgent` 有非常基础但非常重要的字段：

BaseAgent  
`src/google/adk/agents/base_agent.py`

- `name`
- `description`
- `sub_agents`

这里的设计思想很直接：**先把角色定义清楚，再谈它怎么跑。**

### 2.2 它是“被运行时调度的主体”

Agent 并不是你自己手工调用几十个方法拼出来的。

在 ADK 里，Agent 最终会被运行时驱动，它的执行入口是统一的。

最值得先看的源码位置是：

BaseAgent  
`src/google/adk/agents/base_agent.py`

重点方法：

- `run_async`
- `run_live`
- `_run_async_impl`

即使你现在还没精读代码，只看名字也该建立一个基本判断：

- `run_async` 是统一入口
- `_run_async_impl` 是子类真正实现行为的地方

这说明 ADK 不是让每种 agent 随意定义生命周期，而是要求所有 agent 服从同一运行模板。

这是框架设计里非常重要的一步：**先统一执行骨架，再允许具体角色差异化。**

---

## 3. 为什么需要 Agent，而不是只要 Tool

这是初学者最容易混淆的地方。

很多人第一次看 Agent Framework，会下意识把 Agent 理解成“能调用 Tool 的模型”。这个理解只说对了一半。

### 3.1 Tool 是能力，Agent 是使用能力的角色

你可以先用一句最朴素的话区分它们：

- Tool 解决“能做什么”
- Agent 解决“谁决定什么时候做”

举个很实际的例子：

一个“查询订单”函数可以是 Tool。  
因为它本身不负责理解用户意图，也不负责决定何时调用自己，它只是提供一个能力。

但是“售后助手”不能是 Tool。  
因为它要做的是：

- 判断用户在问什么
- 决定是否需要查订单
- 决定查完之后怎么组织回答
- 必要时再转给别的角色

这就是 Agent 的职责。

### 3.2 Tool 不承担运行时责任

Tool 往往是局部能力单元，而 Agent 是运行时中的执行主体。

Agent 会被纳入这些机制中：

- 生命周期管理
- 状态恢复
- 事件记录
- 多 agent 协作
- 分支隔离
- 回调与插件拦截

Tool 通常不承担这些责任。

所以从架构上说：

**Tool 是原子能力，Agent 是运行时里的决策与组织单元。**

如果你把这两者混在一起，系统很快就会退化成“模型 + 一堆函数”的脚本工程，而不是可维护的 Agent Runtime。

---

## 4. Agent 在 ADK Runtime 里的位置

你现在可以先在脑子里放一张极简图：

`用户输入 -> Runner -> Agent -> Flow -> Model/Tools -> Event -> Session`

这张图现在不用完全懂，但必须先有。

因为它能帮你分清楚 Agent 不是孤立存在的。

### 4.1 Runner 负责“调度”

Agent 不自己启动自己。

真正把一次请求跑起来的是 `Runner`。

Runner  
`src/google/adk/runners.py`

可以先不读细节，但要先建立关系：

- Agent 负责定义角色和行为
- Runner 负责把这次请求调度起来

所以：

**Agent 不是总控层，Runner 才是。**

### 4.2 Flow 负责“模型调用链”

Agent 也不直接处理所有模型交互细节。

在 `LlmAgent` 里，真正的模型调用主循环是交给 flow 层的。

LlmAgent  
`src/google/adk/agents/llm_agent.py`

如果你去看 `_run_async_impl`，会发现它不是自己手写一整套模型循环，而是桥接到 `_llm_flow`。

这说明：

- Agent 决定“我是一个什么角色，有哪些约束和能力”
- Flow 决定“这次 LLM 请求链具体怎么跑”

这是一个非常关键的分层思想。

### 4.3 Session 和 Event 负责“事实记录”

Agent 运行过程中发生的事情，不会只留在内存里。

它会被组织成事件，并最终进入 session 历史。

这件事虽然不是 `agents` 模块自己独立完成的，但你必须先知道：

**Agent 是行为主体，但系统事实不是存进 Agent 对象里，而是通过 Event 和 Session 记录。**

这也是后面系统能做恢复、回放、调试、审计的基础。

---

## 5. `BaseAgent`：所有 Agent 的共同骨架

现在开始看源码，但我们只抓最关键的结构。

### 5.1 为什么 `BaseAgent` 必须存在

如果没有 `BaseAgent`，每个具体 agent 都会各自定义：

- 运行入口
- 回调方式
- 父子关系
- 运行状态传递方式

最后框架会彻底失去一致性。

所以 `BaseAgent` 的意义不是“抽象得优雅”，而是：

**强制所有 agent 使用同一套生命周期和树形结构。**

关键源码：

BaseAgent  
`src/google/adk/agents/base_agent.py`

建议先关注这些位置：

- 类定义本身
- `run_async`
- `_run_async_impl`
- `sub_agents`
- `before_agent_callback`
- `after_agent_callback`

### 5.2 `run_async` 是你最先应该看懂的方法

对零基础读者来说，`BaseAgent.run_async` 比字段定义更重要。

因为它告诉你 ADK 心中的 Agent 生命周期是什么。

阅读这个方法时，你应该抓这条主线：

1. 创建当前 agent 的 invocation context
2. 先执行 before callback
3. 再进入子类自己的执行实现
4. 最后执行 after callback

这就是典型的模板方法模式。

你不需要先学会这个术语，但要学会这种观察框架源码的方式：

**先找统一外壳，再找子类真正填空的地方。**

对 ADK 来说：

- 统一外壳在 `BaseAgent.run_async`
- 子类填空点在 `_run_async_impl`

这就是整个 Agent 体系的骨架。

### 5.3 `sub_agents` 不只是组合关系

读到 `sub_agents` 时，初学者很容易把它理解成普通对象包含。

这不够准确。

在 ADK 里，`sub_agents` 参与的是运行时结构，而不是简单的数据嵌套。

它至少影响这些事情：

- 角色树的组织方式
- 模型默认值继承
- agent transfer
- 恢复执行
- 并行或顺序协作

所以 `sub_agents` 的存在说明一件事：

**ADK 从一开始就不是按“单一聊天机器人”设计的，而是按“agent 树”设计的。**

---

## 6. `LlmAgent`：大多数读者真正会用到的主角

### 6.1 为什么 `LlmAgent` 才是日常开发重点

虽然 `BaseAgent` 是基础骨架，但大多数读者真正会实例化的是 `LlmAgent`。

LlmAgent  
`src/google/adk/agents/llm_agent.py`

可以把它理解成：

**一个由大模型驱动的业务角色定义。**

它在 `BaseAgent` 的基础上补齐了日常开发最关心的东西：

- 用哪个模型
- 角色指令是什么
- 可以用哪些工具
- 如何和其他 agent 协作
- 输出需要满足什么结构

### 6.2 初学者不要先被字段淹没

`LlmAgent` 的字段很多，第一次看容易头大。

这里建议你先不要试图全部记住，而是先分成四类：

1. 角色定义
   - `name`
   - `description`
   - `instruction`

2. 能力配置
   - `model`
   - `tools`

3. 协作方式
   - `sub_agents`
   - `mode`

4. 运行时高级控制
   - `output_schema`
   - `planner`
   - `code_executor`
   - 各种 callback

这样你就不会把 `LlmAgent` 看成一坨字段，而是能看出它在描述一个完整的业务角色。

### 6.3 它不是“模型调用函数”，而是“角色声明 + 运行桥接”

这是理解 `LlmAgent` 的关键。

如果你去看它的核心执行实现：

LlmAgent  
`src/google/adk/agents/llm_agent.py`

重点方法：

- `_run_async_impl`

你会发现它并不是手写了所有模型调用细节，而是把主循环交给 flow 层。

这说明 `LlmAgent` 的真正角色是：

- 对上，向业务方暴露“一个可定义的角色”
- 对下，把这个角色接到 ADK 的 LLM Runtime 上

所以不要把它理解成“模型 API 包装器”。

它更接近：

**一个可声明、可组合、可运行的智能角色对象。**

---

## 7. Agent 生命周期：一次运行到底发生了什么

现在我们把前面的内容串起来，给出一个适合零基础读者的 Agent 生命周期模型。

### 7.1 第一步：Runner 决定要跑哪个 Agent

用户请求先到 Runner。

Runner 负责：

- 找到 session
- 创建 invocation
- 决定当前该由哪个 agent 接手

此时 Agent 还没有真正开始执行业务逻辑。

### 7.2 第二步：`BaseAgent.run_async` 进入统一外壳

进入 Agent 层后，`BaseAgent.run_async` 开始接管生命周期。

这一步会：

- 为当前 agent 派生 invocation context
- 触发 before 回调
- 再进入子类实现

### 7.3 第三步：具体 Agent 执行自己的核心逻辑

如果是 `LlmAgent`，此时会进入它的 `_run_async_impl`。

再往下，flow 层开始组织模型调用、工具调用、转移、输出。

所以从执行链上看：

`Runner -> BaseAgent.run_async -> LlmAgent._run_async_impl -> Flow`

这条链建议你反复记住。

因为后面读 ADK 源码时，很多复杂感都来自“忘了自己现在处在哪一层”。

### 7.4 第四步：事件不断产出并被外部消费

Agent 在执行过程中不会只返回一个最终字符串。

它会不断产出 `Event`，这些事件随后会被 Runner 消费并落入 session。

所以 Agent 生命周期不是：

`输入 -> 一次函数调用 -> 输出`

而是：

`输入 -> 多步执行 -> 连续事件 -> 最终完成`

这也是 Agent Runtime 和普通函数式调用最根本的差异之一。

### 7.5 第五步：after callback 和结束

子类执行完后，`BaseAgent.run_async` 还会继续处理 after callback。

这个设计的意义是：你可以在不污染主体逻辑的前提下做统一收尾。

这也是生产系统里很常见的需求，例如：

- 统一补日志
- 统一修改输出
- 统一埋点

---

## 8. `Context` 和 `InvocationContext`：这是 ADK 最容易读糊涂、也最值得弄懂的地方

这一节要讲细一点。

因为很多读者第一次学 ADK，真正卡住的不是 `BaseAgent` 或 `LlmAgent`，而是：

- 为什么已经有了 Agent 对象，还要有 `InvocationContext`？
- 为什么已经有了 `InvocationContext`，还要再包一层 `Context`？
- 这两个对象到底谁负责什么？

如果这一层没理顺，后面读 `Runner`、`Workflow`、`ToolContext`、`Flow` 时会一直混。

所以这里我们不追求一句话带过，而是要把它讲成“脑子里能站住”的运行模型。

### 8.1 先记住一句话

你可以先把两者粗暴区分成这样：

- `InvocationContext`：一次请求级别的“总上下文”
- `Context`：当前执行单元级别的“工作上下文”

这个说法还不够精确，但足够帮你先分层。

### 8.2 为什么不能只靠普通函数参数

先回到一个真实问题。

如果一个 agent 只做最简单的单轮聊天，那么它看起来确实可以只依赖几个参数：

- 用户输入
- 模型
- 工具列表

但真实的 Agent Runtime 很快就会遇到下面这些事情：

1. 当前请求属于哪个 session？
2. 这次请求的 invocation id 是什么？
3. 如果前面某个 agent 已经跑过一半，怎么恢复它的状态？
4. 某个工具要求人工确认，确认回来之后怎么继续？
5. 某个 agent 产出的 event 要不要立刻写入 session？
6. 当前正在流式输出的是 partial event 还是最终 event？
7. 现在这一步是 root agent 在跑，还是 workflow 里的某个 node 在跑？

这些都不是“某个函数的业务参数”，而是**运行时事实**。

所以 ADK 才必须引入 context 对象，而不是只靠函数入参。

---

### 8.3 `InvocationContext`：一次请求的总控制台

关键源码：

InvocationContext  
`src/google/adk/agents/invocation_context.py`

先看类注释本身。  
这一段非常重要，因为它不是普通 docstring，而是在定义 ADK 对“运行层级”的理解：

- invocation
- agent call
- step

如果你以后只重读一个注释，优先重读这里。

#### 8.3.1 它解决的核心问题

`InvocationContext` 解决的是：

**一条用户请求进入系统后，在“整次执行期间”需要被所有参与者共享的运行时事实，放在哪里。**

注意这句话里的关键词是“整次执行期间”。

它不是某个 agent 私有的数据，也不是某个 tool 私有的数据，而是整个 invocation 共享的总上下文。

#### 8.3.2 你读源码时最该先看的字段

第一次读 `InvocationContext`，不要试图一个字段不漏地看完。  
先抓下面几个最有解释力的字段：

InvocationContext  
`src/google/adk/agents/invocation_context.py`

重点字段：

- `invocation_id`
- `session`
- `agent`
- `agent_states`
- `end_of_agents`
- `end_invocation`
- `plugin_manager`
- `_event_queue`

这几个字段几乎就能解释它为什么存在。

#### 8.3.3 `session`：说明它不是一次纯内存调用

`session` 的存在告诉你一件非常关键的事：

**ADK 从设计上就不是“临时回答一下就结束”的脚本系统。**

一次 invocation 永远挂在某个 session 上。  
也就是说，一个 agent 在运行时看到的不是孤立输入，而是一个有历史、有状态、有事件序列的会话环境。

所以当你看到：

InvocationContext  
`src/google/adk/agents/invocation_context.py`

字段：

- `session`

你脑子里要自动联想到：

- 会话历史
- 事件回放
- 多轮状态
- 恢复执行

#### 8.3.4 `agent_states` 与 `end_of_agents`：说明 ADK 天生考虑恢复

这两个字段很容易被初学者忽略，但它们其实特别能体现 ADK 的架构深度。

InvocationContext  
`src/google/adk/agents/invocation_context.py`

相关字段和方法：

- `agent_states`
- `end_of_agents`
- `set_agent_state`
- `reset_sub_agent_states`
- `populate_invocation_agent_states`

它们说明：

1. ADK 不只是让 agent“现在跑一下”。
2. ADK 还考虑 agent“跑到一半停住，下一次再接着跑”。

这就是为什么 `InvocationContext` 不是普通上下文对象，而是带有**恢复语义**的运行时对象。

对生产系统来说，这是非常重要的能力。  
因为一旦有：

- 长任务
- 人工确认
- 授权跳转
- 外部工具异步回调

你就不可能再把 agent 执行当成一次短平快函数调用。

#### 8.3.5 `end_invocation`：说明终止是运行时决策，不只是函数 return

另一个很值得注意的字段是：

InvocationContext  
`src/google/adk/agents/invocation_context.py`

- `end_invocation`

它的意义是：某个 callback、tool、agent 在运行中可以直接决定“整次 invocation 到这里结束”。

这和普通函数世界很不一样。

普通函数里，结束通常意味着当前函数 return。  
但在 ADK 里，结束是一个跨层级的运行时控制信号。

也就是说：

- 某个局部组件不只是在返回值
- 它还可能影响整次调用是否继续

这正是 Runtime 系统和普通库函数的本质区别。

#### 8.3.6 `plugin_manager`：说明横切逻辑也属于 invocation

很多人以为 plugin 只是 Runner 外面挂的一层辅助能力。  
但在 ADK 里，plugin manager 被放进 `InvocationContext`，这件事说明：

**插件不是调用外的附属品，而是当前 invocation 执行语义的一部分。**

比如：

- before callback
- after callback
- tool error interception
- model response rewrite

这些横切行为都要共享同一次 invocation 的上下文，因此它必须被纳入总控制台。

#### 8.3.7 `_event_queue`：说明 Runner 和 Agent 之间不是直接 return 关系

这一点很关键。

InvocationContext  
`src/google/adk/agents/invocation_context.py`

相关位置：

- `_event_queue`
- `_enqueue_event`

很多零基础读者一开始会默认：

`agent.run() -> return 最终结果`

但 ADK 不是这么设计的。

ADK 更接近：

`agent 运行中不断产出 event -> Runner 消费 event -> 再决定如何落盘和返回`

所以 `_event_queue` 的存在说明一件事：

**Agent 和 Runner 的关系不是“函数调用与返回值”，而是“生产事件与消费事件”。**

只要你真的理解了这一点，后面读 SSE streaming、pause/resume、workflow output 时会轻松很多。

---

### 8.4 `Context`：当前执行单元手上的工作台

关键源码：

Context  
`src/google/adk/agents/context.py`

如果说 `InvocationContext` 是总控制台，那么 `Context` 更像当前执行单元拿在手里的工作台。

它不是要取代 `InvocationContext`，而是在它的基础上提供一个更贴近“当前动作”的视角。

#### 8.4.1 它解决的核心问题

`Context` 解决的问题是：

**当某个 agent、node、tool 正在执行时，它应该以什么接口读取状态、记录动作、声明输出、处理恢复信息。**

你可以把它理解成：

- `InvocationContext` 面向整次调用
- `Context` 面向当前一步操作

#### 8.4.2 `state`：不是普通 dict，而是“带增量语义的状态视图”

Context  
`src/google/adk/agents/context.py`

重点属性：

- `state`

这一点必须讲清楚。

`ctx.state` 看起来像一个字典，但它不是单纯的 Python dict。  
它底层接的是一个带 delta 语义的状态对象。

这意味着：

- 你不只是“改了值”
- 你还在记录“这一步改了哪些状态”

为什么这很重要？

因为在 ADK 里，状态更新最终要进入事件系统和 session 持久化。  
如果没有 delta 语义，运行时就很难精确知道：

- 本步到底改了什么
- 这些改动是否应该持久化
- 回放时该如何恢复

所以 `ctx.state` 的设计不是语法糖，而是为了让状态修改能进入 Runtime。

#### 8.4.3 `actions`：当前这一步除了输出内容，还想声明什么

Context  
`src/google/adk/agents/context.py`

重点属性：

- `actions`

`actions` 的存在揭示了 ADK 的一个重要观念：

**一次执行步骤不只是产出文本，它还可能产出控制语义。**

例如当前这一步可能还想表达：

- 更新状态
- 设置路由
- 标记输出
- 触发 UI 渲染提示

这些都不是普通“返回值”能优雅承载的，所以需要 `actions`。

初学者可以先不深究每种 action，但要先知道：

`Context` 不只是“读上下文”，也是“写控制意图”的接口。

#### 8.4.4 `session`：当前执行单元看到的仍然是同一个会话世界

Context  
`src/google/adk/agents/context.py`

重点属性：

- `session`

这说明 `Context` 不是脱离 invocation 的独立小世界。  
它依然挂在同一个 session 上，只是以“当前执行视角”暴露出来。

这也是为什么你可以在局部步骤里：

- 读当前状态
- 看当前会话
- 决定输出

但它背后仍然属于整次 invocation。

#### 8.4.5 workflow 字段：说明它不仅服务于 agent，也服务于 node

如果你继续往下看 `Context`，会发现它还有很多 workflow 相关字段：

Context  
`src/google/adk/agents/context.py`

例如：

- `parent_ctx`
- `node`
- `node_path`
- `run_id`
- `resume_inputs`
- `attempt_count`
- `output`
- `route`

这些字段告诉你一件更深层的事：

**ADK 并没有把 agent 和 workflow 当成两套完全分裂的世界。**

相反，它试图让 `Context` 成为一个足够通用的局部执行上下文，既能服务于 agent，也能服务于 workflow node。

这也是后面你会看到 agent 可以被包装成 workflow node 的原因。

#### 8.4.6 `Context` 是“局部操作接口”，不是“全局事实仓库”

这里再强调一次边界：

- 全局事实仓库更接近 `InvocationContext`
- 当前执行操作接口更接近 `Context`

如果把两者混起来，你就会误以为：

- `Context` 是 session 总控台
- 或者 `InvocationContext` 是给业务代码直接操作的对象

这两种理解都会把源码读偏。

---

### 8.5 为什么 ADK 要拆成两层，而不是只保留一个 Context

这是最值得思考的设计问题。

因为很多框架最终只保留一个“万能 context”，然后把所有东西都塞进去。

ADK 没这么做，背后的原因很工程化。

#### 原因一：整次调用级别的事实，和当前执行单元级别的接口，不是一回事

一个对象如果同时承担：

- session 级事实
- agent 恢复状态
- 插件管理
- 事件队列
- 当前 node 输出
- 当前 step 路由

它很快就会变成不可理解的巨型杂物箱。

ADK 的拆分方式本质上是在做职责分离：

- `InvocationContext` 管总账
- `Context` 管当前工位

#### 原因二：不是每个局部执行单元都应该直接操作全局控制字段

例如：

- `_event_queue`
- `plugin_manager`
- `agent_states`
- `end_of_agents`

这些更接近运行时核心控制面，不应该在每个局部业务动作里被随意操作。

而 `Context` 则把更安全、更符合当前执行语义的接口暴露出来，例如：

- `state`
- `actions`
- `output`
- `route`

这其实也是一种封装。

#### 原因三：Workflow 和 Agent 共享 Runtime，但局部语义不同

ADK 不只是聊天 agent 框架，它还带 workflow。

这意味着运行时需要同时支持：

- invocation 级别的统一事实
- node/agent 级别的局部执行语义

拆两层，正好更容易承载这两种需求。

---

### 8.6 零基础读者该如何读这两个文件

建议这样读：

#### 第一步：先读 `InvocationContext` 类注释

InvocationContext  
`src/google/adk/agents/invocation_context.py`

只做一件事：

弄清楚 invocation、agent call、step 三层概念。

#### 第二步：只抓 8 个字段

先看：

- `invocation_id`
- `session`
- `agent`
- `agent_states`
- `end_of_agents`
- `end_invocation`
- `plugin_manager`
- `_event_queue`

不要一开始看全部字段。

#### 第三步：再看三个方法

再读：

- `_enqueue_event`
- `set_agent_state`
- `should_pause_invocation`

这三处最能体现它的 Runtime 味道。

#### 第四步：切到 `Context`

Context  
`src/google/adk/agents/context.py`

先只看：

- `state`
- `actions`
- `session`
- `node_path`
- `output`

这样你先建立“工作台”的感觉，再去看剩余 workflow 字段。

---

### 8.7 这一层你真正该建立的认知

如果这一节读完，你脑子里还剩一句最重要的话，那应该是：

**`InvocationContext` 负责保存整次请求共享的运行时事实，`Context` 负责给当前执行单元提供可操作的局部视角。**

一旦你把这句话吃透，后面再看：

- Agent 生命周期
- Runner 调度
- Workflow 节点执行
- ToolContext
- pause/resume

都会顺很多。

---

## 9. 如何学习 `src/google/adk/agents`

这一节最重要，因为本书不是源码展览，而是要带读者学会自己读。

### 9.1 正确顺序

建议按这个顺序读：

1. `src/google/adk/agents/__init__.py`
   先看公开面，知道这个模块对外想暴露什么。

2. `src/google/adk/agents/base_agent.py`
   先抓生命周期骨架，不要先看细节字段。

3. `src/google/adk/agents/llm_agent.py`
   再看最常用的具体 agent，理解业务角色如何接到 LLM Runtime。

4. `src/google/adk/agents/invocation_context.py`
   看一次调用在运行时到底携带哪些共享信息。

5. `src/google/adk/agents/context.py`
   看当前执行单元怎样读写状态、产出动作。

6. `src/google/adk/agents/run_config.py`
   理解“角色定义”和“本次运行配置”是两回事。

7. `src/google/adk/agents/sequential_agent.py`
8. `src/google/adk/agents/parallel_agent.py`
9. `src/google/adk/agents/loop_agent.py`
   这三个放最后看，把它们当作编排演进材料，而不是系统核心入口。

### 9.2 每次读源码时只追一个问题

初学者读框架源码最容易失败的方式，就是试图一次看懂所有东西。

更好的方法是每次只追一个问题：

- `BaseAgent`：Agent 生命周期怎么统一？
- `LlmAgent`：角色定义怎样接到模型调用链？
- `InvocationContext`：一次请求有哪些共享事实？
- `Context`：当前执行单元怎样读写状态？

你只要坚持这种阅读方法，ADK 会比看起来容易很多。

---

## 10. 如何开始使用 ADK 的 Agent

从实践角度讲，你的第一个 ADK agent 不应该太复杂。

第一步应该是：先定义一个清晰的业务角色，而不是急着堆很多工具和子 agent。

例如，一个最小的 `LlmAgent`，本质上要先回答四个问题：

1. 这个角色叫什么？
2. 它负责什么问题？
3. 它遵循什么行为规则？
4. 它需要哪些外部能力？

如果这四个问题都没想清楚，就不要急着加复杂运行时能力。

对生产系统来说，Agent 设计最忌讳的一件事就是：

**角色边界模糊。**

一个名字叫“assistant”的万能 agent，通常意味着设计还没有收敛。

更好的方式是：

- 一个 agent 解决一类问题
- 需要更多能力时，加 tool
- 需要更多角色时，加 agent tree 或 workflow

这也是 ADK 这套设计真正鼓励的建模方式。

---

## 11. 本章总结：先记住这四句话

如果你看完这一章，只记住四句话，也够了。

### 第一句

**Agent 是业务角色，不是模型调用函数。**

### 第二句

**Tool 是能力，Agent 是决定何时使用能力的主体。**

### 第三句

**Runner 调度 Agent，Agent 再桥接到 Flow 和 Tool。**

### 第四句

**学习 ADK 时，先看 `BaseAgent` 和 `LlmAgent`，先建立运行模型，再看细节字段。**

这四句话，就是你接下来阅读 ADK 全书的起点。

下一章我们进入 `Runner`。  
如果说这一章解决的是“谁来做事”，那么下一章要解决的就是：

**一次用户请求，究竟是如何把这些 Agent 真正跑起来的。**
