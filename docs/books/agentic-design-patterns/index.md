# Agentic 智能体设计模式

> 英文书名：Agentic Design Patterns -- A Hands-On Guide to Building Intelligent Systems
> 状态：在读
> 类型：智能体设计模式 / 工程实践

## 一句话总结

这本书围绕智能体系统的常见设计模式展开，重点不是单次提示词技巧，而是如何把模型、工具、状态、反馈和验证组织成可运行、可调试、可复盘的系统。

## 我为什么读

- 理解智能体系统中常见控制模式的边界。
- 把 Prompt Chain、Agent Loop、Workflow 等概念转成可执行的工程判断。
- 为后续设计和实现 Coding Agent、学习 Agent、研究 Agent 建立模式库。

## 阅读问题

- 什么任务适合 Prompt Chaining，什么任务更适合 Agent Loop？
- 程序和模型之间应该如何分配控制权？
- 如何用工作流、状态机和验证机制提高智能体系统可靠性？
- 每一种设计模式在真实工程中有哪些适用场景和失败模式？

## 当前结论

1. 智能体设计模式的核心是控制流、状态和验证，而不是让模型多调用几次。
2. Prompt Chaining 适合输入输出明确、阶段稳定的语义处理任务。
3. Agent Loop 适合需要根据环境反馈动态调整路径的开放任务。
4. 成熟系统通常会组合 Workflow、Agent Loop、Prompt Chain 和普通程序验证。

## 章节

- [第 1 章：提示链（Prompt Chain）](chapter-01-prompt-chaining.md)
- [第 2 章：路由（Routing）](chapter-02-routing.md)
- [第 3 章：并行化（Parallelization）](chapter-03-parallelization.md)

## 可执行行动

- [ ] 为每章提炼一个设计模式定义。
- [ ] 为每个模式补充适用场景、反例和工程检查清单。
- [ ] 将模式和 Pi / Codex / OpenHands 等真实 Coding Agent 实现对应起来。

## 保留怀疑

- 书中的模式分类是否覆盖真实智能体系统的复杂性？
- 不同框架对相同模式的命名是否会造成理解偏差？
- 哪些模式可以用确定性程序替代，而不应该交给 LLM？
