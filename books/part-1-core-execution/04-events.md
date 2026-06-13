# 第四章 events：系统怎样把每一次动作都变成统一事实协议

上一章我们刚刚把一个关键边界钉死：

- `Session` 不是 prompt history
- `Session` 是运行时连续事实的容器

但上一章其实还留了一个更根本的问题没有展开：

**既然 `Session` 里保存的是一串 `Event`，那这个 `Event` 到底是什么？**

如果你第一次看 ADK，很容易把 `Event` 想成“聊天消息对象”。

这个理解不够。

因为在真实运行里，被系统记录下来的不只是：

- 用户说了什么
- 模型回了什么

还包括：

- function call
- function response
- `state_delta`
- artifact 版本变化
- workflow route
- `branch`
- `isolation_scope`
- UI widget 渲染指令
- agent 结束、rewind、compaction 这些控制语义

如果这些东西分别用不同结构保存，系统很快就会遇到一个经典问题：

- `Runner` 看到一种对象
- `Session` 保存另一种对象
- `workflow` 再发明第三种对象
- 前端事件流还要再转第四种

这种系统会非常难维护。

所以 ADK 在 `events` 这一层做的事，本质上是：

**用一个统一的事实协议，把“对话内容”“工具动作”“状态变化”“控制信号”收敛到同一种运行时记录里。**

这一章要解决的，就是这个统一协议的设计。

---

## 1. 为什么 ADK 需要统一 `Event` 协议，而不是一堆分散对象

这部分属于“核心执行主干 → events → 为什么存在”分支。

先给一句话定义：

**`Event` 是 ADK 里记录运行事实的统一协议，不只是聊天消息。**

先从一个真实场景开始。

假设用户说：

“帮我分析这个需求，先列提纲，如果信息不够就继续追问我。”

这条请求在运行过程中，可能依次产生这些东西：

1. 一条 user 输入
2. 一条 model 输出
3. 一次 function call
4. 一次 function response
5. 一次 `state_delta`
6. 一个 `route`，把执行导向某个 workflow 分支
7. 一条表示 agent 当前已经结束的控制信号

如果每一类都用独立对象：

- `UserMessage`
- `ModelMessage`
- `ToolCall`
- `ToolResult`
- `StateUpdate`
- `WorkflowSignal`

系统就会面临一个工程问题：

**谁来定义它们之间的统一时序？**

而 ADK 的选择是更激进但也更稳定的一种：

**不管发生的是内容、动作还是控制，都先尽量收敛成 `Event`。**

这样做的价值非常直接：

1. `Session` 只需要保存一种核心事实单元
2. `Runner` 只需要围绕一种协议做流式输出和持久化
3. `workflow`、`tools`、`memory`、前端消费层，都可以围绕同一条事实流理解系统行为

所以这一章你首先要建立一个判断：

**ADK 不是“有事件功能”，而是“运行时本身就是基于事件事实流组织的”。**

---

## 2. `Event` 到底是什么：它不是 message，而是事实单元

这部分属于“核心执行主干 → events → Event 数据模型”分支。

先给一句话定义：

**`Event` 是 ADK 里“已经发生过的一个最小运行事实单元”。**

最重要的源码入口是：

- [event.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:93)

先看 `Event` 直接定义的核心字段：

- [`invocation_id`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:109)
- [`author`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:111)
- [`actions`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:114)
- [`output`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:117)
- [`node_info`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:120)
- [`long_running_tool_ids`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:123)
- [`branch`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:128)
- [`isolation_scope`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:137)
- [`id`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:154)
- [`timestamp`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:156)

只看这些字段，你就应该先有一个直觉：

**这不是一个“聊天对象”，而是一个“运行对象”。**

因为它同时回答了四类问题：

1. 这条事实属于哪次执行  
   看 `invocation_id`

2. 这条事实是谁产生的  
   看 `author`

3. 这条事实除了内容外，还附带了哪些动作  
   看 `actions`

4. 这条事实在 workflow / branch / task isolation 里处于什么位置  
   看 `node_info`、`branch`、`isolation_scope`

### 2.1 `invocation_id`：先区分“同一个 session 里的哪一次执行”

`Session` 负责表示连续交互。

但在同一个 `Session` 里，用户可以多次发消息，系统也可以多次运行。

这时候靠 `session_id` 已经不够了。

还必须再回答：

**这条 `Event` 属于这个 session 里的哪一次 invocation？**

这就是 [`invocation_id`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:109) 的作用。

它让系统能在同一条连续历史里，再切出更细的执行边界。

### 2.2 `author`：不是只有 user 和 model，而是运行角色标识

[`author`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:111) 看起来很简单，但意义并不小。

它不是“UI 显示是谁说的话”这么浅。

