# 从 Prompt Chaining 到 Routing：理解 Agent SDK 里的控制流设计

在学习《Agentic Design Patterns》第二章 Routing 时，我最大的困惑不是“Routing 是什么”，而是书里的代码容易让人分不清：

```text
这到底是在讲 function call，还是在讲 routing？
```

这种困惑很正常。因为在现代 Agent SDK 里，Routing 经常不是一个单独命名为 `router` 的模块，而是隐藏在工具选择、子智能体委托、图节点跳转、结构化输出和 agent loop 里。

如果只看 LangChain 的 `RunnableBranch`，容易把 Routing 理解成某个框架里的 API。如果只看 OpenAI Agents SDK、Vercel AI SDK 或 Pi Coding Agent，又容易把 Routing 看成 function calling 的副作用。

更清晰的理解是：

> Prompt Chaining 解决“任务怎么拆成多个步骤”；Routing 解决“下一步应该走哪条路径”。

二者不是替代关系，而是经常组合使用。

## 一、先区分 Routing 和 Function Call

Routing 和 Function Call 不是同一层概念。

```text
Routing 关心：下一步走哪条路径？
Function Call 关心：调用哪个函数，以及参数是什么？
```

例如：

```python
if intent == "booking":
    result = booking_handler(request)
elif intent == "info":
    result = info_handler(request)
else:
    result = unclear_handler(request)
```

这里有两个部分：

```text
intent == "booking" 这个判断，是 routing。
booking_handler(request) 这个执行，是 function call。
```

如果没有前面的选择逻辑：

```python
result = booking_handler(request)
```

这只是普通函数调用，不是 Routing。

因此，一个简单判断标准是：

> 系统里是否存在多个可能的后续路径，以及一个选择路径的决策点。

如果有，就是 Routing。

## 二、为什么书里的例子容易混淆

书中用 LangChain 和 Google ADK 举例。问题是这些例子里同时出现了两类东西：

```text
1. 选择分支或子智能体
2. 调用具体函数或工具
```

以旅行助手为例：

```text
用户请求
   ↓
Coordinator 判断请求类型
   ↓
Booker Agent 或 Info Agent
   ↓
booking_tool 或 info_tool
```

真正的 Routing 是：

```text
Coordinator -> Booker / Info
```

Function Call 是：

```text
Booker -> booking_handler(...)
Info -> info_handler(...)
```

也就是说：

> Function call 可以是 Routing 之后的执行动作；但 function call 本身不等于 Routing。

不过，在很多现代 SDK 里，模型“选择哪个工具”这件事本身也承担了 Routing 的角色。例如：

```json
{
  "tool": "search_order",
  "arguments": {
    "order_id": "123"
  }
}
```

这里可以拆成两层：

```text
选择 search_order = 工具路由
执行 search_order(args) = 工具调用
```

这就是两者最容易混在一起的原因。

## 三、Prompt Chaining 和 Routing 的关系

上一章 Prompt Chaining 关注的是线性控制流：

```text
Step A -> Step B -> Step C
```

例如：

```text
1. 先总结用户问题
2. 再提取关键信息
3. 最后生成回答
```

不管用户问什么，流程都按固定顺序执行。

Routing 关注的是分支控制流：

```text
Step A -> 判断类型
       -> Route B1
       -> Route B2
       -> Route B3
```

例如：

```text
1. 先判断用户意图
2. 如果是退款，进入 refund_chain
3. 如果是物流，进入 shipping_chain
4. 如果不明确，进入 clarification_chain
```

所以二者的关系可以概括为：

```text
Prompt Chaining = 顺序执行
Routing = 条件选择下一条 chain
```

更准确地说：

> Routing 通常发生在 Chain 的某个节点上。它决定下一步接哪条 Chain、哪个工具、哪个子智能体或哪个工作流。

一个真实系统往往长这样：

```text
用户输入
   ↓
Router 判断任务类型
   ↓
进入某条 Chain
   ↓
Chain 中某一步再由模型选择工具
   ↓
工具结果回到上下文
   ↓
模型继续选择下一步
   ↓
结束、追问或 handoff
```

