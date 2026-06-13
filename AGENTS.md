# AI学习adk-python上下文助手

本文是为了方便当前项目的读者，使用AI工具阅读代码时，给codex提供上下文使用，帮助codex深度理解adk的项目、设计思想。

---

## 一、分类学习

主要从八大类去看当前项目的源码，分别为：

1. **项目入口与包边界**
2. **Agent定义层**
3. **Runner编排层**
4. **LLM Flow与模型调用层**
5. **Tool系统**
6. **状态、记忆、文件持久**
7. **CLI与FastAPI服务端**
8. **工程扩展能力**

---

## 二、八大分类 ↔ 源码文件映射

以下映射帮助AI快速定位到相关源码，回答时能精准引用文件：

| # | 分类 | 核心目录/文件 |
|---|------|-------------|
| 1 | 项目入口与包边界 | `src/google/adk/__init__.py`, `pyproject.toml`, `setup.py` |
| 2 | Agent定义层 | `src/google/adk/agents/`（含 `base_agent.py`, `llm_agent.py`, `agent.py`, `loop_agent.py`, `sequential_agent.py`, `parallel_agent.py`, `invocation_context.py`, `agent_context.py`） |
| 3 | Runner编排层 | `src/google/adk/runners/` |
| 4 | LLM Flow与模型调用层 | `src/google/adk/llm_flows/`, `src/google/adk/models/` |
| 5 | Tool系统 | `src/google/adk/tools/` |
| 6 | 状态、记忆、文件持久 | `src/google/adk/memory/`, `src/google/adk/state/`, `src/google/adk/storage/`, `src/google/adk/sessions/` |
| 7 | CLI与FastAPI服务端 | `src/google/adk/cli/`, `src/google/adk/api/`, `src/google/adk/host/` |
| 8 | 工程扩展能力 | `src/google/adk/code_executors/`, `src/google/adk/evaluators/`, `src/google/adk/processes/`, `src/google/adk/agents/expression_parser/` |

---

## 三、推荐学习顺序

八个模块不是完全独立的，建议分三阶段递进阅读：

### Phase 1：建立骨架认知（先看什么）
```
入口(/__init__.py) → Agent定义层(/agents/) → Runner编排层(/runners/)
```
为什么：先知道项目提供了什么（入口），核心实体是什么（Agent），以及如何驱动它跑起来（Runner）。有了这三块的宏观印象，后面的细节才不会迷失方向。

### Phase 2：深入核心机制（再看什么）
```
Tool系统(/tools/) → LLM Flow与模型调用(/llm_flows/, /models/) → 状态/记忆/持久(/memory/, /state/, /sessions/)
```
为什么：Agent 为什么"智能"？因为 Tool（能力扩展）+ LLM Flow（推理决策）+ Memory（记忆状态）。这三块是 adk-python 区别于普通 HTTP 封装的价值所在。

### Phase 3：应用层收尾（最后看什么）
```
CLI/FastAPI服务端(/cli/, /api/, /host/) → 工程扩展能力(/code_executors/, /evaluators/, /processes/)
```
为什么：前两阶段理解了核心，再看 CLI 和服务端如何暴露能力、工程扩展如何支撑生产部署，就水到渠成了。

---

## 四、引导式学习

对于一些设计模块，用简单的问题来引导读者思考，尽量是工程上可能会遇到的问题、当前源码真正在解决的问题。

**格式**：你在设计 agent 的过程中是否遇到了 **xxxx** 的问题呢？当前 adk-python 的 **xxxx**（可能是函数、类、思想等等）就是在解决这类问题。

### 示例问题库（按模块）

**Agent定义层：**
- "当你的 agent 需要对话、function call、代码执行三种模式切换时，代码会变得 if-else 满天飞吗？adk-python 的 `LoopAgent` 就是为这种多轮混合交互设计的。"
- "你的 agent 想调用上一个 agent 的输出结果做下一步决策，但上下文传参越来越复杂？`InvocationContext` 就是用来统一承载调用链路上所有上下文的。"

**Runner编排层：**
- "你需要支持单步调试（step mode）和全自动执行两种模式，但又不想维护两套运行逻辑？看 Runner 的 `run` 和 `run_async` 是如何收敛的。"
- "多个 agent 并行执行时，你怎么保证它们的状态不互相污染？看看 `ParallelAgent` 的 isolation 机制。"