它真正承担的是：

**告诉系统，这条事实是由哪个运行角色追加到事实流里的。**

在最简单情况下，当然可能是：

- `user`
- 某个 agent 名称

但当多个 agent 参与同一次 invocation 时，`author` 也是区分角色边界的基础信息之一。

### 2.3 `id` 和 `timestamp`：每条事实都要可唯一引用、可排序

[`id`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:154) 和 [`timestamp`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:156) 看似普通，但它们决定了两件事：

1. 每条事实能不能被唯一引用
2. 整条事实流能不能按时间稳定回放

你在 [Event.model_post_init()](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:271) 里会看到，如果没有外部提供 `id`，ADK 会主动生成一个新的 event id。

这说明 `Event` 从一开始就被当成正式记录对象，而不是临时数据包。

---

## 3. `Event` 不是从零开始定义的：它继承了 `LlmResponse`

这部分属于“核心执行主干 → events → Event 与 LlmResponse 的关系”分支。

先给一句话定义：

**`Event` 不是完全独立的数据类，而是在 `LlmResponse` 之上追加运行时语义。**

关键源码：

- [Event(LlmResponse)](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:93)
- [llm_response.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:29)

这一点非常重要，因为它决定了：

**`Event` 既是“系统事实”，又继承了“模型响应载体”的很多能力。**

先看 `LlmResponse` 里已经有的字段：

- [`content`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:62)
- [`partial`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:72)
- [`finish_reason`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:90)
- [`error_code`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:93)
- [`error_message`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:96)
- [`usage_metadata`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:112)
- [`input_transcription`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:126)
- [`output_transcription`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:129)

再看 `LlmResponse` 自带的方法：

- [`get_function_calls()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:158)
- [`get_function_responses()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:167)

这意味着很多“模型层面的内容解析能力”，`Event` 直接继承了。

### 3.1 这套继承关系解决了什么问题

如果 `Event` 不继承 `LlmResponse`，框架就要维护两套近似结构：

- 一套表示模型响应
- 一套表示系统事件

这样会带来两个坏处：

1. 模型内容和事件内容要反复互转
2. 工具调用、流式输出、错误信息这些字段会重复定义

而 ADK 的做法是：

**先承认“很多系统事实本来就长得像模型响应”，然后再把运行时附加信息叠上去。**

这是一种非常务实的设计。

### 3.2 这也意味着 `Event` 不只来自 model

不要因为它继承 `LlmResponse`，就误以为 `Event` 只能代表 model 输出。

恰恰相反。

`Event` 继承的是“内容和响应形状”，但它表达的事实来源可以很多：

- user 输入
- model 输出
- tool response
- workflow 控制事件
- rewind / compaction / UI 渲染信号

也就是说：

**继承 `LlmResponse` 只是为了复用内容协议，不是为了把 `Event` 限制成模型消息。**

这是你读源码时必须区分清楚的边界。

---

## 4. `EventActions`：真正的运行语义大多挂在这里

这部分属于“核心执行主干 → events → EventActions”分支。

先给一句话定义：

**`Event.content` 负责表达“看得见的内容”，`Event.actions` 负责表达“系统要据此做什么”。**

关键源码：

- [EventActions](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:52)

如果说 `Event` 是统一事实协议，那么 `EventActions` 就是这个协议里最像“控制面”的部分。

先看最核心的字段：

- [`state_delta`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:68)
- [`artifact_delta`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:71)
- [`transfer_to_agent`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:75)
- [`requested_auth_configs`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:81)
- [`requested_tool_confirmations`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:93)
- [`compaction`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:99)
- [`end_of_agent`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:102)
- [`agent_state`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:107)
- [`rewind_before_invocation_id`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:111)
- [`route`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:114)
- [`render_ui_widgets`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:117)

### 4.1 `state_delta`：上一章和这一章的真正接缝

[`state_delta`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:68) 是上一章 `sessions` 和这一章 `events` 的真正接缝。

上一章我们讲的是：

- `append_event()` 会消费 `event.actions.state_delta`
- 决定哪些状态写进 session
- 哪些 `temp:` 状态只留在当前 invocation

而这一章要补上的就是：

**状态变化为什么挂在 `Event` 上？**

答案是：

因为 ADK 认为“状态变化”本身也是一种运行事实，不该藏在框架内部。

这意味着：

- 不是某个 agent 私下改了内存
- 而是系统把“状态被怎样更新”也纳入了正式事实流

这就是事件化 runtime 最重要的收益之一。

### 4.2 `route`、`transfer_to_agent`、`end_of_agent`：这已经不是聊天协议了

很多人只看 `content`，会觉得 `Event` 还是比较像聊天消息。

但看到这些字段就该彻底改观：

