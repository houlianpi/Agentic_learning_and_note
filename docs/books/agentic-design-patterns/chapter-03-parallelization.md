# 从 Chain 到 Parallelization：理解 Agent 里的并行化设计

《Agentic Design Patterns》第三章讲的是 Parallelization，也就是并行化。书里的定义并不难理解：

> 当多个子任务互不依赖时，可以同时执行它们，再把结果汇总。

但如果只看概念和一个 LangChain 示例，很容易把并行化理解成：

```text
把几个 LLM 调用放进 Promise.all / asyncio.gather / RunnableParallel
```

这个理解不算错，但太浅。真实 Agent 系统里的并行化，真正难的不是“怎么同时跑”，而是：

```text
哪些任务可以同时跑？
并行任务之间有没有共享状态？
并发数量怎么控制？
部分失败怎么办？
结果顺序怎么保持稳定？
输出太多怎么聚合？
高风险动作能不能并行？
用户取消时如何清理？
```

以 Pi Coding Agent 为例，第三章的模式可以在两个地方看到真实实现：

```text
1. Agent Loop 中的并行工具执行
2. subagent 扩展中的并行子智能体委托
```

前者是 Pi runtime 的核心执行能力，后者更接近第三章“多个子智能体并行研究后汇总”的例子。

## 一、Parallelization 和前两章的关系

前三章可以放在一条控制流主线上理解：

```text
Prompt Chaining:
一个步骤完成后，结果进入下一个步骤。

Routing:
在多个后续路径中选择一条。

Parallelization:
在同一阶段中，同时执行多个互不依赖的步骤。
```

对应到流程形状：

```text
Prompt Chaining:
A -> B -> C

Routing:
A -> B1 / B2 / B3

Parallelization:
A -> [B1, B2, B3 同时执行] -> C
```

所以并行化不是替代 Chain 或 Routing，而是补充它们。

真实 Agent 往往是混合结构：

```text
用户任务
   ↓
先判断任务类型
   ↓
并行收集证据
   ↓
顺序聚合结论
   ↓
再决定下一步
```

也就是：

```text
chain controls phases
parallel accelerates independent work inside a phase
```

## 二、什么时候适合 Chain，什么时候适合 Parallel

判断标准很直接：

```text
后一步需要前一步结果 -> Chain
多个任务只依赖同一个原始输入 -> Parallel
先并行收集，再顺序综合 -> Parallel + Chain
```

适合 Chain 的例子：

```text
1. 读取配置文件
2. 基于配置文件判断问题
3. 基于判断结果修改代码
4. 基于修改结果运行测试
5. 基于测试结果总结
```

后一步离不开前一步输出，所以必须顺序执行。

适合 Parallel 的例子：

```text
1. 子任务 A 查实现代码
2. 子任务 B 查测试覆盖
3. 子任务 C 查历史文档
```

这三个任务都只依赖原始问题，不依赖彼此结果，可以并行。

最常见的是混合模式：

```text
Chain:
  先理解用户问题

Parallel:
  同时查实现、测试、文档

Chain:
  汇总发现 -> 定位原因 -> 修改代码 -> 运行验证
```

## 三、Pi Agent Loop 里的并行工具执行

Pi 中最核心的并行化实现，在 `packages/agent/src/agent-loop.ts`。

当模型返回一条 assistant message 后，Pi 会先找出里面所有工具调用：

```ts
const toolCalls = message.content.filter((c) => c.type === "toolCall");
```

也就是说，模型一次响应里可能包含多个工具调用：

```text
toolCall: read file A
toolCall: read file B
toolCall: grep keyword C
```

如果这些工具调用互不依赖，就可以并行执行。

Pi 的分派逻辑是：

```ts
const hasSequentialToolCall = toolCalls.some(
  (tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
);

if (config.toolExecution === "sequential" || hasSequentialToolCall) {
  return executeToolCallsSequential(currentContext, assistantMessage, toolCalls, config, signal, emit);
}

return executeToolCallsParallel(currentContext, assistantMessage, toolCalls, config, signal, emit);
```