因此，Routing 不是 Prompt Chaining 的替代品，而是让 Chain 从线性流程升级为条件流程。

## 四、现代 Agent SDK 里，这两个概念如何出现

今天做 Agent 开发，不一定使用 LangChain。常见选择包括 OpenAI Agents SDK、LangGraph、Google ADK、LlamaIndex、Vercel AI SDK、PydanticAI 等。

这些 SDK 的 API 不同，但底层都在组合类似的原语：

```text
Prompt
Structured Output
Tool Call
State
Router
Workflow / Graph
Handoff
Guardrail
Trace
```

可以这样对照：

| SDK / 框架 | Prompt Chaining 常见形态 | Routing 常见形态 |
| --- | --- | --- |
| OpenAI Agents SDK | Runner 管理 agent loop、工具调用和多轮执行 | tool choice、handoff、agent-as-tool、guardrail 后分支 |
| LangGraph | Graph node 串联 | conditional edge、router node、Command |
| LlamaIndex | query pipeline、RAG workflow | Router Query Engine、query engine selection |
| Google ADK | agent + tools + sub-agents | coordinator 委托给 sub-agent、multi-agent workflow |
| Vercel AI SDK | tool calling steps、tool loop | 模型选择工具，工具结果回到上下文后继续 |
| PydanticAI | agent graph、structured output pipeline | 用结构化输出约束 route 或最终结果 |

OpenAI Agents SDK 文档中把 runner 描述为负责 agent loop：调用当前 agent 的模型，检查输出，执行工具调用，处理 handoff，直到产生最终答案或暂停等待批准。这就是现代 SDK 对 Prompt Chaining、Routing 和 Tool Calling 的整合形态。

LangGraph 的文档则更显式地区分了 workflow 和 agent：workflow 有预先确定的代码路径，agent 动态决定过程和工具使用。Routing 在 LangGraph 里通常表现为 conditional edge，也就是根据当前状态选择下一个节点。

这说明：框架不同，名称不同，但核心问题相同：

```text
当前状态是什么？
下一步应该走哪条路径？
这条路径由程序决定，还是由模型决定？
结果如何进入下一步上下文？
```

## 五、三类常见 Routing

### 1. 显式 Routing

显式 Routing 是最容易理解的形式：

```ts
const route = await classify(input);

if (route === "refund") {
  return refundChain(input);
}

if (route === "tech_support") {
  return supportChain(input);
}

return clarificationChain(input);
```

这种模式适合高可靠业务流程：

```text
客服
审批
工单
支付
医疗
合规
企业内部流程
```

优点是可解释、可测试、可审计。缺点是灵活性有限，需要提前设计路径。

### 2. 隐式工具 Routing

隐式工具 Routing 不写独立 Router，而是把工具列表交给模型：

```ts
tools = [readFile, searchCode, editFile, runCommand]
```

模型根据上下文输出：

```json
{
  "tool": "searchCode",
  "arguments": {
    "query": "AgentTool"
  }
}
```

这里：

```text
选择 searchCode = routing
执行 searchCode(args) = function call
```

这种模式适合开放任务：

```text
Coding Agent
研究助手
数据分析 Agent
调试 Agent
探索式 RAG
```

优点是灵活。缺点是路径不完全可预测，需要 trace、工具权限、审批和测试来兜底。

### 3. Handoff Routing

Handoff Routing 不是选择工具，而是选择把任务交给哪个 Agent：

```text
Coordinator
   -> BillingAgent
   -> TechSupportAgent
   -> ResearchAgent
   -> WriterAgent
```

适合任务需要明显专业角色的场景：

```text
合同分析：法律 Agent + 财务 Agent + 摘要 Agent
投资研究：数据 Agent + 风险 Agent + 报告 Agent
客服系统：订单 Agent + 退款 Agent + 技术支持 Agent
```

如果每个分支需要不同 instructions、tools、memory 或 policy，就适合拆成子 Agent。

如果只是一个小函数能解决，不需要上升为子 Agent。

## 六、以 Pi Coding Agent 理解隐式工具 Routing

Pi Coding Agent 很多时候没有写一个显式 `router_chain`。

它的做法更像：

