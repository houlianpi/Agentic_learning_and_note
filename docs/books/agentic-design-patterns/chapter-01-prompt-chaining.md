# 从 Prompt Chaining 到 Agent Loop：以 Pi Coding Agent 为例理解智能体工作流

在学习智能体设计模式时，Prompt Chaining 往往是最先接触的模式之一。它的概念看似简单：把多个提示词连接起来，让前一步的输出成为后一步的输入。

但如果只把它理解为“连续调用几次大模型”，就会遗漏这个模式真正重要的部分：

> Prompt Chaining 的核心不是调用次数，而是由程序明确规定任务阶段、数据流向和每一步的输入输出契约。

随着进一步研究 Coding Agent，还会遇到另一种常见结构：Agent Loop。以 Pi 为例，它并没有把“分析、计划、实现、验证”写成固定链条，而是让模型根据工具执行结果动态决定下一步。

这两种模式并不是相互替代的竞品。Prompt Chaining、Agent Loop 和 Workflow 分别解决不同层次的问题。成熟的 Coding Agent 系统通常会组合使用它们。

本文将从 Prompt Chaining 出发，结合 LangChain、LangGraph 和 Pi Agent 的代码，解释三种控制模式之间的关系。

## 一、Prompt Chaining 解决什么问题

假设我们希望让大模型完成一个代码任务：

> 为登录模块增加安全机制：用户连续登录失败 5 次后，将账户锁定 10 分钟。

最简单的做法是写一个很长的提示词：

```text
请分析需求，检查代码，制定实现计划，修改程序，运行测试，最后审查实现是否正确。
```

这种做法存在几个问题：

1. 模型可能还没有充分理解代码就开始修改；
2. 需求分析可能不完整；
3. 后面的实现可能偏离前面的分析；
4. 模型可能跳过测试或审查；
5. 所有过程混在一次调用里，程序无法检查中间结果；
6. 如果某一步出错，只能重新执行整个任务。

Prompt Chaining 会将任务拆成多个相对独立的阶段：

```text
原始需求
   ↓
需求分析
   ↓
实现计划
   ↓
代码修改
   ↓
测试验证
   ↓
代码审查
```

每一步都使用不同的 Prompt，并产生明确的输出。例如，第一步不是输出随意的分析文字，而是输出结构化需求：

```json
{
  "goal": "连续登录失败 5 次后锁定账户 10 分钟",
  "acceptanceCriteria": [
    "登录失败时增加失败次数",
    "第五次失败后锁定账户",
    "锁定期间拒绝登录",
    "十分钟后允许重新尝试",
    "成功登录后清空失败次数"
  ],
  "unknowns": [
    "失败次数保存在内存还是数据库",
    "是否允许管理员提前解除锁定"
  ]
}
```

第二步只负责根据这份需求制定计划。这样带来三个重要能力：中间结果可见、每一步可以独立验证、某一步失败时可以单独重试。

因此，Prompt Chaining 本质上是一种任务分解和控制流设计模式。

## 二、什么才是真正的 Prompt Chain

下面这段话只是一个复合提示词：

```text
请先分析需求，然后制定计划，然后实现，最后进行审查。
```

它不算严格意义上的 Prompt Chain，因为程序只调用了一次模型。模型可以跳过分析、偏离计划或在没有测试证据时声称任务完成。

真正的 Prompt Chain 应由程序控制步骤：

```python
analysis = analyze_chain.invoke({"request": user_request})

plan = plan_chain.invoke({
    "request": user_request,
    "analysis": analysis,
})

implementation = implementation_chain.invoke({
    "request": user_request,
    "analysis": analysis,
    "plan": plan,
})

review = review_chain.invoke({
    "request": user_request,
    "plan": plan,
    "implementation": implementation,
})
```

程序明确知道第一步是分析、第二步是计划、第三步是实现、第四步是审查，还可以在步骤之间加入 JSON Schema 等确定性验证。

这就是 Prompt Chaining 与“让模型自己按顺序做事”的根本区别：顺序由程序保证，而不是只依靠模型遵守自然语言指令。

## 三、LangChain 中的 Prompt Chaining

在 Python 生态中，一个基本 Chain 通常包含三个部分：

```text
Prompt Template
      ↓
Chat Model
      ↓
Output Parser
```

示例：

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4.1-mini", temperature=0)

analysis_prompt = ChatPromptTemplate.from_template(
    """
    分析下面的代码任务。

    用户需求：
    {request}

    请输出：
    1. 任务目标
    2. 验收条件
    3. 技术约束
    4. 尚不明确的问题
    """
)

analysis_chain = analysis_prompt | model | StrOutputParser()

analysis = analysis_chain.invoke({
    "request": "连续登录失败 5 次后锁定账户 10 分钟"
})
```

这里的 `|` 是 LangChain Expression Language（LCEL）的组合操作符。执行过程为：

```text
输入字典
  ↓