- [`route`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:114)
- [`transfer_to_agent`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:75)
- [`end_of_agent`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:102)

这些字段表达的已经不是“说了什么”，而是：

- 接下来走哪条边
- 把控制权转给哪个 agent
- 当前 agent 这轮执行是否结束

也就是说，`Event` 同时承载了：

- 内容层事实
- 编排层事实

这就是它为什么能成为全框架的统一协议。

### 4.3 `requested_auth_configs` 和 `requested_tool_confirmations`：外部交互也走事件流

还有两个很有代表性的字段：

- [`requested_auth_configs`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:81)
- [`requested_tool_confirmations`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:93)

它们说明 ADK 连“工具请求认证”或“请求人工确认”这种事情，也不想做成框架外的隐式 side effect。

而是希望：

**把这些外部交互需求也显式挂进事实流。**

这会直接提升两种能力：

1. 可观测性  
   你能知道系统什么时候发起了认证或确认请求

2. 可恢复性  
   中断之后，系统能根据事件历史判断当前卡在哪一步

所以你会发现：

`EventActions` 其实是在告诉你，ADK 的“事件”从来不只是日志，而是实际运行协议的一部分。

---

## 5. `NodeInfo`、`branch`、`isolation_scope`：同一条事实流如何承载复杂 workflow

这部分属于“核心执行主干 → events → workflow 定位信息”分支。

先给一句话定义：

**`Event` 不只记录发生了什么，还记录这件事发生在执行图的什么位置。**

关键源码：

- [NodeInfo](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:35)
- [`branch`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:128)
- [`isolation_scope`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:137)

### 5.1 `NodeInfo`：workflow 节点位置不是额外查出来的，而是随事件携带

先看 `NodeInfo` 的核心字段：

- [`path`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:45)
- [`output_for`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:51)
- [`message_as_output`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:60)

它们的含义是：

- 这条事件来自执行图的哪个节点路径
- 这条事件的 `output` 算哪些节点的输出
- 这条 `content` 是否应直接当作节点输出

这说明 workflow 节点语义不是存放在别处，而是直接和事件绑定。

于是同一条 session 事实流里，系统依然能知道：

- 哪些事件来自哪个 node
- 哪些输出属于哪个 node
- 哪些内容要被当成 node output 使用

### 5.2 `branch`：解决并行或多 agent 视角污染

[`branch`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:128) 解决的是一个非常真实的问题：

**多个子执行路径并存时，谁该看到谁的历史？**

如果没有 branch 过滤：

- 并行节点会互相看到不属于自己的上下文
- 多个 sub-agent 的历史会污染彼此

这会直接破坏推理稳定性。

所以 branch 的本质不是“多存一个字符串”，而是：

**给同一条全局事实流加上可过滤的局部视角。**

### 5.3 `isolation_scope`：比 branch 更细的逻辑隔离

[`isolation_scope`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:137) 更值得注意。

源码注释已经写得很明确：

- 它是内部机制
- 不应该被外部业务直接依赖

但从学习角度，你必须知道它存在是为了解决什么问题：

**同一个 session 里，某些 task delegation 需要比 branch 更细粒度的上下文隔离。**

最典型的例子就是 task agent：

- coordinator 触发某个 task
- task agent 只该看到属于自己那段任务的事件
- 不该看到 coordinator 的整段广义历史

这就是为什么同一条 session 事实流还能支持局部隔离视图。

这里你应该顺手把上一章的一个点串起来：

- `Session` 保存的是整条事实流
- 但 active agent 最终看到的是经过 `branch` / `isolation_scope` 编排后的局部视图

这也为后面的 `workflow` 章节做了非常重要的铺垫。

---

## 6. `Event` 不是被动记录，它还定义了运行判断逻辑

这部分属于“核心执行主干 → events → Event 行为方法”分支。

先给一句话定义：

**`Event` 不只是数据结构，它还带着一部分对运行语义的判断方法。**

关键方法：

- [`is_final_response()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:277)
- [`has_trailing_code_execution_result()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:294)
- [`message` property](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:232)
- [`_accept_convenience_kwargs()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:159)

### 6.1 `is_final_response()`：结束判断不靠“最后一条消息长得像结尾”

先看 [`is_final_response()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:277)。

这个方法告诉你：

**ADK 判断“这是不是最终响应”，不是靠文案猜，而是靠事件结构判断。**

它的判断逻辑大致是：

- 如果需要跳过总结，或有 `long_running_tool_ids`，可以视为 final
- 否则要求：
  - 没有 function call
  - 没有 function response
  - 不是 partial
  - 没有 trailing code execution result

这说明在 ADK 里，“结束”本身也是一个结构化判断，而不是聊天 UI 的感觉判断。

### 6.2 `message` property：为什么要保留这个别名