```text
1. 把 read、bash、edit、write 等工具暴露给模型
2. 在 system prompt 里说明工具用途
3. 每轮调用模型时，把当前消息和工具列表一起发给模型
4. 模型输出 toolCall
5. Pi runtime 根据 toolCall.name 找到对应工具并执行
6. 工具结果作为 toolResult 回到上下文
7. 模型继续判断下一步
```

这个循环可以写成：

```text
用户请求
   ↓
LLM 选择 read / bash / edit / write / final answer
   ↓
Pi 执行对应工具
   ↓
toolResult 回到上下文
   ↓
LLM 再选择下一步
   ↓
直到没有 toolCall
```

如果用户说：

```text
修复这个测试失败
```

Pi 可能会这样走：

```text
LLM -> grep / rg 查找相关测试
toolResult -> 搜索结果
LLM -> read 读取文件
toolResult -> 文件内容
LLM -> edit 修改代码
toolResult -> 修改结果
LLM -> bash 运行检查
toolResult -> 检查结果
LLM -> final answer
```

从 Prompt Chaining 看，这是多轮上下文传递。

从 Routing 看，每一轮模型都在选择下一步路径：

```text
下一步是 read？
下一步是 bash？
下一步是 edit？
还是直接回答？
```

所以 Pi 同时体现了两种模式：

```text
Prompt Chaining: 上一步结果进入下一轮 prompt
Routing: 每一轮选择下一个工具或停止
```

这也是现代 Agent Loop 的核心。

## 七、Routing 设计时最重要的原则

### 1. Route 要按后续处理差异划分

不要为了分类漂亮而拆 route。

不好的设计：

```text
price_question
discount_question
coupon_question
```

如果这三个 route 后面都走同一个商品政策查询流程，就应该合并：

```text
product_commercial_info
```

Routing 的目的不是给用户意图贴标签，而是决定不同处理路径。

### 2. Route 之间要尽量互斥

不好的设计：

```text
order_support: 处理订单问题
shipping_support: 处理物流问题
```

用户问：

```text
我的订单什么时候送到？
```

这同时像订单问题，也像物流问题。

更好的设计：

```text
order_status:
查询订单是否存在、支付状态、取消状态。
不处理发货后的物流轨迹。

delivery_tracking:
查询发货后的物流位置、预计送达时间、配送异常。
不处理未支付或已取消订单。
```

Route 语义越重叠，模型越容易选错。

### 3. 相似 route 必须写反例

只写正向描述不够。

例如：

```text
billing_support:
Use for invoices, refunds, payment failures.
Do not use for login, password, profile, or permission issues.

account_support:
Use for login, password, profile, permissions.
Do not use for invoices, refunds, or payment failures.
```

反例能帮助模型建立边界。

### 4. 设置 unclear / fallback route

不要强迫模型在两个相似分支里二选一。

应该允许：

```json
{
  "route": "unclear",
  "question": "请问你是想查询订单支付状态，还是查询物流配送状态？"
}
```

这能避免系统看起来自动化，实际上在边界场景乱选。

### 5. Route 输出要结构化

不要让模型自由输出：

```text
I think this is probably billing.
```

应该约束为：

```json
{
  "route": "billing",
  "confidence": 0.86,
  "reason": "用户询问重复扣款",
  "missing_info": []
}
```

结构化输出的好处是：

```text
程序可以验证 route 是否在枚举内
可以按 confidence 决定是否追问
可以记录 reason 便于排查
可以做 routing eval
```

### 6. 高风险路径要加入审批

这些动作不适合只靠隐式 Routing 自动执行：

```text
付款
删除数据
发送邮件
提交代码
修改生产配置
调用不可逆外部 API
```

更稳妥的流程是：

```text
route
   ↓
prepare_action
   ↓
user_approval
   ↓
execute
```

Agent 越能自主行动，越需要 guardrail、approval、trace 和回滚策略。

### 7. Routing 必须测试

至少准备三类测试样本：

```text
happy path: 明显属于某 route
boundary case: 两个 route 都有点像
unclear case: 信息不足，需要追问
```

例如：