这段代码表达了两个规则：

```text
默认可以并行执行工具调用
但只要配置要求顺序执行，或某个工具声明必须顺序执行，就退回顺序模式
```

Pi 的默认工具执行模式也是并行：

```ts
this.toolExecution = runtimeOptions.toolExecution ?? "parallel";
```

这说明并行工具执行不是一个边缘优化，而是 Agent Loop 的默认能力。

## 四、Pi 如何并行执行工具

并行执行函数是 `executeToolCallsParallel`。

核心结构是：

```ts
async function executeToolCallsParallel(...) {
  const finalizedCalls: FinalizedToolCallEntry[] = [];

  for (const toolCall of toolCalls) {
    await emit({
      type: "tool_execution_start",
      toolCallId: toolCall.id,
      toolName: toolCall.name,
      args: toolCall.arguments,
    });

    const preparation = await prepareToolCall(currentContext, assistantMessage, toolCall, config, signal);

    if (preparation.kind === "immediate") {
      ...
      finalizedCalls.push(finalized);
      continue;
    }

    finalizedCalls.push(async () => {
      const executed = await executePreparedToolCall(preparation, signal, emit);
      const finalized = await finalizeExecutedToolCall(...);
      await emitToolExecutionEnd(finalized, emit);
      return finalized;
    });
  }

  const orderedFinalizedCalls = await Promise.all(
    finalizedCalls.map((entry) => (typeof entry === "function" ? entry() : Promise.resolve(entry))),
  );

  ...
}
```

这个实现有一个非常重要的细节：

```text
准备阶段是顺序做的。
真正执行阶段才并行。
```

准备阶段包括：

```text
找到工具
准备参数
校验参数
执行 beforeToolCall hook
判断是否被阻止
```

这些步骤顺序执行，可以保证每个工具调用先经过统一校验和拦截。

真正耗时的部分是：

```ts
const executed = await executePreparedToolCall(preparation, signal, emit);
```

这一步会调用工具自己的 `execute`：

```ts
const result = await prepared.tool.execute(
  prepared.toolCall.id,
  prepared.args as never,
  signal,
  ...
);
```

因此 Pi 的并行化不是“盲目同时跑”，而是：

```text
安全检查顺序做
独立 I/O 工作并行做
结果汇聚后再进入下一轮
```

这比一个简单的 `Promise.all` 更接近真实工程。

## 五、为什么结果顺序要稳定

并行执行会带来一个问题：完成顺序是不稳定的。

例如模型发出两个工具调用：

```text
tool-1: 慢任务
tool-2: 快任务
```

实际完成顺序可能是：

```text
tool-2 先完成
tool-1 后完成
```

Pi 的设计是：

```text
tool_execution_end 按真实完成顺序发事件
toolResult message 按模型原始 tool call 顺序写回上下文
```

测试里明确验证了这一点：

```ts
expect(toolExecutionEndIds).toEqual(["tool-2", "tool-1"]);
expect(toolResultIds).toEqual(["tool-1", "tool-2"]);
expect(turnToolResultIds).toEqual(["tool-1", "tool-2"]);
```

这很关键。

UI 需要真实进度：

```text
哪个工具先完成，就先展示哪个工具完成。
```

但模型下一轮需要稳定上下文：

```text
每次都按原始 tool call 顺序看到结果。
```

否则同一批工具调用，今天上下文是 A、B，明天上下文是 B、A，模型行为会漂。

并行 Agent 里，稳定的汇总顺序是正确性的一部分。

## 六、什么时候不能并行：executionMode

Pi 的工具可以声明执行模式：

```ts
executionMode?: ToolExecutionMode;
```

类型定义说明：

```text
"sequential": this tool must execute one at a time with other tool calls.
"parallel": this tool can execute concurrently with other tool calls.
```