[`message` property](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:232) 看起来像个小细节，但很体现框架兼容思路。

它允许：

- 外部把 `message` 当作 `content` 的别名来读写
- 同时又兼容某些 subclass 把 `message` 真正定义成字段

也就是说，ADK 在这里做的是：

**尽量让用户侧构造和框架内部结构保持一致，同时不破坏子类扩展。**

### 6.3 `_accept_convenience_kwargs()`：为什么 `Event` 初始化可以更像业务对象

[`_accept_convenience_kwargs()`](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:159) 是另一个很好的例子。

它允许这些便捷写法：

- `message` 自动路由到 `content`
- `state` 自动路由到 `actions.state_delta`
- `route` 自动路由到 `actions.route`
- `node_path` 自动路由到 `node_info.path`

这说明框架并不强迫所有调用方都手动构造完整嵌套结构。

它在保持协议统一的同时，也尽量让构造过程更接近业务语义。

这是一种非常实用的工程设计，而不是纯粹的模型洁癖。

---

## 7. 从 `sessions` 走到 `events`，你现在应该怎样重新理解事实流

这部分属于“核心执行主干 → events → 与上一章的承接”分支。

现在把第三章和第四章连起来看，你应该把它们理解成：

- `Session` 解决：事实放在哪条连续容器里
- `Event` 解决：每一条事实长成什么统一协议

也就是说：

**上一章讲的是“容器”，这一章讲的是“容器里的原子单元”。**

于是整条执行链就清楚了：

1. `Runner` 跑起来一次 invocation
2. 执行过程中不断产生 `Event`
3. `Session` 持续把 `Event` 保存下来
4. 后续模块再基于这些 `Event` 做：
   - workflow 编排
   - LLM context 组织
   - memory 摘要或检索
   - 回放与审计

所以不要把 `Event` 看小了。

在 ADK 里，它不是“日志实体”。

更准确地说：

**`Event` 是整个 runtime 的最小事实语言。**

---

## 8. 这一章读完后，推荐怎样继续读源码

这部分属于“核心执行主干 → events → 源码阅读顺序”分支。

推荐顺序如下：

1. [event.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:93)
   - 先看 `Event` 的字段
   - 再看 `NodeInfo`
   - 最后看 `is_final_response()` 和 `_accept_convenience_kwargs()`

2. [llm_response.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/models/llm_response.py:29)
   - 看 `Event` 继承了哪些响应层能力
   - 特别看 `content`、`partial`、`get_function_calls()`、`get_function_responses()`

3. [event_actions.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event_actions.py:52)
   - 重点看 `state_delta`
   - `route`
   - `end_of_agent`
   - `agent_state`
   - `rewind_before_invocation_id`

4. 回看 [03-sessions.md](/Users/a1111/Desktop/work/work_harness/adk-python/books/part-1-core-execution/03-sessions.md)
   - 把 `append_event()` 和 `Event.actions.state_delta` 再串一次

5. 再进入下一章 `workflow`
   - 因为 `NodeInfo`、`branch`、`route` 这些字段，真正的价值会在 workflow 里完全展开

---

## 9. 本章小结：`Event` 是 ADK 的统一事实语言

这一部分属于“核心执行主干 → events → 小结”分支。

这一章你至少应该记住四句话：

1. `Event` 不是聊天消息对象，而是 ADK 的最小运行事实单元。
2. `Event` 通过继承 `LlmResponse` 复用了内容、流式、function call/response 等模型层能力。
3. `EventActions` 让状态变化、agent transfer、workflow route、rewind、UI 渲染这些运行语义进入统一事实流。
4. `NodeInfo`、`branch`、`isolation_scope` 让同一条 session 历史能承载复杂 workflow，而不只是线性对话。

如果你把这一章学明白，就不会再把 ADK 理解成：

- “一个会存聊天记录的 Agent 框架”

而会开始把它理解成：

- **一个围绕统一事件事实流构建的运行时系统**

下一章，我们就继续顺着这个方向往下走：

**既然 `Event` 已经能承载这么多事实，那 `workflow` 又是怎样利用这些事实把复杂执行流组织起来的？**

---

📊 本次学习小结
- 模块：`events`（[src/google/adk/events/event.py](/Users/a1111/Desktop/work/work_harness/adk-python/src/google/adk/events/event.py:93)）
- 当前深度：L2（理解 How），局部进入 L3（理解为什么 ADK 要把内容、动作、控制统一成一种事实协议）
- 下一阶段建议：进入 `workflow`，把 `NodeInfo`、`route`、`end_of_agent` 这些字段放回编排层再看一遍
- 掌握程度自评：🟡 基本掌握（如果还没把 `EventActions` 和 `Session.append_event()` 串起来，建议回看上一章）
