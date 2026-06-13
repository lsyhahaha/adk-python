# 第三章 sessions：系统为什么必须记住已经发生过的事

上一章我们已经解决了一个关键问题：

- Agent 是谁在干活
- Runner 是一次请求怎样被真正调度起来

但只理解到这里，仍然不够支撑一个真实可用的 ADK Runtime。

因为真实系统不会只处理“用户说一句，模型回一句，然后整件事彻底结束”的玩具流程。

真实业务里，你一定会遇到这些情况：

- 用户第一轮问：“帮我查一下 12345 订单的状态。”
- 第二轮紧接着追问：“那这单能开票吗？”
- 中间 agent 可能调用 tool，甚至调用到一半等待外部结果回来。
- 同一个用户可能同时开多个会话。
- 运营、研发、审计还可能要回放某一次执行过程，看看系统到底做过什么决定。

这时候，如果系统只是把上下文放在某个 Python 变量里，或者每次都临时把“最近几轮对话”拼进 prompt，它很快就会失去三个生产系统最需要的能力：

1. **连续性**：下一次请求还能接着上一次继续。
2. **可恢复性**：执行中断后还能从系统事实继续，而不是从头猜。
3. **可审计性**：你能准确回答“系统当时到底看到了什么、写了什么、调用了什么”。

所以，`Runner` 解决的是：

**这次请求怎么跑起来。**

而 `sessions` 解决的是：

**跑过之后，系统如何把已经发生过的事记住、恢复、隔离和持久化。**

这一章不要把它理解成“会话存储模块介绍”。

更准确地说，这一章讲的是：

**ADK 如何把运行过程沉淀成一套可持续引用的系统事实。**

---

## 1. 为什么聊天历史不能只放在 prompt 或 Python 变量里

先从一个非常常见、也非常容易被低估的误区开始。

很多人第一次做 Agent 系统时，会自然想到两种做法：

- 做法一：把历史对话保存在内存变量里，下次请求继续复用。
- 做法二：每次把最近几轮消息重新拼进 prompt，交给模型自己理解。

这两种做法在 demo 阶段都能跑，但到了真实业务里，会很快暴露问题。

### 1.1 只靠 Python 变量，进程一断就丢

假设系统正在处理一个复杂任务：

- 用户先提问
- agent 调用 tool 查订单
- tool 又触发外部 API
- 中间系统重启，或者 worker 切换

如果历史只在内存里，你就会发现：

- `Runner` 知道这次请求正在跑
- 但系统不知道之前已经发生了什么
- 更不知道哪些 `state` 已经被写入，哪些还只是临时值

结果就是：

- 不是重复执行
- 就是接不上上下文
- 更严重时，会把同一个动作做两遍

对于生产系统，这种行为是不能接受的。

### 1.2 只靠 prompt 历史，拿不到“系统事实”

另一种常见误区是：  
“反正模型只需要上下文，那我把历史对话拼进去不就行了？”

这也不够。

因为系统真正需要保留的，不只是“自然语言聊天记录”，还包括：

- 用户输入
- 模型输出
- function call
- function response
- `state_delta`
- branch / `isolation_scope`
- invocation 级别的执行信息

这些东西里，有一大半根本不是“聊天记录”能完整表达的。

比如：

- 一个 tool 到底有没有成功执行
- 某个 `user:` 状态有没有被更新
- 当前这条事件属于哪个 task delegation
- live 模式下哪些 partial event 不该写入持久层

这些都属于**运行时事实**，而不是 prompt 拼接能替代的东西。

### 1.3 所以 `sessions` 真正解决的问题是什么

这一部分属于思维导图里的“核心执行主干 → sessions → 为什么存在”分支。

先给一句话定义：

**`sessions` 不是为了“保存聊天记录”，而是为了给运行时提供连续、可恢复、可审计的事实容器。**

你可以先建立一个最重要的判断：

- prompt history 是“给模型看的上下文材料”
- `Session` 是“给系统自己保存事实的运行容器”