```text
"我想取消订单" -> order_status
"订单什么时候送到" -> delivery_tracking
"我还没下单，能送到北京吗" -> pre_sale_info
"我付钱了但没收到确认邮件" -> 需要明确优先级或进入 unclear
```

没有 routing eval，系统迟早会在边界问题上出错。

## 八、什么时候用 Chain，什么时候用 Routing

可以用这个判断公式：

```text
确定性高的地方，用 Chain。
分支选择的地方，用 Routing。
需要外部动作的地方，用 Tool Calling。
需要专业角色的地方，用 Handoff。
不确定或高风险的地方，用 Clarification / Approval。
```

如果流程稳定：

```text
classify -> retrieve -> answer -> verify
```

适合 Prompt Chain。

如果用户请求类型差异很大：

```text
refund / shipping / technical_support / unclear
```

适合 Routing。

如果任务开放、路径不可预先枚举：

```text
读代码、搜索、修改、运行检查、继续探索
```

适合 Agent Loop + Tool Routing。

如果每个分支都有独立角色和工具：

```text
ResearchAgent / CodeAgent / ReviewAgent
```

适合 Handoff Routing。

## 九、一个完整的设计模板

设计一个 Agent 系统时，可以按下面的问题展开：

```text
1. 用户任务是否有固定步骤？
   有 -> 先设计 Prompt Chain。

2. 中间是否需要根据输入或状态选择路径？
   有 -> 增加 Router。

3. 每个 route 后面的处理流程是否真的不同？
   不同 -> 保留 route。
   相同 -> 合并 route。

4. route 是否容易混淆？
   容易 -> 增加反例、优先级、fallback。

5. route 输出是否结构化？
   否 -> 增加 schema。

6. 是否涉及高风险动作？
   是 -> 增加 approval 和 guardrail。

7. 是否需要多个专业角色？
   是 -> 使用 handoff 或 agent-as-tool。

8. 是否有测试样本覆盖边界场景？
   没有 -> 先补 routing eval。
```

这比一开始就纠结“用 LangChain 还是 OpenAI Agents SDK”更重要。

SDK 只是实现方式。控制流设计才是 Agent 应用的骨架。

## 十、总结

Routing 不是简单的 function call，也不是 LangChain 某个 API 的名字。

它是一种控制流设计模式：

> 根据输入、上下文、工具结果或系统状态，选择下一条处理路径。

Prompt Chaining 和 Routing 的关系是：

```text
Prompt Chaining 解决步骤拆分。
Routing 解决路径选择。
Agent Loop 把“模型判断、工具执行、结果回传、继续判断”组合成动态流程。
```

在现代 Agent SDK 里，这些能力经常被封装成：

```text
tool calling
structured output
conditional edge
handoff
agent-as-tool
guardrail
trace
```

所以学习 Routing 时，重点不是记住某个框架的写法，而是建立一套判断：

```text
这里是否有多个后续路径？
路径是否真的需要区分？
谁来做选择：代码、规则、模型、embedding，还是分类器？
选择结果是否结构化？
错误选择时是否可回退？
高风险动作是否需要审批？
边界样例是否测试过？
```

把这些问题想清楚，Routing 才会从“模型选了一个函数”变成真正可设计、可验证、可维护的 Agent 控制流。

## 参考资料

- [Agentic Design Patterns: Chapter 2 Routing](https://github.com/xindoo/agentic-design-patterns/blob/main/chapters/Chapter%202_%20Routing.md)
- [OpenAI Agents SDK: Agents guide](https://developers.openai.com/api/docs/guides/agents)
- [OpenAI Agents SDK: Running agents](https://developers.openai.com/api/docs/guides/agents/running-agents)
- [OpenAI Agents SDK: Orchestration and handoffs](https://developers.openai.com/api/docs/guides/agents/orchestration)
- [LangGraph: Workflows and agents](https://docs.langchain.com/oss/python/langgraph/workflows-agents)
- [LangGraph: Graph API overview](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [Google ADK](https://adk.dev/)
- [Vercel AI SDK: Tools and tool calling](https://ai-sdk.dev/docs/ai-sdk-core/tools-and-tool-calling)
- [PydanticAI: Structured output](https://pydantic.dev/docs/ai/core-concepts/output/)