如果某个工具有共享状态、顺序依赖或副作用，就应该声明：

```ts
executionMode: "sequential"
```

Pi 的 tic-tac-toe 扩展示例很好地说明了这个问题。

这个工具让 Agent 下棋。Agent 要移动光标再落子：

```text
move_down
move_right
play
```

如果这几个工具调用并行执行，就可能出现：

```text
play 比 move_down / move_right 先完成
棋子落在错误位置
```

所以该工具强制顺序执行：

```ts
executionMode: "sequential" as ToolExecutionMode,
```

这说明并行化的边界是：

```text
无共享状态、无顺序依赖、可独立失败的任务 -> 可以并行
共享状态、顺序敏感、有副作用的任务 -> 不要并行
```

## 七、subagent 扩展：更接近第三章的并行子智能体

除了核心 Agent Loop，Pi 的示例扩展里还有一个更贴近第三章的实现：

```text
packages/coding-agent/examples/extensions/subagent
```

文件开头说明：

```ts
/**
 * Subagent Tool - Delegate tasks to specialized agents
 *
 * Spawns a separate `pi` process for each subagent invocation,
 * giving it an isolated context window.
 *
 * Supports three modes:
 *   - Single: { agent: "name", task: "..." }
 *   - Parallel: { tasks: [{ agent: "name", task: "..." }, ...] }
 *   - Chain: { chain: [{ agent: "name", task: "... {previous} ..." }, ...] }
 *
 * Uses JSON mode to capture structured output from subagents.
 */
```

这里的设计非常清楚：

```text
single   = 调一个子 agent
parallel = 多个子 agent 并行跑
chain    = 多个子 agent 顺序跑，后一步可以引用前一步输出
```

这正好对应：

```text
Prompt Chaining -> chain 模式
Parallelization -> parallel 模式
```

## 八、子 Agent 如何定义

子 Agent 不是写死在代码里的类，而是 markdown 配置文件。

加载逻辑在 `agents.ts`：

```ts
const { frontmatter, body } = parseFrontmatter<Record<string, string>>(content);

if (!frontmatter.name || !frontmatter.description) {
  continue;
}

const tools = frontmatter.tools
  ?.split(",")
  .map((t: string) => t.trim())
  .filter(Boolean);

agents.push({
  name: frontmatter.name,
  description: frontmatter.description,
  tools: tools && tools.length > 0 ? tools : undefined,
  model: frontmatter.model,
  systemPrompt: body,
  source,
  filePath,
});
```

一个子 Agent 文件可以长这样：

```md
---
name: researcher
description: Research code and summarize findings
tools: read,grep,find
model: gpt-5
---

你是研究型子 Agent，只负责查找信息，不修改代码。
```

`frontmatter` 提供配置：

```text
name
description
tools
model
```

正文作为这个子 Agent 的 system prompt。

支持两个来源：

```text
user    -> ~/.pi/agent/agents
project -> 当前项目最近的 .pi/agents
```

这让 subagent 可以做到角色隔离：

```text
研究 Agent 只能读和搜索
审查 Agent 只负责找风险
写作 Agent 只负责整理输出
```

## 九、subagent 工具如何暴露给主 Agent

扩展通过 `pi.registerTool` 注册一个工具：

```ts
pi.registerTool({
  name: "subagent",
  label: "Subagent",
  description: [
    "Delegate tasks to specialized subagents with isolated context.",
    "Modes: single (agent + task), parallel (tasks array), chain (sequential with {previous} placeholder).",
    ...
  ].join(" "),
  parameters: SubagentParams,

  async execute(_toolCallId, params, signal, onUpdate, ctx) {
    ...
  },
});
```

主 Agent 看到的是一个普通工具：

```text
subagent
```

但这个工具内部会再启动新的 Pi 进程。

参数 schema 支持三种模式：