这两者有关联，但绝对不是一回事。

后面第九章讲 `memory` 时，这个区别会更重要。因为：

- `Session` 保存的是发生过什么
- `Memory` 沉淀的是以后给模型参考什么

先把这个边界钉死，后面就不容易混。

---

## 2. `Session` 到底是什么：一次连续交互的事实容器

这部分属于“核心执行主干 → sessions → Session 数据模型”分支。

先给一句话定义：

**`Session` 是一次连续交互的事实容器，不是 LLM prompt 的别名。**

最直接的源码入口是：

[session.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/session.py:28)

先看 `Session` 里最核心的字段：

- [`id`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/session.py:39)
- [`app_name`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/session.py:41)
- [`user_id`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/session.py:43)
- [`state`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/session.py:45)
- [`events`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/session.py:47)
- [`last_update_time`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/session.py:50)

这些字段非常少，说明一件事：

**`Session` 本身并不复杂，复杂的是围绕它的写入语义。**

### 2.1 `id / app_name / user_id`：先回答“这是谁的哪次连续交互”

`Session` 不是全局唯一的一团历史，它一定挂在三层标识下：

- `app_name`：属于哪个 ADK app
- `user_id`：属于哪个用户
- `id`：这个用户在这个 app 下的哪一个 session

这意味着系统天然支持这样的事实隔离：

- 同一个 app 的不同用户，session 隔离
- 同一个用户的不同 session，彼此隔离
- 同一个用户在同一个 app 里，也可以并行存在多条连续会话

这不是锦上添花，而是生产系统的基本要求。

### 2.2 `events`：真正被记录下来的不是字符串，而是 `Event`

`Session.events` 的类型不是 `list[str]`，也不是“message history”。

它是：

- `list[Event]`

这点非常关键。

因为它说明：

**`Session` 里真正追加的是 `Event`，不是聊天气泡。**

这也是下一章 `events` 会展开讲的内容。你现在先记住：

- `Session` 是容器
- `Event` 是容器里被追加的最小事实单元

也就是说，`sessions` 这一层负责“怎么组织和保存事实”，而不是定义“事实长什么样”。  
“事实协议”本身是 `events` 这一章的职责。

### 2.3 `state`：这不是普通字典，而是运行状态的当前视图

第一次看 `Session.state`，很多人会下意识觉得它就是“存点变量”。

这个理解太浅了。

更准确地说：

**`Session.state` 是当前 session 可见运行状态的聚合视图。**

为什么说是“聚合视图”而不是单纯的持久字段？

因为后面你会看到：

- 有些状态是 session-local
- 有些状态是 `user:` 级别
- 有些状态是 `app:` 级别
- 还有些 `temp:` 状态只在当前 invocation 内有效

所以当你读取 `session.state` 时，你拿到的不是“数据库里某一列的原样内容”，而是系统整理后的当前视图。

### 2.4 `last_update_time`：这不是装饰字段，而是并发一致性的一部分

很多教材在讲会话对象时，会把更新时间当成边角信息。

但在 ADK 里，`last_update_time` 不是摆设。

它至少承担两个角色：

1. 表示这个 session 最近一次被写入的时间
2. 配合具体存储实现，帮助检测 stale session 或并发写入冲突

你在 [DatabaseSessionService.append_event()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/database_session_service.py:647) 里会看到更严格的 revision 检查逻辑。  
这说明 session 不是随便追加一点历史就完了，它要面对真实的并发和恢复问题。

所以，到这里可以先给一个中间结论：

**`Session` 是运行时中的连续事实容器；它看起来很薄，但被设计成了一个可以承接真实执行链的稳定边界。**

---

## 3. `BaseSessionService` 解决什么问题：把 session 生命周期和存储实现解耦

这部分属于“核心执行主干 → sessions → SessionService contract”分支。

先给一句话定义：