ChatPromptTemplate 格式化提示词
  ↓
ChatOpenAI 调用模型
  ↓
模型返回 AssistantMessage
  ↓
StrOutputParser 提取文本
  ↓
得到 Python 数据
```

## 四、LangChain 相关库分别负责什么

### `langchain-core`

提供底层通用抽象，包括 Prompt Template、Message、Runnable、Output Parser、LCEL 和工具协议。它可以理解为 LangChain 生态的基础协议层。

### `langchain`

提供更高层的 LLM 应用能力，包括 Chain、Agent、Retriever 和文档处理流程。不同版本之间的包结构会调整，当前很多底层类型会直接从 `langchain_core` 导入。

### `langchain-openai`

它是 LangChain 与 OpenAI API 之间的适配层，提供 `ChatOpenAI`、`OpenAIEmbeddings`、工具调用、流式响应和结构化输出。它是 Chain 中的模型节点，不负责整个工作流编排。

### `langchain-community`

主要提供社区维护的第三方集成，例如文档加载器、搜索引擎、向量数据库、第三方数据源和某些本地模型。它不是实现 Prompt Chaining 的必要组件。只有需要加载 Git 仓库、网页或数据库等外部数据时才可能用到。

### `langgraph`

LangGraph 用有向图和共享状态组织复杂工作流，适合条件分支、循环、重试、人工审批、多 Agent、状态持久化和中断恢复。

简单 Prompt Chain 通常是线性的：

```text
A → B → C
```

LangGraph 可以表达：

```text
分析需求
   ↓
制定计划
   ↓
审查计划
   ├── 通过 → 实现
   └── 不通过 → 返回制定计划
```

当 Prompt Chain 出现分支、循环和恢复需求时，它就开始演化为 Workflow 或状态图。

## 五、Prompt Chaining 的优势与局限

Prompt Chaining 适合输入输出明确、步骤相对稳定的任务。它能够降低单次推理负担、暴露中间结果、为不同步骤选择不同模型、独立重试失败步骤，并保留审计记录。

但它也存在明显局限：

- 前一步的错误会向后传播；
- 每增加一步都会增加 Token、费用、延迟和失败概率；
- 固定流程难以处理未知环境反馈；
- 自由文本不是可靠的数据契约；
- 把任务拆得过细会带来不必要的复杂度。

因此，重要的中间结果适合使用 JSON、Pydantic Model、TypedDict 或 JSON Schema 等结构化形式。

## 六、Pi Agent 使用的是 Agent Loop

Pi 是一个终端 Coding Agent。模型可以使用 `read`、`write`、`edit` 和 `bash` 等工具。它没有把“分析、计划、实现、验证”写成固定链条，而是采用动态 Agent Loop：

```text
用户消息
   ↓
调用模型
   ↓
模型输出普通文本或工具调用
   ├── 普通文本 → 准备结束
   └── 工具调用
          ↓
       Pi 执行工具
          ↓
       工具结果写回上下文
          ↓
       再次调用模型
```

核心实现位于 Pi 仓库的 [`packages/agent/src/agent-loop.ts`](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)。简化后可以表示为：

```typescript
while (shouldContinue) {
    const assistantMessage = await callModel(context);
    const toolCalls = extractToolCalls(assistantMessage);

    if (toolCalls.length === 0) {
        break;
    }

    const toolResults = await executeTools(toolCalls);
    context.messages.push(...toolResults);
}
```

关键区别在于：Prompt Chain 的下一步由程序预先决定；Pi Agent Loop 的下一步由模型根据当前上下文和工具结果动态决定。

## 七、理解 Pi Agent Loop：Message、Turn 和 Run

### Message

Message 是一条对话消息，可以是用户消息、AssistantMessage 或 ToolResultMessage。

### Turn

Turn 是一次模型响应，以及该响应产生的全部工具调用和工具结果：

```text
Turn 1
├── AssistantMessage：调用 read
└── ToolResultMessage：返回文件内容
```

### Run

从用户提交请求到 Agent 最终结束，是一次完整 Run：

```text
Run
├── 用户请求
├── Turn 1：读取文件
├── Turn 2：搜索代码
├── Turn 3：修改代码
├── Turn 4：运行测试
└── Turn 5：输出最终回答
```

一个 Run 包含多个 Turn；一个 Turn 包含一个 AssistantMessage 和零个或多个 ToolResultMessage。

## 八、Pi 的双层循环

Pi 的主循环可以简化为：

```typescript
while (true) {
    let hasMoreToolCalls = true;

    while (hasMoreToolCalls || pendingMessages.length > 0) {
        const message = await streamAssistantResponse(...);
        const toolCalls = extractToolCalls(message);

        if (toolCalls.length > 0) {
            const toolResults = await executeToolCalls(...);
            context.messages.push(...toolResults);
            hasMoreToolCalls = true;
        } else {
            hasMoreToolCalls = false;
        }

        pendingMessages = await getSteeringMessages();
    }

    const followUps = await getFollowUpMessages();

    if (followUps.length > 0) {
        pendingMessages = followUps;
        continue;
    }

    break;
}
```

内层循环处理当前 Agent 的“模型生成—工具执行—结果写回”过程。外层循环负责处理 Agent 原本准备结束后的 follow-up 消息。

## 九、工具结果为什么必须写回上下文

假设模型调用：

```json
{
  "type": "toolCall",
  "id": "call_1",
  "name": "read",
  "arguments": { "path": "src/auth.ts" }
}
```

Pi 执行后生成：

```json
{
  "role": "toolResult",
  "toolCallId": "call_1",
  "toolName": "read",
  "content": [
    {
      "type": "text",
      "text": "export function login() { ... }"
    }
  ],
  "isError": false
}
```

下一次模型调用能够看到自己执行了什么动作、动作是否成功、环境返回了什么，以及下一步应该做什么。这形成反馈闭环：

```text
模型推断
  ↓