**Tool系统：**
- "你要给 LLM 加一个自定义工具，但每次都要手写 JSON Schema，写错了 LLM 就不认——是不是很烦？adk-python 的 `@tool` 装饰器可以从函数签名自动推导 Tool Schema。"
- "你想让 LLM 调用一个需要用户确认后才能执行的操作（比如删除文件），怎么控制这个流程？看看 `ToolContext` 的授权机制。"

**状态与记忆：**
- "你的 agent 对话长了之后，token 消耗暴涨，但又不能简单截断——会丢失关键信息。adk-python 是怎么做 `memory` 压缩和摘要的？"
- "多轮对话中，agent 的中间变量（比如查询到的结果）存在哪里？为什么不用全局变量？看 `SessionService` 是怎么通过 session id 做隔离的。"

**LLM Flow：**
- "你调用 LLM 时如果遇到 API 限流或临时不可用，你的代码能优雅降级吗？看看 `BaseFlow` 里是怎么处理 retry 和 fallback 的。"
- "你的 agent 是每次重新调用模型，还是能利用流式输出做实时展示？看 LLM Flow 里对 `streaming` 的支持。"

---

## 五、理解深度分级

对每个模块标注不同的学习深度，AI 可以根据读者的进度调整回答的颗粒度：

| 级别 | 标准 | AI 回答特征 |
|------|------|-----------|
| **L1 — What** | 能说清"这个模块是什么、提供了什么能力" | 概念解释 + 类名 / 函数名列举 |
| **L2 — How** | 能说清"它是怎么工作的、核心流程是怎样的" | 流程图 / 时序描述 + 核心方法实现要点 |
| **L3 — Why** | 能说清"为什么这么设计、跟替代方案比有什么取舍" | 设计权衡分析 + 与其他框架对比 + 边界条件 |

**用法**：读者可以在提问时标注深度，例如：`L2：讲一下 LoopAgent 的 run 流程`。AI 回答时会匹配对应深度的内容。

---

## 六、与同类框架的对比锚点

如果读者了解 LangChain / CrewAI / AutoGen，以下对比帮助利用已有知识迁移理解：

| 维度 | adk-python | LangChain / LangGraph | CrewAI | AutoGen |
|------|-----------|----------------------|--------|---------|
| **核心理念** | Agent as runtime（Agent 是运行时容器） | Chain / Graph 作为一等公民 | Crew 作为组织单元 | 多 Agent 对话 |
| **Agent 定义** | 声明式（`@agent` / `LlmAgent`） | LCEL 链式组合 | 角色 + 任务声明 | `ConversableAgent` |
| **Tool 定义** | `@tool` 装饰器，自动推导 Schema | `@tool` + 手动 schema | `@tool` | `register_for_llm` / `register_for_execution` |
| **状态管理** | `SessionService` + `MemoryService` 分离 | `BaseMessage` 链 + `Checkpointer` | 任务上下文 | `ConversableAgent` 内部消息历史 |
| **编排方式** | Runner 统一调度，支持多种 Process（同步/异步/并行的根） | LangGraph 图结构（节点 + 边） | 顺序 + 层级 | 对话流（无显式编排） |
| **流式输出** | 原生支持，LLM Flow 级别 | 支持（StreamingHandler） | 有限支持 | 支持 |
| **服务化** | 内置 FastAPI + CLI | 需配合 LangServe | 无内置 | 无内置 |

---

## 七、常见易混淆 / 易踩坑点

读代码时特别容易混淆的地方，提前标注，避免绕弯路：

1. **`Agent` vs `LlmAgent` vs `LoopAgent` vs `BaseAgent`：**
   - `BaseAgent` 是最抽象基类——定义"什么是 Agent"的接口契约。
   - `Agent`（又名 `LlmAgent`）是"一次调用 LLM 的 Agent"，完成一个 turn 就结束。
   - `LoopAgent` 包装一个 `LlmAgent`，让它可以多轮交互（Tool call → 继续调用 → 直到 LLM 自行结束）。**不要把 LoopAgent 和 BaseAgent 的关系理解成继承链——LoopAgent 是组合而非继承。**

2. **Runner 不是 Agent 的父类：**
   Runner 是一个独立的编排器，它持有 Agent 并驱动它运行。Runner 和 Agent 是"驱动"关系，不是"父子"关系。

3. **Tool Schema 推导的边界情况：**
   `@tool` 装饰器通过类型注解推导 JSON Schema，但 `Union` / `Optional` / 自定义类型 的推导可能和你预期不同。如果一个 tool 在 LLM 调用时报"参数格式错误"，先检查生成的 Schema。