**`BaseSessionService` 是 session 生命周期和后端存储之间的 contract。**

关键源码：

[base_session_service.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:54)

如果没有这一层，会发生什么？

- `Runner` 必须自己知道怎么建 session
- `Runner` 必须自己知道怎么查历史
- `Runner` 还得知道 SQLite、数据库、内存、托管后端各自怎么写

这样很快就会把调度层和存储层耦死。

而 ADK 的做法是：

**让 `Runner` 只依赖 session service contract，不依赖具体存储细节。**

### 3.1 先看最基本的四个生命周期方法

`BaseSessionService` 最基础的方法是：

- [`create_session()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:61)
- [`get_session()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:83)
- [`list_sessions()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:94)
- [`delete_session()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:109)

这四个方法对应的不是数据库 CRUD 教程，而是运行时最基本的生命周期动作：

- 创建一次连续交互
- 读取一次连续交互
- 枚举某个用户或 app 下的连续交互
- 删除不再需要的连续交互

这里要注意一个重要细节：

`get_session()` 不是简单“按主键查对象”。

它还支持 `GetSessionConfig`，例如：

- `num_recent_events`
- `after_timestamp`

这说明 ADK 在设计上已经考虑到：

**读取 session 不一定总是“全量加载全部历史”，而可能是按恢复、回放、裁剪等需求取局部视图。**

### 3.2 `get_user_state()` 为什么单独存在

这是 `BaseSessionService` 里非常容易被忽略、但很有工程意义的一个点。

方法：