模型提出动作
  ↓
程序执行动作
  ↓
环境返回观察结果
  ↓
模型重新推断
```

它与经典的 `Reason → Act → Observe → Reason` 结构一致。

## 十、Pi 如何处理工具调用

Pi 不会无条件执行模型生成的工具调用。完整管线为：

```text
模型生成 ToolCall
  ↓
查找工具
  ↓
转换兼容参数
  ↓
验证参数 Schema
  ↓
beforeToolCall
  ↓
执行工具
  ↓
afterToolCall
  ↓
生成 ToolResultMessage
  ↓
写入上下文
```

如果工具不存在或参数无效，Pi 会生成错误 ToolResult，让模型尝试恢复，而不是直接让 Agent 崩溃。`beforeToolCall` 可以进行权限检查、用户审批和危险命令拦截；`afterToolCall` 可以清理敏感数据、截断过长结果或改变终止状态。

因此，模型只是提出动作，程序仍然掌握验证、执行和停止的控制权。

## 十一、串行、并行与安全终止

多个互不依赖的读取或搜索可以并行执行；修改后读取、修改后测试等存在依赖的操作必须串行。即使并行执行，Pi 也会按模型原始调用顺序把工具结果写入上下文，避免执行速度导致随机消息顺序。

可靠的 Agent Loop 还必须处理终止和异常：

- 模型没有调用工具时准备结束；
- 模型请求失败或取消时结束 Run；
- 工具失败时把错误写回上下文，让模型恢复；
- 工具调用因输出长度被截断时拒绝执行；
- 宿主程序可以根据 Turn 数、费用、上下文、安全策略或用户请求主动停止。

这说明生产级 Agent 不是“模型想做什么就做什么”，而是模型提出动作、程序验证并执行、环境产生结果、模型根据结果继续判断。

## 十二、Steering 与 Follow-up

Steering 用于调整当前任务，例如：

```text
先不要修改测试文件。
```

它会在当前 Turn 的工具执行完成后、下一次模型调用前加入上下文，保证 AssistantMessage、ToolCall 和 ToolResult 组成完整 Turn。

Follow-up 是等待当前任务结束后再处理的消息，例如：

```text
修复完成后，再帮我更新 CHANGELOG。
```

因此：

```text
Steering：影响当前任务的后续行为
Follow-up：当前任务完成后再继续
```

Pi 的内层循环处理 steering，外层循环处理 follow-up。

## 十三、为什么 Pi Agent Loop 不是 Prompt Chain

Pi 也会多次调用模型，但多次调用并不等于 Prompt Chaining。

Prompt Chaining 由程序预先规定每一步语义：

```typescript
const analysis = await analyze(input);
const plan = await createPlan(analysis);
const patch = await implement(plan);
const review = await review(patch);
```

Pi Agent Loop 只规定执行协议：

```typescript
while (shouldContinue) {
    const response = await model(context);
    const calls = extractToolCalls(response);
    const results = await execute(calls);
    context.messages.push(...results);
}
```

程序事先不知道模型会选择：

```text
read → search → edit → bash → edit → bash
```

还是直接输出回答。

最准确的区别是：Prompt Chain 由程序控制任务的语义步骤；Agent Loop 由程序控制执行协议和安全边界，模型根据环境反馈控制任务的语义路径。

## 十四、当前开源 Coding Agent 的主流方案

从运行内核来看，Agent Loop 是当前开源 Coding Agent 的主流方案，而不是纯线性 Prompt Chaining。Pi、OpenAI Codex、OpenHands、Cline、SWE-agent 和 Goose 等项目虽然名称和实现不同，但都围绕类似结构构建：

```text
LLM
  ↓ Action / Tool Call
