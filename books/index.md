# ADK 教科书目录

说明：本目录严格以 `src/google/adk` 下的顶层模块为章节来源，但章节顺序不按字母排序，而按“先主干、后机制、再外围”的学习路径安排。这样更适合从架构理解走到实现细节。

## 第一部分：核心执行主干

第一章 [`agents`](./part-1-core-execution/01-agents.md)：Agent 抽象与执行主体

第二章 [`runners.py`](./part-1-core-execution/02-runners.md)：请求如何被调度起来

第三章 [`sessions`](./part-1-core-execution/03-sessions.md)：会话、状态与历史持久化

第四章 [`events`](./part-1-core-execution/04-events.md)：统一事件协议与系统事实流

第五章 [`workflow`](./part-1-core-execution/05-workflow.md)：节点图、编排与复杂执行流

## 第二部分：LLM 运行时机制

第六章 [`flows`](./part-2-llm-runtime/06-flows.md)：LLM 调用编排流水线

第七章 [`models`](./part-2-llm-runtime/07-models.md)：模型抽象、请求与响应协议

第八章 [`tools`](./part-2-llm-runtime/08-tools.md)：工具系统与函数调用能力

第九章 [`memory`](./part-2-llm-runtime/09-memory.md)：长期记忆与会话外知识沉淀

第十章 [`artifacts`](./part-2-llm-runtime/10-artifacts.md)：文件与产物管理

第十一章 [`apps`](./part-2-llm-runtime/11-apps.md)：应用容器与运行配置聚合

## 第三部分：横切能力与系统控制面

第十二章 [`plugins`](./part-3-cross-cutting-control/12-plugins.md)：插件机制与横切扩展

第十三章 [`auth`](./part-3-cross-cutting-control/13-auth.md)：认证、授权与凭据交换

第十四章 [`telemetry`](./part-3-cross-cutting-control/14-telemetry.md)：追踪、监控与可观测性

第十五章 [`errors`](./part-3-cross-cutting-control/15-errors.md)：错误模型与异常边界

第十六章 [`features`](./part-3-cross-cutting-control/16-features.md)：特性开关与兼容控制

第十七章 [`optimization`](./part-3-cross-cutting-control/17-optimization.md)：性能与成本优化机制

第十八章 [`planners`](./part-3-cross-cutting-control/18-planners.md)：规划能力与任务分解

## 第四部分：基础设施与运行环境

第十九章 [`platform`](./part-4-infrastructure-environment/19-platform.md)：平台级基础能力抽象

第二十章 [`environment`](./part-4-infrastructure-environment/20-environment.md)：环境探测与运行环境适配

第二十一章 [`dependencies`](./part-4-infrastructure-environment/21-dependencies.md)：依赖管理与可选能力加载

第二十二章 [`code_executors`](./part-4-infrastructure-environment/22-code-executors.md)：代码执行环境与沙箱能力

第二十三章 [`utils`](./part-4-infrastructure-environment/23-utils.md)：通用工具函数与底层辅助设施

## 第五部分：生态集成与分布式协作

第二十四章 [`integrations`](./part-5-ecosystem-distributed/24-integrations.md)：外部系统集成层

第二十五章 [`a2a`](./part-5-ecosystem-distributed/25-a2a.md)：Agent-to-Agent 协作协议

第二十六章 [`skills`](./part-5-ecosystem-distributed/26-skills.md)：技能封装与能力复用

## 第六部分：工程化验证与开发入口

第二十七章 [`evaluation`](./part-6-engineering-entry/27-evaluation.md)：评估体系与效果验证

第二十八章 [`cli`](./part-6-engineering-entry/28-cli.md)：命令行、服务入口与开发调试

第二十九章 [`examples`](./part-6-engineering-entry/29-examples.md)：示例工程与最佳实践入口

## 第七部分：实验区与前沿探索

第三十章 [`labs`](./part-7-experimental-frontier/30-labs.md)：实验性能力与未稳定特性