```ts
const SubagentParams = Type.Object({
  agent: Type.Optional(Type.String(...)),
  task: Type.Optional(Type.String(...)),
  tasks: Type.Optional(Type.Array(TaskItem, ...)),
  chain: Type.Optional(Type.Array(ChainItem, ...)),
  agentScope: Type.Optional(AgentScopeSchema),
  confirmProjectAgents: Type.Optional(...),
  cwd: Type.Optional(Type.String(...)),
});
```

并行调用形态类似：

```json
{
  "tasks": [
    {
      "agent": "researcher",
      "task": "Find where parallel tool execution is implemented."
    },
    {
      "agent": "reviewer",
      "task": "Review risks of parallel tool execution."
    }
  ],
  "agentScope": "both"
}
```

## 十、为什么三种模式只能选一种

执行函数一开始会检查：

```ts
const hasChain = (params.chain?.length ?? 0) > 0;
const hasTasks = (params.tasks?.length ?? 0) > 0;
const hasSingle = Boolean(params.agent && params.task);
const modeCount = Number(hasChain) + Number(hasTasks) + Number(hasSingle);

if (modeCount !== 1) {
  return {
    content: [
      {
        type: "text",
        text: `Invalid parameters. Provide exactly one mode.\nAvailable agents: ${available}`,
      },
    ],
    details: makeDetails("single")([]),
  };
}
```

这是一个很重要的约束。

不能同时传：

```text
tasks
chain
agent + task
```

因为它们的执行语义不同：

```text
tasks       = 互不依赖，可以并行
chain       = 前后依赖，必须顺序
agent+task  = 单个子任务
```

这也是通用 Agent 开发中的关键原则：

> LLM 可以选择执行结构，但 runtime 必须用 schema 和校验保证语义明确。

## 十一、单个子 Agent 如何运行

核心函数是 `runSingleAgent`。

它先根据 agent 配置构造命令参数：

```ts
const args: string[] = ["--mode", "json", "-p", "--no-session"];
if (agent.model) args.push("--model", agent.model);
if (agent.tools && agent.tools.length > 0) args.push("--tools", agent.tools.join(","));
```

这些参数的含义是：

```text
--mode json   子进程输出 JSON 事件，方便父进程解析
-p            prompt 模式
--no-session  子 Agent 不写入当前主会话
--model       可为子 Agent 指定模型
--tools       可限制子 Agent 可用工具
```

如果子 Agent 有自己的 system prompt，会写入临时文件：

```ts
if (agent.systemPrompt.trim()) {
  const tmp = await writePromptToTempFile(agent.name, agent.systemPrompt);
  tmpPromptDir = tmp.dir;
  tmpPromptPath = tmp.filePath;
  args.push("--append-system-prompt", tmpPromptPath);
}
```

然后启动一个新的 Pi 进程：

```ts
const proc = spawn(invocation.command, invocation.args, {
  cwd: cwd ?? defaultCwd,
  shell: false,
  stdio: ["ignore", "pipe", "pipe"],
});
```

这就是“隔离上下文”的来源：

```text
每个子 Agent 是一个独立 Pi 进程
有自己的 prompt
有自己的工具限制
有自己的上下文窗口
有自己的输出事件流
```

## 十二、父进程如何收集子 Agent 输出

子 Agent 用 JSON mode 输出事件。父进程逐行解析 stdout：

```ts
proc.stdout.on("data", (data) => {
  buffer += data.toString();
  const lines = buffer.split("\n");
  buffer = lines.pop() || "";
  for (const line of lines) processLine(line);
});
```

`processLine` 会解析 JSON 事件：