- [`get_user_state()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:114)

它解决的问题是：

**系统在创建新 session 之前，也可能需要先知道这个 user 级别已经保存过什么状态。**

例如：

- 用户级偏好设置
- 某个跨 session 的身份绑定
- 某个需要长期沿用的业务变量

如果没有 `get_user_state()`，系统就只能：

- 先 `list_sessions()`
- 再逐个 `get_session()`
- 再自己拼出 user-scoped state

这不仅低效，而且把存储语义泄漏给了调用方。

所以 `get_user_state()` 的存在，不是“多加一个方便方法”，而是明确告诉你：

**ADK 认为 `user:` 级状态是正式的一等概念。**

### 3.3 为什么说它是 contract，而不是某个默认实现

`BaseSessionService` 自己不关心你把数据放在哪。

它只规定：

- 你必须能 create / get / list / delete
- 你必须能 `append_event()`
- 你可以选择是否支持 `get_user_state()`

然后具体实现再各自完成：

- `InMemorySessionService`
- `SqliteSessionService`
- 更重的数据库后端

这就是典型的工程化分层：

- `Runner` 只面向 contract
- 存储后端在 contract 背后替换

你读源码时要特别注意，不要把这一层理解成“只是为了多态写法好看”。

它的真正价值是：

**让运行时调度逻辑不需要知道存储细节，同时让不同部署形态能复用同一套 session 生命周期语义。**

---

## 4. `append_event()` 为什么是本章核心：事件落库前如何更新 `state`

这部分属于“核心执行主干 → sessions → append_event 提交语义”分支。

先给一句话定义：

**`append_event()` 不是简单的 `session.events.append(...)`，而是把运行中的变化提交成系统事实的核心入口。**

关键源码：

[base_session_service.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:154)

如果你只看函数名，会以为它只是“追加一条 event”。

但真正的逻辑顺序是：

1. 过滤 `partial` event
2. 先 `_apply_temp_state()`
3. 再 `_trim_temp_delta_state()`
4. 再 `_update_session_state()`
5. 最后 `session.events.append(event)`

这个顺序不是随手写的，而是设计出来的。

### 4.1 为什么 `partial` event 不直接进 session

第一步就是：

- 如果 `event.partial` 为真，直接返回

这背后的意思很明确：

**ADK 不把“尚未完成的中间片段”默认当成正式系统事实。**

原因很好理解：

- partial transcript 可能只是中间识别结果
- streaming 片段可能只是增量输出
- 如果全写进去，历史会充满半成品和噪声

这也是后面 `Runner` 里 live 模式要额外做过滤的基础。

### 4.2 为什么必须先 `_apply_temp_state()`

这一段是本章最值得认真讲透的地方。

`append_event()` 先做的是：

- [_apply_temp_state(session, event)](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:174)

它会把 `event.actions.state_delta` 里以 `temp:` 开头的键，先写进当前内存里的 `session.state`。

注意，是**先写进内存**。

为什么必须这样？

因为 `temp:` 的设计目标是：

**在当前 invocation 内可见，但不进入持久化事实。**

这意味着同一次执行链里，后续 agent 可能还要读这个值；但系统又不希望它污染长期状态。

所以如果你先把它裁掉，再更新内存，后续节点就读不到它。

这会直接破坏同一次 invocation 内的运行语义。

### 4.3 为什么紧接着 `_trim_temp_delta_state()`

第二步做的是：

- [_trim_temp_delta_state(event)](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:187)

它会把 `temp:` 前缀的 `state_delta` 从 event 里裁掉。

这一步的目的正好和前一步互补：

- 前一步保证“当前 invocation 还能继续读”
- 这一步保证“持久化历史不要被临时变量污染”

所以整个组合语义是：

**`temp:` 状态要对当前执行可见，但不能冒充长期系统事实。**

这就是为什么顺序不能反。

### 4.4 `_update_session_state()` 才是正式状态提交

在裁掉 `temp:` 之后，剩下的 `state_delta` 再进入：

- [_update_session_state(session, event)](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:204)

这里更新的是“应该成为正式 session 状态一部分”的值。

到这一步时，进入 `session.state` 的就不再是临时辅助变量，而是当前 session 应该保留的正式状态。

最后才会：

- `session.events.append(event)`

也就是说，追加 event 不是一个孤立动作，而是和状态提交绑定在一起的。

### 4.5 这一整套顺序解决了什么工程问题

你可以把 `append_event()` 的设计目的总结成一句话：

**ADK 需要保证“当前执行还看得到的状态”和“长期保存下来的状态”既有关联，又不混淆。**

如果没有这层语义，系统很快会出现两种错误：

1. 临时变量丢失，导致同一次 invocation 后续节点读不到
2. 临时变量被持久化，导致后续 session 恢复时读到不该长期存在的垃圾状态

这就是为什么我前面说：  
`sessions` 真正复杂的地方，不在 `Session` 这个模型，而在 `append_event()` 这种提交语义。

---

## 5. `state` 的三层作用域：`app:`、`user:`、session-local，以及 `temp:` 的临时语义

这部分属于“核心执行主干 → sessions → state 作用域”分支。

先给一句话定义：

**ADK 里的 state 不是平铺字典，而是带作用域的运行状态。**

关键源码：

- [state.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/state.py:61)
- [in_memory_session_service.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/in_memory_session_service.py:61)
- [sqlite_session_service.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/sqlite_session_service.py:132)

先看 `State` 里定义的几个前缀：

- [`APP_PREFIX = "app:"`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/state.py:64)
- [`USER_PREFIX = "user:"`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/state.py:65)
- [`TEMP_PREFIX = "temp:"`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/state.py:66)

这几个前缀直接告诉你：  
ADK 不把所有状态都塞进同一个语义平面。

### 5.1 session-local：默认就是“这次会话自己的状态”

没有前缀的 key，可以先理解成：

**session-local 状态。**

它只属于这个 `session_id` 对应的连续交互。

例如：

- 某次对话里已经解析出的订单号
- 当前会话里积累的中间结果
- 只在这次连续交互里需要保留的业务变量

这种状态不会天然共享给别的 session。

### 5.2 `user:`：同一个用户跨 session 共享

`user:` 前缀代表：

**同一个 app 下，同一个 user 的跨 session 状态。**

这就意味着：

- 用户在 session A 里写入的 `user:xxx`
- 在 session B 里也可能被看到

为什么这样设计？

因为真实业务里，很多信息并不属于某一次聊天，而属于这个用户本身。

比如：

- 用户长期偏好
- 绑定过的账户信息
- 跨 session 复用的身份态

[`BaseSessionService.get_user_state()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:114) 的存在，就是在告诉你：

