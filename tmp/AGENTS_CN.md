# AI 编码助手上下文

本文档为 AI 编码助手（Claude Code、Gemini CLI、GitHub Copilot、Cursor 等）提供上下文，帮助它们理解 ADK Python 项目并协助开发。

## 项目概览

Agent Development Kit（ADK）是一个开源、代码优先的 Python 工具包，用于以灵活且可控的方式构建、评估和部署复杂的 AI Agent。虽然 ADK 针对 Gemini 和 Google 生态进行了优化，但它本身是模型无关、部署无关的，并且设计上兼容其他框架。ADK 的设计目标是让 Agent 开发更像软件开发，使开发者更容易创建、部署和编排从简单任务到复杂工作流的各种 Agent 架构。

### 核心组件

-   **Agent** - 定义身份、指令和工具的蓝图（`LlmAgent`、`LoopAgent`、`ParallelAgent`、`SequentialAgent` 等）。
-   **Runner** - 编排 Agent 执行的执行引擎。它管理 “Reason-Act” 循环，处理 session 中的消息，生成事件，调用 LLM，执行工具，并处理多 Agent 协作。它会与 session 管理、artifact 存储、memory 等服务交互，并集成应用级插件。Runner 提供不同执行模式：`run_async` 用于生产环境中的异步执行，`run_live` 用于双向流式交互，`run` 用于适合本地测试和调试的同步执行。每次 invocation 结束时，它可以执行事件压缩，以管理 session 历史大小。
-   **Tool** - Agent 可以调用的函数或能力（Python 函数、OpenAPI 规范、MCP 工具、Google API 工具）。
-   **Session** - 对话状态管理（内存、Vertex AI、Spanner 后端）。
-   **Memory** - 跨 session 的长期记忆。

### Runner 如何工作

Runner 是管理 Agent 执行的无状态编排引擎。它不会在内存中持有对话历史，而是依赖 `SessionService`、`ArtifactService` 和 `MemoryService` 等服务进行持久化。

**Invocation 生命周期：**

每次调用 `runner.run_async()` 或 `runner.run()` 都会处理一个用户回合，称为一次 **invocation**。

1.  **Session 获取：** 当调用 `run_async()` 并传入 `session_id` 时，runner 会从 `SessionService` 获取 session 状态，包括所有对话事件。
2.  **上下文创建：** 它会创建一个 `InvocationContext`，其中包含 session、新的用户消息，以及对持久化服务的引用。
3.  **Agent 执行：** Runner 使用这个上下文调用 `agent.run_async()`。随后 Agent 进入 reason-act 循环，可能包括：
    *   调用 LLM 进行推理。
    *   执行工具（函数调用）。
    *   生成文本或音频响应。
    *   将控制权转交给子 Agent。
4.  **事件流与持久化：** Agent 执行中的每一步（LLM 调用、工具调用、工具响应、模型响应）都会生成 `Event` 对象。Runner 将这些事件流式返回给调用方，同时通过 `SessionService` 将它们追加到 session 中。
5.  **Invocation 完成：** 一旦 Agent 为当前回合生成最终响应（例如给用户的文本响应），Agent 的执行循环就会结束。
6.  **事件压缩：** 如果配置了事件压缩，runner 可能会总结 session 中较早的事件，以管理上下文窗口限制，并向 session 追加一个 `CompactedEvent`。
7.  **下一回合：** 当用户发送另一条消息时，一个新的 `run_async()` invocation 会开始。它会重新加载 session，而该 session 现在已经包含此前所有回合产生的事件。

## 项目架构