```ts
if (event.type === "message_end" && event.message) {
  const msg = event.message as Message;
  currentResult.messages.push(msg);

  if (msg.role === "assistant") {
    currentResult.usage.turns++;
    const usage = msg.usage;
    if (usage) {
      currentResult.usage.input += usage.input || 0;
      currentResult.usage.output += usage.output || 0;
      currentResult.usage.cacheRead += usage.cacheRead || 0;
      currentResult.usage.cacheWrite += usage.cacheWrite || 0;
      currentResult.usage.cost += usage.cost?.total || 0;
      currentResult.usage.contextTokens = usage.totalTokens || 0;
    }
    if (!currentResult.model && msg.model) currentResult.model = msg.model;
    if (msg.stopReason) currentResult.stopReason = msg.stopReason;
    if (msg.errorMessage) currentResult.errorMessage = msg.errorMessage;
  }

  emitUpdate();
}
```

所以每个子 Agent 的结果不是一段普通字符串，而是结构化结果：

```text
messages
stderr
exitCode
usage
model
stopReason
errorMessage
```

这对 UI 展示、失败诊断、成本统计和最终汇总都很重要。

## 十三、Chain 模式：顺序依赖

`chain` 模式逻辑如下：

```ts
const results: SingleResult[] = [];
let previousOutput = "";

for (let i = 0; i < params.chain.length; i++) {
  const step = params.chain[i];
  const taskWithContext = step.task.replace(/\{previous\}/g, previousOutput);

  const result = await runSingleAgent(...);
  results.push(result);

  const isError = isFailedResult(result);
  if (isError) {
    return {
      content: [{ type: "text", text: `Chain stopped at step ${i + 1} (${step.agent}): ${errorMsg}` }],
      details: makeDetails("chain")(results),
      isError: true,
    };
  }

  previousOutput = getFinalOutput(result.messages);
}
```

这里体现了标准 Prompt Chaining：

```text
step 1 输出 -> previousOutput
step 2 的 task 里使用 {previous}
step 2 输出 -> previousOutput
step 3 继续
```

例如：

```json
{
  "chain": [
    {
      "agent": "researcher",
      "task": "Find the relevant files for parallel execution."
    },
    {
      "agent": "writer",
      "task": "Using this research, write an explanation: {previous}"
    }
  ]
}
```

第二步依赖第一步输出，所以不能并行。

## 十四、Parallel 模式：并发执行独立子任务

`parallel` 模式从 `tasks` 参数进入。

先做任务数量限制：

```ts
if (params.tasks.length > MAX_PARALLEL_TASKS)
  return {
    content: [
      {
        type: "text",
        text: `Too many parallel tasks (${params.tasks.length}). Max is ${MAX_PARALLEL_TASKS}.`,
      },
    ],
    details: makeDetails("parallel")([]),
  };
```

限制值定义在文件顶部：

```ts
const MAX_PARALLEL_TASKS = 8;
const MAX_CONCURRENCY = 4;
const PER_TASK_OUTPUT_CAP = 50 * 1024;
```

这说明真实并行化必须有资源边界：

```text
最多提交多少任务
最多同时跑多少任务
每个任务最多展示多少输出
```

然后初始化占位结果：

```ts
const allResults: SingleResult[] = new Array(params.tasks.length);

for (let i = 0; i < params.tasks.length; i++) {
  allResults[i] = {
    agent: params.tasks[i].agent,
    agentSource: "unknown",
    task: params.tasks[i].task,
    exitCode: -1, // -1 = still running
    messages: [],
    stderr: "",
    usage: ...
  };
}
```

`exitCode = -1` 表示还在运行。这样 UI 能展示进度。

并行执行发生在这里：

```ts
const results = await mapWithConcurrencyLimit(params.tasks, MAX_CONCURRENCY, async (t, index) => {
  const result = await runSingleAgent(
    ctx.cwd,
    agents,
    t.agent,
    t.task,
    t.cwd,
    undefined,
    signal,
    (partial) => {
      if (partial.details?.results[0]) {
        allResults[index] = partial.details.results[0];
        emitParallelUpdate();
      }
    },
    makeDetails("parallel"),
  );

  allResults[index] = result;
  emitParallelUpdate();
  return result;
});
```