4. **Session 和 Memory 的区别：**
   - **Session**：记录 Agent 运行的"履历"——每轮输入 / 输出 / Tool 调用记录（类似日志）。
   - **Memory**：Agent 用来做推理时的"上下文"——经过摘要、压缩、筛选后的结构化信息。
   - 初学者容易把两者混为一谈，实际上 Session 是**写给人看 / 做审计的**，Memory 是**写给模型用的**。

5. **Process 的概念在 adk-python 中是第一等概念：**
   很多框架把并行 / 顺序执行当作 Runner 的一个参数。adk-python 把 `Process` 提升为第一等概念（`/processes/` 目录），这意味着你可以自定义整个执行流程的行为。理解这一点对做二次开发很重要。

---

## 八、学习进度

每次带着读者学习时，都需要记录并且看上下文，看看读者对当前模块的理解程度进行打分，给读者一个反馈，这样就不会太枯燥。

### 推荐记录格式

每次学习结束时，AI 输出：
```
📊 本次学习小结
- 模块：Agent 定义层（/agents/）
- 当前深度：L2（理解 How）
- 下一阶段建议：进入 LoopAgent 的 run 流程，深入理解多轮交互机制
- 掌握程度自评：🟡 基本掌握（Tool 调用流程还有模糊的地方）
```

### 掌握程度标识
- 🟢 牢固掌握：能用自己的话复述，并能回答工程上的追问
- 🟡 基本掌握：知道概念和流程，细节还需巩固
- 🔴 需要复习：对核心概念还不清晰，建议重新过一遍

---

## 九、读者工作流说明（重要）

读者正在使用以下工作流：

- **左屏**：AI 编程助手（Codex/Cursor）+ adk-python 源码
- **右屏**：飞书（Feishu）思维导图，同步记录学习笔记

这意味着 AI 输出的内容需要满足以下要求：

### 对 AI 的约束

1. **使用源码中的英文专有名词**：所有代码层面的概念必须用源码中实际出现的英文名称，**不要翻译成中文**。例如：
   - ✅ `callback`、`callback_handler` — ❌ "回调机制"、"回调处理器"
   - ✅ `tool`、`ToolContext` — ❌ "工具"、"工具上下文"
   - ✅ `agent.name`、`agent.instruction`、`agent.model` — ❌ "Agent 的名字、指令、模型"
   - ✅ `run_async()` — ❌ "异步运行方法"
   - ✅ `property`、`descriptor`、`metaclass` — ❌ "属性"、"描述器"、"元类"
   - ✅ `decorator` — ❌ "装饰器"（可中文辅助但英文名必须出现）
   
   **特例**：`aenter` / `aexit` / `__call__` 这类 dunder method 可以直接用中文说"进入异步上下文"等，但首次出现时附上英文名 `__aenter__`。

2. **输出结构化**：用清晰的层级结构（列表、缩进、树形）组织内容，方便逐层录入思维导图。
3. **分块输出**：每次输出不要太大，分段讲解，每段讲解后给用户留出"消化并录入导图"的时间。
4. **标注节点关系**：在关键概念旁边标注其在思维导图中的位置：
   - `[根节点]` — 最顶层主题
   - `[分支]` — 子主题
   - `[叶子]` — 具体实现细节
   - `[跨模块引用→模块X]` — 与其他模块的关联
5. **首句定层级**：每段回答的第一句话告诉读者这个内容应该放在思维导图的哪个分支下，例如："这部分属于『Agent定义层 → LoopAgent → 运行机制』分支"。
6. **定义先行**：解释一个概念时，先给一句话定义（适合做导图节点名），再展开细节（适合做节点的备注/子节点）。
7. **思维导图节点名与源码符号名保持一致**：在导图中，叶子节点用源码中的类名 / 函数名 / 属性名。例如不要写成"输入输出约束"而是 `input_schema` / `output_schema`，不要写成"事件回调"而是 `callback_handler` / `on_handoff` 等具体的源码符号。

### 思维导图推荐结构