执行环境
  ↓ Observation / Tool Result
LLM
```

原因是代码任务具有较强不确定性。程序通常无法提前知道问题在哪个文件、需要搜索什么、第一次修改是否有效，以及测试失败后应该采取什么动作。

但 Prompt Chaining 并没有被淘汰。它仍适合需求提取、上下文整理、代码审查、测试报告和 PR 描述生成等输入输出明确的局部任务。

## 十五、更成熟的方案：Workflow 包住 Agent Loop

成熟的 Coding Agent 通常采用混合架构：

```text
┌─────────────────────────────────────┐
│ 外层 Workflow / 状态机               │
│                                     │
│ 分析 → 计划 → 实现 → 验证 → 审查      │
└─────────────────────────────────────┘
        │                    │
        ▼                    ▼
┌────────────────┐   ┌────────────────┐
│ Agent Loop     │   │ Agent Loop     │
│ read           │   │ read diff      │
│ search         │   │ run tests      │
│ inspect        │   │ inspect errors │
│ reason         │   │ review         │
└────────────────┘   └────────────────┘
```

外层 Workflow 保证分析、计划、实现、验证和审查等关键阶段不被跳过，并负责持久化、失败重试、人工审批和中断恢复。每个开放式阶段内部使用 Agent Loop，根据环境反馈动态工作。输入输出明确的子任务则继续使用 Prompt Chain。

三种控制方式分别负责不同层次：

| 层次 | 合适方案 |
|---|---|
| 完整任务生命周期 | Workflow、状态机或 LangGraph |
| 开放式代码探索和修改 | Agent Loop |
| 输入输出明确的局部处理 | Prompt Chaining |
| 确定性验证 | 普通程序代码 |

## 十六、选择模式的实用方法

可以使用下面的判断顺序：

```text
能否用确定性程序完成？
→ 使用普通代码

步骤能否提前确定，且输入输出明确？
→ 使用 Prompt Chain

下一步是否依赖未知环境反馈？
→ 使用 Agent Loop

是否需要长期状态、分支、恢复或人工审批？
→ 使用 Workflow、状态机或 LangGraph
```

例如，文件是否存在、命令退出码、测试是否通过、JSON 是否满足 Schema 等，都应该由普通程序验证，而不是询问 LLM。

## 十七、容易出现的误解

### 多次调用模型就是 Prompt Chaining

不一定。只有程序明确控制步骤，并将前一步输出作为后一步输入时，才属于严格意义上的 Prompt Chaining。

### 一个 Prompt 中写“先分析再实现”就是 Prompt Chain

不是。它只是要求模型在一次调用中遵守逻辑顺序，程序无法确认模型是否真的完成了每个阶段。

### Agent Loop 比 Prompt Chain 更先进

二者解决的问题不同。Prompt Chain 强调可预测的数据流，Agent Loop 强调根据环境反馈动态调整，Workflow 强调完整任务生命周期管理。

### Agent Loop 应该把所有事情交给模型

不正确。能用确定性代码完成的判断不应交给概率性的 LLM。模型负责语义和未知情况，程序负责状态、权限、验证和执行边界。

### 模型调用工具意味着模型拥有系统控制权

不正确。模型只是提出工具调用，程序仍负责查找工具、验证参数、检查权限、决定是否执行、控制并发、过滤结果和判断是否继续。

## 十八、最终理解

Prompt Chaining 是一种任务分解模式。它通过程序预先规定多个 LLM 步骤，让前一步的结构化输出成为后一步的输入，适合流程稳定、输入输出明确、需要检查中间结果的任务。

Agent Loop 是一种动态执行模式。模型根据当前上下文提出工具调用，程序验证并执行工具，然后把环境结果返回模型。下一步行为取决于真实反馈，因此适合代码探索、修改、调试和测试等路径不确定的任务。

Workflow 或状态机负责更高层的任务生命周期。它可以保证关键阶段不会被跳过，同时提供分支、重试、人工审批、状态持久化和中断恢复。

三者之间的关系可以概括为：

> Workflow 决定任务经过哪些阶段，Agent Loop 决定一个开放阶段中下一步做什么，Prompt Chaining 负责阶段内输入输出明确的连续语义处理。

对于完整的 Coding Agent，推荐的架构是：

```text
确定性 Workflow
    +
动态 Agent Loop
    +
局部 Prompt Chain
    +
普通程序验证
```

其中，普通程序提供确定性，Prompt Chain 提供结构化分解，Agent Loop 提供环境适应能力，Workflow 提供全局控制和可恢复性。

一个可靠的智能体系统，不是简单地“让模型多思考几次”，而是合理分配模型与程序之间的控制权：模型负责处理语义和未知情况，程序负责状态、验证、权限和执行边界。