这就是第三章的核心：

```text
多个独立任务同时启动
每个任务使用自己的子 Agent
父 Agent 等待它们全部完成
最后聚合结果
```

## 十五、并发限制：不是所有任务一起跑

`mapWithConcurrencyLimit` 是一个小型 worker pool：

```ts
async function mapWithConcurrencyLimit<TIn, TOut>(
  items: TIn[],
  concurrency: number,
  fn: (item: TIn, index: number) => Promise<TOut>,
): Promise<TOut[]> {
  if (items.length === 0) return [];
  const limit = Math.max(1, Math.min(concurrency, items.length));
  const results: TOut[] = new Array(items.length);
  let nextIndex = 0;

  const workers = new Array(limit).fill(null).map(async () => {
    while (true) {
      const current = nextIndex++;
      if (current >= items.length) return;
      results[current] = await fn(items[current], current);
    }
  });

  await Promise.all(workers);
  return results;
}
```

如果有 8 个任务，`MAX_CONCURRENCY = 4`，执行形态类似：

```text
worker 1 -> task 0 -> task 4
worker 2 -> task 1 -> task 5
worker 3 -> task 2 -> task 6
worker 4 -> task 3 -> task 7
```

这比直接：

```ts
await Promise.all(tasks.map(runSingleAgent));
```

更安全。

原因是：

```text
不会一次启动太多 Pi 进程
不会一次打爆模型 API
不会让本机 I/O 和 CPU 失控
失败重试不会瞬间放大流量
成本更可控
```

真实 Agent 并行化必须有并发上限。

## 十六、并行进度如何反馈给用户

并行任务可能运行很久。用户不能只看到一个 spinner。

`subagent` 里有进度更新：

```ts
const emitParallelUpdate = () => {
  if (onUpdate) {
    const running = allResults.filter((r) => r.exitCode === -1).length;
    const done = allResults.filter((r) => r.exitCode !== -1).length;
    onUpdate({
      content: [
        { type: "text", text: `Parallel: ${done}/${allResults.length} done, ${running} running...` },
      ],
      details: makeDetails("parallel")([...allResults]),
    });
  }
};
```

用户看到的就是：

```text
Parallel: 2/5 done, 3 running...
```

这属于并行 Agent 的可观测性设计。

如果没有这种反馈，并行系统会变成黑盒：

```text
不知道谁在跑
不知道谁完成了
不知道谁失败了
不知道是否应该取消
```

## 十七、并行结果如何汇总

所有任务结束后，`subagent` 会统计成功数并生成摘要：

```ts
const successCount = results.filter((r) => !isFailedResult(r)).length;
const summaries = results.map((r) => {
  const output = truncateParallelOutput(getResultOutput(r));
  const status = isFailedResult(r)
    ? `failed${r.stopReason && r.stopReason !== "end" ? ` (${r.stopReason})` : ""}`
    : "completed";
  return `### [${r.agent}] ${status}\n\n${output}`;
});

return {
  content: [
    {
      type: "text",
      text: `Parallel: ${successCount}/${results.length} succeeded\n\n${summaries.join("\n\n---\n\n")}`,
    },
  ],
  details: makeDetails("parallel")(results),
};
```

这里有两层输出：

```text
content:
给主模型看的摘要文本。