更多细节请参考 [ADK Project Overview and Architecture](https://github.com/google/adk-python/blob/main/contributing/adk_project_overview_and_architecture.md)。

### 源码结构

```
src/google/adk/
├── agents/          # Agent 实现（LlmAgent、LoopAgent、ParallelAgent 等）
├── runners.py       # 核心 Runner 编排类
├── tools/           # Tool 生态（50+ 文件）
│   ├── google_api_tool/
│   ├── bigtable/, bigquery/, spanner/
│   ├── openapi_tool/
│   └── mcp_tool/    # Model Context Protocol
├── models/          # LLM 集成（Gemini、Anthropic、LiteLLM）
├── sessions/        # Session 管理（内存、Vertex AI、Spanner）
├── memory/          # 长期记忆服务
├── evaluation/      # 评估框架（47 个文件）
├── cli/             # CLI 工具和 Web UI
├── flows/           # 执行流编排
├── a2a/             # Agent-to-Agent 协议
├── telemetry/       # 可观测性与 tracing
└── utils/           # 工具函数
```

### 测试结构

```
tests/
├── unittests/       # 2600+ 单元测试，覆盖 236+ 文件
│   ├── agents/
│   ├── tools/
│   ├── models/
│   ├── evaluation/
│   ├── a2a/
│   └── ...
└── integration/     # 集成测试
```

### ADK Live（双向流式）

-   ADK live 功能可以通过 `runner.run_live(...)` 和对应的 FAST api endpoint 使用。
-   ADK live 构建在 [Gemini Live API](https://cloud.google.com/vertex-ai/generative-ai/docs/live-api) 之上。我们通过 [GenAI SDK](https://github.com/googleapis/python-genai) 集成 Gemini Live API。
-   ADK live 相关配置位于 [run_config.py](https://github.com/google/adk-python/blob/main/src/google/adk/agents/run_config.py)。
-   多 Agent 场景下的 ADK live：我们会将音频转换为文本。该文本会作为上下文传递给下一个 Agent。
-   大多数逻辑位于 [base_llm_flow.py](https://github.com/google/adk-python/blob/main/src/google/adk/flows/llm_flows/base_llm_flow.py) 和 [gemini_llm_connection.py](https://github.com/google/adk-python/blob/main/src/google/adk/models/gemini_llm_connection.py)。
-   输入转写和输出转写应该作为 Event 添加到 session。
-   用户音频或模型音频应该保存到 artifacts 中，并在 Event 中引用它。
-   测试位于 [tests/unittests/streaming](https://github.com/google/adk-python/tree/main/tests/unittests/streaming)。

### Agent 结构约定（必需）

**所有 Agent 目录都必须遵循此结构：** `my_agent/ ├── __init__.py # 必须包含：from . import agent └── agent.py # 必须定义：root_agent = Agent(...) 或 app = App(...)`

**根据你的需求选择一种模式：**

**选项 1 - 简单 Agent（适用于没有插件的基础 Agent）：** ```python from google.adk.agents import Agent from google.adk.tools import google_search

root_agent = Agent( name="search_assistant", model="gemini-2.5-flash",
instruction="You are a helpful assistant.", description="An assistant that can
search the web.", tools=[google_search] ) ```

**选项 2 - App 模式（当你需要插件、事件压缩、自定义配置时）：** ```python from google.adk import Agent from google.adk.apps
import App from google.adk.plugins import ContextFilterPlugin

root_agent = Agent( name="my_agent", model="gemini-2.5-flash", instruction="You
are a helpful assistant.", tools=[...], )

app = App( name="my_app", root_agent=root_agent, plugins=[
ContextFilterPlugin(num_invocations_to_keep=3), ], ) ```

**理由：** 这种结构允许 ADK CLI（`adk web`、`adk run` 等）在不需要额外配置的情况下自动发现并加载 Agent。

## 开发环境设置

### 要求

**最低要求：**

- Python 3.10+（**强烈推荐 Python 3.11+**，性能最好）
- `uv` 包管理器（**必需**，比 pip/venv 更快）

**如果尚未安装 uv：** `bash curl -LsSf https://astral.sh/uv/install.sh | sh`

### 设置说明

**标准开发环境设置：** ```bash

# 使用 Python 3.11 创建虚拟环境

uv venv --python "python3.11" ".venv" source .venv/bin/activate

# 安装所有开发依赖

uv sync --all-extras ```

**仅测试所需的最小设置（与 CI 匹配）：** `bash uv sync --extra test`

**虚拟环境使用（必需）：** - **始终直接使用** `.venv/bin/python` 或 `.venv/bin/pytest` - **或者** 先通过 `source .venv/bin/activate` 激活环境 - **绝不要使用** `python -m venv` - 如果缺少虚拟环境，始终用 `uv venv` 创建。

**理由：** `uv` 明显更快，并能保证团队内依赖解析一致。

### 构建

```bash
# 构建 wheel
uv build

# 安装本地构建用于测试
pip install dist/google_adk-<version>-py3-none-any.whl
```

### 本地运行 Agent

**用于交互式开发和调试：** ```bash

# 启动 Web UI（推荐用于开发）

adk web path/to/agents_dir ```

**用于基于 CLI 的测试：** ```bash

# 交互式 CLI（提示用户输入）

adk run path/to/my_agent ```

**用于 API/生产模式：** ```bash

# 启动 FastAPI server

adk api_server path/to/agents_dir ```

**用于运行评估：** ```bash

# 针对 Agent 运行 eval set

adk eval path/to/my_agent path/to/eval_set.json ```

## ADK：风格指南

### Python 风格指南

项目遵循 Google Python Style Guide。关键约定通过 `pylint` 和提供的 `pylintrc` 配置文件强制执行。以下是一些关键风格点：

*   **缩进**：2 个空格。
*   **行长度**：最大 80 个字符。
*   **命名约定**：
    *   `function_and_variable_names`：`snake_case`
    *   `ClassNames`：`CamelCase`
    *   `CONSTANTS`：`UPPERCASE_SNAKE_CASE`
*   **Docstring**：所有 public module、function、class 和 method 都必须有 docstring。
*   **Imports**：组织并排序。
*   **错误处理**：应捕获具体异常，而不是像 `Exception` 这样的通用异常。

### 自动格式化（提交前必需）

**提交代码前始终运行：** `pre-commit run --all-files`

**只检查格式而不修改：** `bash pyink --check --diff --config pyproject.toml src/ isort --check src/`

**格式标准（由 CI 强制）：** - **Formatter:** `pyink`（Google 风格 Python formatter） - **行长度：** 最多 80 个字符 - **缩进：** 2 个空格（绝不使用 tab） - **Import sorter：** 使用 Google profile 的 `isort` - **Linter：** 使用 Google Python Style Guide 的 `pylint`

**理由：** 一致的格式可以消除风格争论，使 code review 聚焦于逻辑而不是样式。

### 在 ADK 源码中

以下风格适用于 ADK 源码（GitHub 仓库的 `src/` 目录下）。

#### 使用相对导入（必需）

```python
# DO - 使用相对导入
from ..agents.llm_agent import LlmAgent

# DON'T - 不要使用绝对导入
from google.adk.agents.llm_agent import LlmAgent
```

**理由：** 相对导入让代码更易维护，并避免大型代码库中的循环导入问题。

#### 从模块导入，不要从 `__init__.py` 导入（必需）

```python
# DO - 直接从模块导入
from ..agents.llm_agent import LlmAgent

# DON'T - 不要从 __init__.py 导入
from ..agents import LlmAgent
```

**理由：** 直接模块导入让依赖更明确，也改善 IDE 导航和重构体验。

#### 始终使用 `from __future__ import annotations`（必需）

**规则：** 每个源文件都必须在 license header 之后、任何其他 import 之前包含 `from __future__ import annotations`。

```python
# Copyright 2026 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

from __future__ import annotations  # REQUIRED - 始终包含这一行

# ... rest of imports ...
```

**理由：** 这允许不加引号地前向引用类，提高代码可读性和类型提示支持（PEP 563）。

### 在 ADK 测试中

#### 使用绝对导入（必需）

**规则：** 测试代码必须使用绝对导入（`google.adk.*`），以匹配用户导入 ADK 的方式。

```python
# DO - 使用绝对导入
from google.adk.agents.llm_agent import LlmAgent

# DON'T - 测试中不要使用相对导入
from ..agents.llm_agent import LlmAgent
```

**理由：** 测试应该使用与用户相同的导入路径，从而捕获 public API 的问题。

## ADK：本地测试

### 单元测试

**快速开始：** 使用以下命令运行所有测试：`bash pytest tests/unittests`

**推荐：** 提交 PR 前匹配 CI 配置：`bash uv sync --extra test && pytest tests/unittests`

**其他选项：** ```bash

# 并行运行测试以加快执行

pytest tests/unittests -n auto

# 开发期间运行指定测试文件

pytest tests/unittests/agents/test_llm_agent.py

```

### 测试理念

**使用真实代码而不是 mock：** ADK 测试应尽可能使用真实实现，而不是 mock。只 mock 外部依赖，例如网络调用或云服务。

**测试接口行为，而不是实现细节：** 测试应该验证 public API 的行为是否正确，而不是验证内部如何实现。这让测试在重构时更稳定，并确保与用户的契约保持不变。

**测试要求：** - 尽可能快速且隔离 - 使用真实 ADK 组件；只 mock 外部依赖（LLM API、云服务等） - 聚焦 public interface 和行为，而不是内部实现 - 测试名称要能描述正在验证的行为 - 新功能、边界情况和错误条件要有高覆盖 - 位置：`tests/unittests/`，并遵循源码结构。

## Docstring 和注释

### 注释 - 解释为什么，而不是解释做了什么

理念：写得好的代码应该基本自解释。注释承担不同职责：它们应该解释复杂算法、不明显的业务逻辑，或某个实现选择背后的理由，也就是代码本身无法表达的内容。避免只是复述代码做了什么的注释（例如 `# increment i above i += 1`）。

风格：注释应写成完整句子。块注释必须以 `#` 后跟一个空格开头。

## 版本管理

ADK 遵循 Semantic Versioning 2.0.0。

核心原则：adk-python 项目严格遵循 Semantic Versioning 2.0.0 规范。所有发布版本都采用 MAJOR.MINOR.PATCH 格式。

### Breaking Change

Breaking change 是任何引入向后不兼容 public API 变更的修改。在 ADK 的上下文中，这意味着该变更可能迫使使用该框架的开发者修改现有代码才能升级到新版本。Public API 不仅限于 Python 函数和类签名；它还包括已存储信息的数据 schema（如评估数据集）、命令行接口（CLI），以及服务端通信所使用的数据格式。

### Public API Surface 定义

ADK 的 “public API” 是一个广泛契约，超越了 Python 函数签名。以下任一区域中的 breaking change 都可能破坏用户工作流，以及围绕 ADK 构建的更广泛 Agent 和 Tool 生态。v1.0.0 中引入的 breaking changes 分析展示了该契约的广泛性。出于版本管理目的，ADK Public API Surface 定义如下：

-   `google.adk` namespace 中所有 public class、method 和 function。

-   所有内置 Tools 的名称、必需参数和预期行为（例如 `google_search`、`BuiltInCodeExecutor`）。

-   持久化数据的结构和 schema，包括 Session 数据、Memory 和 Evaluation datasets。

-   ADK API server（FastAPI server）被 `adk web` 使用的 JSON request/response 格式，包括字段大小写约定。

-   命令行接口（CLI）的 commands、arguments 和 flags（例如 `adk deploy`）。

-   被框架加载的 Agent 定义所期望的文件结构（例如 `agent.py` 约定）。

#### Breaking Changes 检查清单：

以下变更被视为 breaking，需要提升 MAJOR 版本号。

-   API Signature Change：重命名、删除或修改任何 public class、method 或 function 的必需参数（例如从 `BaseSessionService` 中移除 `list_events` 方法）。

-   Architectural Shift：核心组件行为发生根本性变化（例如将所有 service method 都改为 async，迫使消费者使用 await）。

-   Data Schema Change：持久化数据 schema 中的非追加式变更，导致旧数据无法读取或无效（例如重新设计 MemoryService 和 evaluation dataset schema）。

-   Tool Interface Change：重命名内置 tool，改变其必需参数，或改变其基本用途（例如用 `BuiltInCodeExecutor` 替换 `BuiltInCodeExecutionTool`，并将其从 Agent 的 tools 参数移动到 code_executor 参数）。

-   Configuration Change：改变框架加载的配置文件或 Agent 定义文件的必需结构（例如简化 MCPToolset 的 `agent.py` 结构）。

-   Wire Format Change：修改 API server 交互的数据格式（例如将所有 JSON payload 从 snake_case 切换为 camelCase）。

-   Dependency Removal：移除对此前集成的第三方库或 tool 类型的支持。

## Commit Message 格式（必需）

**所有 commit 都必须** 遵循 [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) 格式。

**格式：** ``` <type>(<scope>): <description>

[optional body]

[optional footer] ```

**常见类型：** `feat`、`fix`、`refactor`、`docs`、`test`、`chore`

**示例：** ``` feat(agents): Add support for App pattern with plugins

fix(sessions): Prevent memory leak in session cleanup

refactor(tools): Unify environment variable enabled checks ```

**理由：** Conventional commits 支持自动生成 changelog 和版本管理。

## 关键文件和位置

重要项目文件快速参考：

-   **主配置：** `pyproject.toml`（使用 `flit_core` build backend）
-   **依赖：** `uv.lock`（由 `uv` 管理）
-   **Linting：** `pylintrc`（Google Python Style Guide）
-   **自动格式化：** `pre-commit`（运行 isort + pyink）
-   **在所有文件上运行：** `pre-commit run --all-files`
-   **安装为 hook：** `pre-commit install`
-   **CLI 入口：** `src/google/adk/cli/cli_tools_click.py`
-   **Web UI 后端：** `src/google/adk/cli/adk_web_server.py`
-   **主导出：** `src/google/adk/__init__.py`（导出 Agent、Runner）
-   **示例：** `contributing/samples/`（100+ Agent 实现）

## 其他资源

- **文档：** https://google.github.io/adk-docs
- **示例：** https://github.com/google/adk-samples
- **架构详情：** `contributing/adk_project_overview_and_architecture.md`
- **贡献指南：** `CONTRIBUTING.md`
- **LLM 上下文：** `llms.txt`（摘要版）、`llms-full.txt`（完整版）

## Python 技巧

### 通用 Python 最佳实践

*   **Constants：** 使用不可变的全局常量集合（tuple、frozenset、immutabledict）以避免难以发现的 bug。优先使用常量，而不是散落的字符串/整数 literal，特别是 dictionary key、pathname 和 enum。
*   **Naming：** 对映射使用类似 `value_by_key` 的命名，以提高查找时的可读性（例如 `item = item_by_id[id]`）。
*   **Readability：** 使用 f-string 进行简洁的字符串格式化，但日志使用惰性求值的 `%` 风格模板。使用 `repr()` 或 `pprint.pformat()` 输出便于人类阅读的 debug 信息。使用 `_` 作为数字 literal 的分隔符，提高可读性。
*   **Comprehensions：** 使用 list、set 和 dict comprehensions 简洁地构建集合。
*   **Iteration：** 直接迭代容器，不要使用索引。需要索引时使用 `enumerate()`，需要键和值时使用 `dict.items()`，并行迭代时使用 `zip()`。
*   **Built-ins：** 利用 `all()`、`any()`、`reversed()`、`sum()` 等内置函数，写出更简洁高效的代码。
*   **Flattening Lists：** 使用 `itertools.chain.from_iterable()` 高效地扁平化 list of lists，避免不必要的复制。
*   **String Methods：** 使用 `startswith()` 和 `endswith()` 配合字符串 tuple 一次检查多个前缀或后缀。
*   **Decorators：** 使用 decorator 为函数添加通用功能（如 logging、timing、caching），而不修改核心逻辑。使用 `functools.wraps()` 保留原函数元数据。
*   **Context Managers：** 使用 `with` 语句和 context manager（来自 `contextlib` 或自定义 `__enter__`/`__exit__` 的类），确保资源即使在异常情况下也能正确初始化和清理。
*   **Else Clauses：** 使用 `try/except` 中的 `else` 子句（无异常时运行），以及 `for/while` 循环中的 `else` 子句（循环未被 `break` 中断时运行），写出更具表达力且更不容易出错的代码。
*   **Single Assignment：** 优先使用单次赋值形式（一个变量只赋值一次），而不是先赋值再变更，以减少 bug 并提升可读性。适当使用条件表达式。
*   **Equality vs. Identity：** 对 singleton 比较使用 `is` 或 `is not`（例如 `None`、`True`、`False`）。值比较使用 `==`。
*   **Object Comparisons：** 实现自定义类时，要谨慎处理 `__eq__`。对于未处理类型返回 `NotImplemented`。考虑子类和 hashing 等边界情况。优先使用 `attrs` 或 `dataclasses` 自动处理。
*   **Hashing：** 如果对象相等，它们的 hash 必须相等。确保 `__hash__` 使用的属性是不可变的。如果实现了自定义 `__eq__` 但没有正确的 `__hash__`，请使用 `__hash__ = None` 禁用 hashing。
*   **`__init__()` vs. `__new__()`：** `__new__()` 创建对象，`__init__()` 初始化对象。对于不可变类型，修改必须发生在 `__new__()` 中。
*   **Default Arguments：** 绝不要使用可变默认参数。使用 `None` 作为 sentinel value。
*   **`__add__()` vs. `__iadd__()`：** `x += y`（in-place add）如果实现了 `__iadd__`，可能会就地修改对象（例如 list）；而 `x = x + y` 会创建新对象。当多个变量引用同一对象时，这一点很重要。
*   **Properties：** 仅在需要时使用 `@property` 创建 getter 和 setter，同时保持简单属性访问语法。避免为计算昂贵或可能失败的操作使用 property。
*   **Modules for Namespacing：** 使用 module 作为分组和命名空间的主要机制，而不是 class。避免使用 `@staticmethod` 和不使用 `self` 的 method。
*   **Argument Passing：** Python 是 call-by-value，其中传递的 value 是对象引用（指针）。赋值会把名字绑定到对象。通过一个名字修改可变对象会影响绑定到同一对象的所有名字。
*   **Keyword/Positional Arguments：** 使用 `*` 强制 keyword-only 参数，使用 `/` 强制 positional-only 参数。这可以防止参数顺序错误，并让 API 更清晰，尤其是当函数有多个相同类型参数时。
*   **Type Hinting：** 使用类型注解提高可读性、可调试性和可维护性。容器注解使用 `collections.abc` 中的抽象类型（例如 `Sequence`、`Mapping`、`Iterable`）。为返回值添加注解，包括 `None`。为函数参数和返回值选择最合适的抽象类型。
*   **`NewType`：** 使用 `typing.NewType` 基于 primitive 创建不同类型（如 `int` 或 `str`），以防止参数顺序错误并提升类型安全。
*   **`__repr__()` vs. `__str__()`：** 实现 `__repr__()` 用于无歧义、面向开发者的字符串表示，最好是可求值的。实现 `__str__()` 用于人类可读输出。`__str__()` 默认使用 `__repr__()`。
*   **F-string Debug：** 使用 `f"{expr=}"` 进行简洁调试输出，同时展示表达式及其值。

### 库和工具

*   **`collections.Counter`：** 用于高效统计 iterable 中的 hashable object。
*   **`collections.defaultdict`：** 适用于初始化 dictionary value 时避免 key 检查，例如向 list 追加元素。
*   **`heapq`：** 使用 `heapq.nlargest()` 和 `heapq.nsmallest()` 高效查找最大/最小的 N 个元素。使用 `heapq.merge()` 合并多个有序 iterable。
*   **`attrs` / `dataclasses`：** 使用这些库可以轻松定义简单类，并自动生成 `__init__`、`__repr__`、`__eq__` 等样板方法。
*   **NumPy：** 使用 NumPy 进行高效数组计算、逐元素操作、数学函数、过滤和聚合。
*   **Pandas：** 逐行构建 DataFrame 时，先追加到 list of dicts，再一次性调用 `pd.DataFrame()`，避免低效复制。中间行数据使用 `TypedDict` 或 `dataclasses`。
*   **Flags：** 使用 `argparse` 或 `click` 等库解析命令行 flag。以类型安全方式访问 flag 值。
*   **Serialization：** 对跨语言序列化，考虑 JSON（内置）、Protocol Buffers 或 msgpack。对带验证的 Python 序列化，使用 `pydantic` 进行运行时校验和自动序列化/反序列化，或使用 `cattrs` 对 `dataclasses`/`attrs` 做性能导向的序列化/反序列化。
*   **Regular Expressions：** 使用 `re.VERBOSE` 让复杂 regex 通过空白和注释更可读。选择正确方法（`re.search`、`re.fullmatch`）。简单字符串检查避免使用 regex（使用 `in`、`startswith`、`endswith`）。多次使用的 regex 用 `re.compile()` 编译。
*   **Caching：** 谨慎使用 `functools.lru_cache`。优先返回不可变类型。memoize method 时要小心，因为 instance 作为 cache key 可能导致内存泄漏；可考虑 `functools.cached_property`。
*   **Pickle：** 因安全风险和兼容性问题，避免使用 `pickle`。序列化优先使用 JSON、Protocol Buffers 或 msgpack。
*   **Multiprocessing：** 注意某些平台上的 `multiprocessing` 潜在问题，尤其是与 `fork` 相关的问题。对于 I/O-bound 任务，可考虑 threads（`concurrent.futures.ThreadPoolExecutor`）或 `asyncio` 等替代方案。
*   **Debugging：** 使用 `IPython.embed()` 或 `pdb.set_trace()` 进入交互式 shell 调试。可使用可视化 debugger。日志应包含上下文，包括输入和异常信息，使用 `logging.exception()` 或 `exc_info=True`。
*   **Property-Based Testing & Fuzzing：** 使用 `hypothesis` 进行自动生成测试用例的 property-based testing。对于 coverage-guided fuzzing，可考虑 `atheris` 或 `python-afl`。

### 测试

*   **Assertions：** 使用 pytest 原生 `assert` 语句和信息丰富的表达式。Pytest 会自动提供详细失败信息，展示相关值。当表达式本身不够清楚时，可使用 `assert condition, "helpful message"` 添加自定义消息。
*   **Custom Assertions：** 为重复的复杂检查编写可复用 helper function（不是 method）。使用 `pytest.fail("message")` 以自定义消息显式失败。
*   **Parameterized Tests：** 使用 `@pytest.mark.parametrize` 在不同输入上运行相同测试逻辑以减少重复。这比 `parameterized` 库更符合惯用法。
*   **Fixtures：** 使用 pytest fixtures（通过 `@pytest.fixture`）进行测试 setup、teardown 和依赖注入。Fixtures 比基于 class 的 setup method 更清晰，也更容易在测试间共享。
*   **Mocking：** 使用 `mock.create_autospec()` 并设置 `spec_set=True` 创建符合原对象接口的 mock，防止拼写错误和 API 不匹配。使用 context manager（`with mock.patch(...)`）管理 mock 生命周期并确保 patch 被停止。优先通过 fixture 注入依赖，而不是 patch。
*   **Asserting Mock Calls：** 断言 mock 调用时，使用 `mock.ANY` 和其他 matcher 进行部分参数匹配（例如 `assert_called_once_with`）。
*   **Temporary Files：** 使用 pytest 的 `tmp_path` 和 `tmp_path_factory` fixtures 创建隔离且自动清理的临时文件/目录。在 pytest 测试中优先于 `tempfile` 模块。
*   **Avoid Randomness：** 不要在单元测试中使用随机数生成输入。这会导致 flaky 且难以调试的测试。应使用确定性、易推理并覆盖具体行为的输入。
*   **Test Invariants：** 聚焦 public API 的不变行为，而不是实现细节。
*   **Test Organization：** 除非需要在 class 中共享多个测试方法的 fixture，否则优先使用简单测试函数，而不是基于 class 的测试。使用描述性测试名称说明正在测试的行为。

### 错误处理

*   **Re-raising Exceptions：** 使用裸 `raise` 重新抛出当前异常，以保留原始 stack trace。使用 `raise NewException from original_exception` 链接异常并提供上下文。使用 `raise NewException from None` 抑制原异常上下文。
*   **Exception Messages：** 抛出异常时始终包含描述性消息。
*   **Converting Exceptions to Strings：** `str(e)` 可能信息不足。`repr(e)` 通常更好。若需要包含 traceback 和 chained exception 的完整细节，使用 `traceback` 模块中的函数（例如 `traceback.format_exception(e)`、`traceback.format_exc()`）。
*   **Terminating Programs：** 对预期终止使用 `sys.exit()`。未捕获的非 `SystemExit` 异常应表示 bug。避免使用 `os.abort()` 这类会立即且不干净退出的函数。
*   **Returning None：** 保持一致。如果函数可能返回值，则所有路径都应该返回一个值（显式使用 `return None`）。裸 `return` 只用于概念上没有返回值的提前退出函数（注解为 `-> None`）。