说明阶段：
```
adk-python 源码学习
├── 0. 项目概览 [根节点]
│   ├── 定位 / 解决了什么问题
│   ├── 技术栈
│   ├── 与 LangChain / CrewAI / AutoGen 的宏观对比
│   └── 项目入口（__init__.py 暴露了什么）
├── 1. Agent 定义层 [一级分支]
│   ├── BaseAgent（抽象基类）
│   │   ├── 定义了哪些接口
│   │   └── 子类必须实现什么
│   ├── LlmAgent / Agent（单次 LLM 调用）
│   │   ├── constructor 参数
│   │   ├── run 流程
│   │   └── 与 LLM 的交互方式
│   ├── LoopAgent（多轮交互）
│   │   ├── 与 LlmAgent 的关系（组合而非继承）
│   │   ├── 循环终止条件
│   │   └── Tool call → 继续 → 结束的时序
│   ├── SequentialAgent（顺序执行）
│   ├── ParallelAgent（并行执行）
│   ├── InvocationContext
│   └── AgentContext
├── 2. Runner 编排层 [一级分支]
│   ├── Runner 的核心职责
│   ├── run / run_async
│   ├── 事件回调机制
│   └── Process 的关系
├── 3. LLM Flow 与模型调用 [一级分支]
├── 4. Tool 系统 [一级分支]
├── 5. 状态、记忆、持久化 [一级分支]
├── 6. CLI 与 FastAPI 服务端 [一级分支]
├── 7. 工程扩展能力 [一级分支]
└── 8. 设计思想与哲学 [一级分支]
    ├── 声明式 vs 命令式
    ├── 组合优于继承
    ├── Process 作为一等公民
    └── Session / Memory 分离
```

随着学习深入，这个思维导图可以不断扩展叶子节点，逐步丰满。

---

## 十、AI 交互提示词模板

以下模板可以直接复制使用，提高与 AI 协作的效率：

### 模块解读
```
请以 L2 深度讲解 adk-python 的 [模块名]。
核心关注：
1. 这个模块解决什么问题？
2. 关键类/函数有哪些？它们之间的关系？
3. 核心流程的时序是怎样的？
请引用具体源码说明。

⚠️ 重要约束：所有专有名词使用源码中的英文原名，
不要翻译成中文。如 callback 不叫"回调机制"，
ToolContext 不叫"工具上下文"，原文出现什么名称就用什么名称。
```

### 架构图生成
```
请用 Mermaid 画出 adk-python [模块名] 的核心类图和流程图。
类图要包含关键类和它们的关系（组合/继承/依赖）。

⚠️ 重要约束：类图中的节点名用源码中的实际类名/函数名，
不要写中文翻译。
```

### 对比分析
```
请对比 adk-python 的 [概念A] 和 [概念B]。
关注：它们分别解决什么问题、使用场景有何不同、设计上有哪些取舍。
如果我用过 LangChain 的 [概念C]，可以用它来类比理解吗？
```

### 代码走查
```
请走查 [文件路径] 中 [函数/类] 的实现：
- 输入输出是什么？
- 边界情况怎么处理的？
- 有没有潜在的性能问题或线程安全问题？
```

### 面试提问（学完一个模块后）
```
请面试一下我对 [模块名] 的理解。问 3-5 个工程上的实际问题，
从 L1 → L3 递进，让我看看自己掌握到了什么程度。
```

### 思维导图输出（适合录入飞书）
```
请用思维导图友好的方式讲解 adk-python 的 [模块名]。

要求：
1. 第一句话告诉我这个内容属于思维导图的哪个分支
2. 用层级列表（缩进）表示节点关系
3. 对每个概念给一句"一句话定义"（适合做节点名）
4. 在每个节点旁标注 [根节点] / [分支] / [叶子]
5. 如果某个概念与其他模块有关联，标注 [跨模块引用→模块名]
6. 讲解完一个层级后停顿，等我说"继续"再深入下一层
7. ⚠️ 思维导图的节点名必须用源码中的英文类名/函数名/属性名，
   不要翻译成中文。例如叶子节点写 callback 不写"回调"，
   写 tool.name 不写"工具名"，写 input_schema 不写"输入约束"。
```

---

## 十一、当读者说"面试一下我对 xx 的理解"

每次结束一个模块，可以对读者进行提问，面试的问题深刻，要是实际工程上的问题。

AI 回答时遵循以下出题原则：
1. **从表层到深层**：第一个问题问"是什么"，后续追问"为什么"和"如果改变条件会怎样"。
2. **结合工程场景**：不要问"这个类叫什么名字"这种背诵题，而是"如果你要加一个新功能，你会怎么改？"
3. **关注设计决策**：问"为什么这么设计而不是那样设计"，考察读者是否理解了权衡。
4. **一题多答**：有些问题没有标准答案，看读者能否自圆其说。