details:
给 UI、调试、展开查看用的完整结构化结果。
```

如果单个任务输出太长，会截断展示：

```ts
function truncateParallelOutput(output: string): string {
  const byteLength = Buffer.byteLength(output, "utf8");
  if (byteLength <= PER_TASK_OUTPUT_CAP) return output;

  ...
}
```

这解决了并行化的另一个实际问题：

```text
并行任务越多，输出越容易爆炸。
```

如果全部塞给后续模型，可能导致：

```text
上下文超限
成本上升
综合质量下降
重要信息被淹没
```

所以并行之后必须有聚合器。

## 十八、LLM 会自己判断 Chain 还是 Parallel 吗

会，但不能完全相信。

LLM 可以输出：

```json
{
  "mode": "parallel",
  "tasks": [
    { "agent": "researcher", "task": "Find implementation files" },
    { "agent": "tester", "task": "Find related tests" }
  ],
  "reason": "These tasks only depend on the original request."
}
```

也可以输出：

```json
{
  "mode": "chain",
  "chain": [
    { "agent": "researcher", "task": "Find implementation files" },
    { "agent": "writer", "task": "Use previous result: {previous}" }
  ],
  "reason": "The second step depends on the first output."
}
```

但工程上不能只靠 LLM 自觉。

LLM 可能误判：

```text
以为两个写操作可以并行
以为测试可以在修改前跑
以为两个子任务没有共享状态
把需要审批的动作放进并行任务
```

更稳妥的方式是：

```text
LLM 提出执行结构
runtime 负责约束检查
```

Pi 的 `subagent` 就是这样：

```text
LLM 可以选择传 tasks 或 chain
schema 限制参数结构
runtime 强制只能选择一种模式
parallel 有任务数量上限
parallel 有并发上限
chain 遇到失败会停止
parallel 会汇总所有结果
```

## 十九、通用 Agent 应该怎么编码更友好

如果在开发一个通用 Agent，不应该提前假设任务一定是 chain 或 parallel。

更友好的设计是让 LLM 产出结构化计划，再由 runtime 根据依赖关系调度。

可以定义一个通用 Plan：

```ts
type Step = {
  id: string;
  kind: "agent" | "tool" | "llm" | "human";
  task: string;
  agent?: string;
  tool?: string;
  dependsOn: string[];
  readonly?: boolean;
  risk?: "low" | "medium" | "high";
};

type Plan = {
  steps: Step[];
};
```

例如：

```json
{
  "steps": [
    {
      "id": "find_impl",
      "kind": "agent",
      "agent": "researcher",
      "task": "Find relevant implementation files",
      "dependsOn": [],
      "readonly": true,
      "risk": "low"
    },
    {
      "id": "find_tests",
      "kind": "agent",
      "agent": "tester",
      "task": "Find related tests",
      "dependsOn": [],
      "readonly": true,
      "risk": "low"
    },
    {
      "id": "synthesize",
      "kind": "llm",
      "task": "Summarize findings and propose next action",
      "dependsOn": ["find_impl", "find_tests"],
      "risk": "low"
    }
  ]
}
```

调度器看到：

```text
find_impl dependsOn = []
find_tests dependsOn = []
```

就可以并行。

看到：

```text
synthesize dependsOn = ["find_impl", "find_tests"]
```

就必须等前两步完成后再执行。

这样就把问题从：

```text
让 LLM 选择 chain 还是 parallel
```

变成：

```text
让 LLM 声明依赖关系，由 runtime 自动调度
```

## 二十、通用调度器的伪代码

一个最小调度器可以这样写：

```ts
async function runPlan(plan: Plan) {
  const completed = new Map<string, StepResult>();

  while (completed.size < plan.steps.length) {
    const ready = plan.steps.filter((step) => {
      if (completed.has(step.id)) return false;
      return step.dependsOn.every((id) => completed.has(id));
    });

    const safeReady = applyPolicy(ready);

    const results = await runWithConcurrencyLimit(safeReady, 4, async (step) => {
      return runStep(step, completed);
    });

    for (const result of results) {
      completed.set(result.stepId, result);
    }
  }

  return completed;
}
```

这个模型自然支持：

```text
无依赖步骤 -> 并行
有依赖步骤 -> 顺序等待
多个阶段 -> 自动形成 parallel + chain
```

通用 Agent 的默认策略可以保守一些：

```text
无依赖 + 只读 + 低风险 -> 可以并行
写操作 -> 默认串行
高风险操作 -> 需要审批
依赖关系不明确 -> 退回 Chain
超过并发上限 -> 分批执行
```

一句话：

> 不能证明可以并行，就不要并行。

并行是优化，不是默认正确性。

## 二十一、并行模式的工程注意事项

### 1. 判断任务是否真的独立

适合并行：

```text
多来源搜索
多文件只读分析
多角度评审
多个方案生成
多个 API 查询
多个子 Agent 独立研究
```

不适合并行：

```text
读文件 -> 基于文件修改 -> 运行测试
```

后一步需要前一步结果，就是 Chain。

### 2. 警惕共享状态

危险例子：

```text
subagent A 修改同一个文件
subagent B 也修改同一个文件
```

并行更适合只读分析。涉及写操作时，最好让并行任务只产出建议，由主 Agent 串行应用。

### 3. 必须限制并发数

并行不是有多少任务就跑多少任务。

需要限制：

```text
任务总数
同时运行数
单任务超时
总预算
失败重试次数
输出大小
```

### 4. 结果顺序要稳定

执行完成顺序可以不稳定，但写回上下文的顺序最好稳定。

推荐：

```text
进度按完成顺序更新
最终汇总按任务原始顺序输出
```

### 5. 失败策略要提前设计

要明确：

```text
一个失败是否终止整个批次？
是否等待其他任务完成？
是否返回部分成功？
是否重试失败任务？
失败结果是否进入汇总？
```

研究任务通常可以部分成功。部署任务通常不能。

### 6. 并行后必须聚合

并行只是 map，后面还需要 reduce。

聚合器要处理：

```text
去重
归类
冲突检测
可信度排序
提取共识
保留少数意见
生成下一步行动
```

### 7. 高风险动作不要并行执行

不适合并行自主执行：

```text
修改同一代码库
提交代码
删除文件
发送邮件
付款
调用生产 API
数据库写入
```

更安全的模式：

```text
parallel research
  -> aggregate plan
  -> user approval
  -> sequential execution