**ADK 认为这类状态是正式一级概念。**

### 5.3 `app:`：同一个 app 跨用户共享

`app:` 前缀更高一级。

它表示：

**同一个 app 下共享的状态。**

这类状态通常适合放：

- 全局业务配置
- 某种应用级缓存结果
- 所有用户都应看到的一致运行参数

注意，这不是“推荐你随便共享一切”，而是 ADK 给你提供了清晰的语义边界。

工程上最重要的不是“共享”，而是：

**共享必须是显式的，而不是误打误撞发生的。**

### 5.4 `temp:`：只活在当前 invocation

`temp:` 是最容易误读的前缀。

它不是“低优先级状态”，而是：

**只在当前 invocation 内有效的临时状态。**

它的关键特征是：

- 当前执行链后续节点可以读
- 不应该被持久化到长期事实

这就是上一节里 `_apply_temp_state()` 和 `_trim_temp_delta_state()` 要配合出现的原因。

也正因此，这里必须明确一句：

**`temp:` 不是弱化版 session 状态，而是“只对当前执行暴露、对长期历史隐藏”的运行时辅助状态。**

### 5.5 `InMemorySessionService` 和 `SqliteSessionService` 怎么体现这种分层

先看 [`InMemorySessionService.create_session()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/in_memory_session_service.py:78)：

- 它会把输入状态先拆成 `app` / `user` / `session`
- `app` 部分写进 `self.app_state`
- `user` 部分写进 `self.user_state`
- session-local 部分进入 `Session.state`

然后 `get_session()` 返回时，再通过 `_merge_state()` 把这些状态合并成调用方看到的 `session.state` 视图。

这说明：

**你读到的 `session.state`，并不等于“session 表里原始存了什么”，而是合并后的当前可见视图。**

[`SqliteSessionService.create_session()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/sqlite_session_service.py:157) 也是同样的思想，只是换成了持久化表：

- `app_states`
- `user_states`
- `sessions`
- `events`

这套表结构本身就体现了状态分层，不是临时拼出来的。

所以，读者到这里要建立一个很重要的习惯：

**看到 `session.state` 时，不要先问“这是个 dict”，要先问“这是哪个作用域合并后的视图”。**

---

## 6. `Runner` 在什么时机写 session：不是跑完再存，而是沿执行链持续 append

这部分属于“核心执行主干 → sessions → Runner 写入链路”分支。

先给一句话定义：

**`Runner` 不是最后统一保存结果，而是在执行链的关键节点持续把事实写回 session。**

关键源码：

[runners.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/runners.py:731)

这是理解 ADK 的一个重要认知切换：

- 不是“先跑完逻辑，再把结果存下来”
- 而是“边跑边生成 event，边把正式事实写进 session”

### 6.1 `_append_user_event()`：用户输入先写入 session

先看用户消息进入系统时的动作：

- [_append_user_event()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/runners.py:731)

它会创建一个用户事件，然后调用：

- [`self.session_service.append_event(session=ic.session, event=event)`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/runners.py:759)

这说明什么？

说明用户输入本身就是正式系统事实，不是某个临时参数。

更进一步，它还会处理：

- `isolation_scope`

也就是说，连“这条用户消息属于哪个逻辑隔离范围”都会在写入时被明确标记。

这已经不是普通聊天记录系统的语义了，而是 runtime 级事实记录。

### 6.2 `_consume_event_queue()`：执行中产出的 event 会持续写回

再看 [_consume_event_queue()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/runners.py:778)。

这个方法不是简单把 event yield 给调用方，而是会在非 `partial` 情况下先：