```

### 8. 需要可观测性和取消清理

并行执行时，用户需要知道：

```text
哪些任务在跑
哪些完成了
哪些失败了
花了多少 token / 钱
输出是什么
是否被取消
```

取消时也要停止所有子任务。Pi 的 `runSingleAgent` 会监听 `AbortSignal`，对子进程发 `SIGTERM`，必要时再 `SIGKILL`。

## 二十二、总结

第三章的 Parallelization，不应该只理解成：

```text
同时调用几个 LLM
```

更准确地说，它是一种控制流优化模式：

> 在一个阶段中识别互不依赖的子任务，同时执行它们，再稳定、可控地聚合结果。

Pi 中有两个很好的落地实现：

```text
Agent Loop:
模型一次发出多个独立 tool calls，runtime 并发执行，再稳定写回 tool results。

Subagent 扩展:
主 Agent 把多个独立任务委托给隔离的子 Agent 进程，并发运行后汇总结果。
```

并行化的价值是性能和覆盖面：

```text
更快获取多个来源的信息
同时从多个角度分析问题
让不同专业 Agent 独立工作
减少 I/O 和模型调用等待时间
```

但它带来的工程复杂度也很真实：

```text
依赖判断
共享状态
并发上限
结果顺序
失败策略
输出膨胀
冲突处理
预算控制
取消清理
```

所以成熟的 Agent 并行化通常不是：

```text
parallel everything
```

而是：

```text
并行收集 / 分析
顺序聚合 / 决策
受控执行 / 审批
```

一句话总结：

> Prompt Chaining 让任务按依赖顺序推进；Routing 让任务选择不同路径；Parallelization 让同一阶段中互不依赖的任务同时推进。真正的 Agent 工程，需要把三者组合起来，并用 runtime 约束保证正确性。

## 参考资料

- [Agentic Design Patterns: Chapter 3 Parallelization](https://github.com/xindoo/agentic-design-patterns/blob/main/chapters/Chapter%203_%20Parallelization.md)