- 走 plugin 回调
- 合并修改后的 event
- 再 `append_event()` 进 session

这背后的设计思想非常明确：

**流给外部看的 event，和存进 session 的 event，必须尽量保持一致。**

否则就会出现一种非常糟糕的情况：

- 调用方看到的是 A
- session 里记下的是 B

这样系统就失去了可审计性。

### 6.3 `_exec_with_plugin()`：不是只有主执行链才会写

`_exec_with_plugin()` 更能说明这个设计。

它处理的包括：

- `before_run` 提前返回
- 正常事件流
- live / bidi 模式下的特殊缓冲
- plugin 对 event 的修改

但无论路径怎么分叉，只要某个 event 被认定应该成为正式事实，它最终都会进入：

- `self.session_service.append_event(...)`

这说明 session 写入不是某个角落里的副作用，而是 Runner 主执行链的正式组成部分。

### 6.4 live 模式为什么不能无脑全写

[`Runner._should_append_event()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/runners.py:1309) 明确告诉你：

有些 live 模式下的事件不该直接进入 session，尤其是：

- 带大块 inline media 的模型响应

原因不是“懒得存”，而是这些内容：

- 体积大
- 噪声高
- 对事实恢复不一定必要

更合理的做法是：

- 大 blob 放到 artifact
- session 里保留必要引用或正式事件

这再次说明：

**session 不是“所有运行数据的垃圾桶”，而是系统正式事实的受控沉淀层。**

### 6.5 这一节最该记住的结论

如果你只记一句话，就记这句：

**在 ADK 里，session 不是请求结束后的归档，而是请求执行过程中的持续提交面。**

这也是为什么 `sessions` 会放在 `runners` 后面、`events` 前面：

- `Runner` 负责把请求跑起来
- `sessions` 负责把跑出来的事实持续接住
- `events` 则定义这些事实单元本身

这个顺序是有逻辑的，不是随便排的。

---

## 7. `Session` vs `Event` vs `Memory`：分别负责什么，不负责什么

这部分属于“核心执行主干 → sessions → 边界澄清”分支。

先给一句话定义：

**`Session` 负责承载连续事实，`Event` 负责表达单个事实，`Memory` 负责给模型提供提炼后的长期参考。**

这三个概念不拆清，后面读源码很容易越读越混。

### 7.1 `Session`：负责“把事实组织成连续交互”

`Session` 负责的是：

- 标识一次连续交互
- 聚合当前可见 `state`
- 保存一串 `Event`
- 提供恢复、回放、审计的容器边界

它**不负责**：

- 定义事件协议细节
- 决定模型实际看到哪些上下文
- 直接承担长期知识检索

### 7.2 `Event`：负责“表达已经发生的最小事实单元”

`Event` 负责的是：

- 表达用户输入
- 表达模型输出
- 表达 function call / function response
- 表达 `state_delta`
- 表达 branch / `isolation_scope` / node 信息

它是系统的事实原子。

这也就是为什么可以说：

**`Event` 才是 ground truth 的最小单元，`Session` 是这些 ground truth 的组织容器。**

下一章会专门讲：  
为什么 ADK 选择用统一 `Event` 协议来承接这么多不同执行形态。

### 7.3 `Memory`：负责“以后给模型参考什么”

`Memory` 的角色完全不同。

它更接近：

- 长期记忆
- 检索结果
- 摘要沉淀
- 面向模型的知识提炼视图

所以它关心的问题不是：

- “真实发生过什么”

而是：

- “以后什么内容值得再给模型看”

这就是为什么不能拿 `Memory` 替代 `Session`：

- `Memory` 可能会摘要、筛选、压缩
- `Session` 需要保留审计和恢复所需的正式事实链

换句话说：

- `Session` 是 system of record
- `Memory` 是 model-facing knowledge layer

### 7.4 一个简短对比表

| 概念 | 主要职责 | 不负责什么 |
|---|---|---|
| `Session` | 组织一次连续交互的事实容器 | 不定义事件协议，不直接替代长期记忆 |
| `Event` | 表达单个已发生事实 | 不负责聚合整段会话 |
| `Memory` | 提炼可复用的长期参考 | 不适合做审计、回放、精确恢复 |

### 7.5 本章和后续章节的桥

到这里，三条后续阅读路径就自然接上了：

1. 通往下一章 `events`
   - 因为 `Session` 里真正保存的是 `Event`
   - 所以接下来要看“事实单元本身长什么样”

2. 通往后面的 `workflow`
   - 因为同一个 session 里并不只有线性聊天历史
   - 还有 branch、`isolation_scope`、invocation、node path 这些复杂执行信息

3. 通往第九章 `memory`
   - 因为 `Session` 记录的是系统事实
   - `Memory` 沉淀的是模型以后可能要参考的知识

如果你把这三条桥记住，后面的章节就不会各看各的。

---

## 8. 这一章读完后，应该怎样继续读源码

这部分属于“核心执行主干 → sessions → 源码阅读顺序”分支。

如果你已经理解了这一章的主线，接下来最推荐的源码顺序是：

1. [session.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/session.py:28)
   - 先看 `Session` 这个容器到底有哪几个核心字段

2. [base_session_service.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:54)
   - 重点看 [create_session()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:61)
   - [get_session()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:83)
   - [get_user_state()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:114)
   - [append_event()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:154)
   - [_apply_temp_state()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:174)
   - [_trim_temp_delta_state()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/base_session_service.py:187)

3. [runners.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/runners.py:731)
   - 重点找 [_append_user_event()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/runners.py:731)
   - [_consume_event_queue()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/runners.py:778)
   - [_should_append_event()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/runners.py:1309)
   - 看 `Runner` 是在什么时机把事实写进 session

4. [in_memory_session_service.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/in_memory_session_service.py:61)
   - 看最容易理解的一版实现
   - 特别看 state 拆分和 `_merge_state()`

5. [sqlite_session_service.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/sqlite_session_service.py:132)
   - 看持久化实现怎么把 `app_states`、`user_states`、`sessions`、`events` 分开存
   - 理解 ADK 是怎样把运行时语义落进实际存储结构

如果你按这个顺序读，会比一上来直接扎进数据库实现轻松很多。

---

## 9. 本章小结：`sessions` 不是聊天记录，而是运行时事实系统

这一部分属于“核心执行主干 → sessions → 小结”分支。

你现在应该能把这一章压缩成四句话：

1. `Runner` 解决“请求怎么跑起来”，`sessions` 解决“跑过之后系统怎么记住发生过什么”。
2. `Session` 是连续交互的事实容器，真正被保存进去的是 `Event`。
3. `append_event()` 是本章核心，因为它同时处理 event 追加和 `state` 提交语义，尤其是 `temp:` 的临时状态边界。
4. `Session`、`Event`、`Memory` 不是同义词，它们分别服务于恢复/审计、事实表达、模型参考三个不同目标。

如果把这一章学明白，你就不再会把 ADK 看成“模型 + 历史消息列表”的简单封装。

你会开始意识到：

**ADK 在 `sessions` 这一层，已经把 Agent Runtime 最关键的一部分工程化能力搭起来了。**

下一章，我们就继续顺着这个问题往下看：

**既然 `Session` 保存的是一串 `Event`，那 `Event` 本身到底是怎样定义成统一事实协议的？**

---

📊 本次学习小结
- 模块：`sessions`（[src/google/adk/sessions/](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/sessions/session.py:28)）
- 当前深度：L2（理解 How），局部进入 L3（理解为什么 `temp:` 不持久化）
- 下一阶段建议：进入下一章 `events`，把 `Session` 里的“事实单元”彻底拆开
- 掌握程度自评：🟡 基本掌握（如果还没读 `runners.py` 里的写回链路，建议配合本章再过一遍）
