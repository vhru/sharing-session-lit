# Train-ticket AI Agent — English speaker notes

对应当前 17 页 slides。此次仅编写讲稿与改进意见，未修改 slides 或其内嵌 notes。

英文正文约 1,360 词，另有约 25 词的 Q&A 开场。按每分钟约 100 词，加上指图和停顿，目标约 15 分钟，另留 5 分钟 Q&A。实际时间以彩排为准。

使用方式：只朗读每页的英文正文。中文提示用于记忆重点。英文中的 **加粗句** 是建议停顿强调的设计判断。各页最后一句已尽量承担转场。

## 先回答两处内容问题

### 第 9 页：“本例应呈现的内容”是什么意思？

原意是“回到开场的查询例子，用户最终需要看到什么”：更新后的查询条件，以及查询接口返回的结果或当前状态。它是编辑说明式措辞，放在演示页上显得像尚未完成的占位文字。

建议后续改为“用户最终看到什么”，保留两项：

- 更新后的条件：后天，虹桥到杭州，14:00 前到达。
- 查询返回的结果或状态，具体内容以接口结果为准。

这一块只负责让 walkthrough 收尾。第 9 页的核心仍是输出检查与流式体验之间的取舍，不需要另外展开一段结果展示设计。

### 取舍应该在哪里讲？

从请求处理部分开始讲，并在工程机制部分继续展开。当前版本把“怎么运行”讲清楚了，但部分页面没有充分说出“为什么这样选，以及需要承担什么代价”。

建议重点讲下面四组，其他页面简要带过：

| 位置 | 已有设计 | 值得讲出的选择理由 | 同时承担的代价 |
| --- | --- | --- | --- |
| 第 3、7、8 页 | LLM 理解表达，JSON draft 保存任务条件，FSM / PocketFlow 组织执行 | 执行条件明确，多轮变更可检查 | 需要维护 schema、状态与转换规则 |
| 第 5 页 | 从业务站点目录获得真实候选，再结合任务判断 | 可查询实体与当前业务数据对齐 | 目录更新、候选表示和索引都需要维护 |
| 第 6 页 | 小规模数据采用本地检索，结合词项、向量与重排 | 按当前站点规模控制基础设施投入，同时处理不同表达 | 本地索引仍需维护，多阶段检索仍有延迟与调试成本 |
| 第 9 页 | 先缓冲、检查和脱敏，再释放内容 | 用户看到内容前保留检查机会 | 增加首段等待，需要同时看首段与完整耗时 |

第 10–13 页再补充工程层面的取舍：标准文案的一致性与覆盖范围、并行的延迟收益与额外工作、缓存复用与失效边界、失败恢复与结果可信度。不用每页都机械地说一次 “The trade-off is…”。

这些是对已有设计的理由与代价的解释，不代表当时做过所有备选方案的 A/B 测试。语言选择策略、缓存复盘检查项和最后两页的闭环建议，继续与已实现机制区分。

---

## Page 1 — Building a business AI agent

中文定位：开场；说明自己从 0 到 1 的项目，以及分享范围。

Hi everyone. Today I'll share a train-ticket agent I built from scratch at Trip.com.

It was designed for customers, so it had to handle changing requests and connect them to real business services.

I'll walk through one request, explain a few design choices, and finish with what I'd improve in the development process.

## Page 2 — The user task

中文定位：用同一个两轮查询串起全篇。

Let's start with a simple example.

A user says, “I want to travel from Shanghai to Hangzhou tomorrow, and arrive before two p.m.”

Then they say, “Actually, make it the day after tomorrow, and leave from Hongqiao.”

The agent needs to update the date and departure station. But it must keep Hangzhou and the arrival-time limit.

It then needs to find the right station and run the search. I'll use this example throughout the talk.

## Page 3 — System overview

中文定位：先讲职责，突出模型理解与流程控制的分工。

This diagram shows the main responsibilities in the system.

At the top, we have routing and input checks, plus checks on the final output.

In the middle, the agent understands the request, updates the task state, and calls tools. We used a state machine with PocketFlow to organize these steps.

At the bottom are the supporting inputs: conversation memory, the station catalog, and business APIs.

**We gave the model room to understand the user, while keeping the execution conditions explicit.**

That made the flow easier to inspect. It also meant maintaining the schema and state transitions.

## Page 4 — Routing and context

中文定位：入口决定归属；本轮 query 与压缩 memory 提供理解输入。

First, the main agent routes the request to the train-ticket agent.

Our backend layer, or BFF, checks the content and business scope with a lightweight model.

We then load the prompt for the current task. The model also receives the current query and compressed conversation memory.

This keeps the input focused. But compression needs care, because details from an earlier turn may still matter.

Now the agent has enough context to work on the user's place names.

## Page 5 — Matching business entities

中文定位：模型理解“虹桥”，业务数据限定可查询的真实站点。

In our example, the user says “Hongqiao.” The earlier turn gives us Shanghai as context.

The model can use those clues, but the search API needs an actual station from the business catalog.

So we retrieve real candidates and use the user's task to help select the right one.

**This gives us control over which entities the agent can use, but it creates data-maintenance work.**

Stations can open, close, or change. We need to keep the catalog and its search representation up to date.

## Page 6 — Hybrid retrieval

中文定位：本页是重点取舍；按规模选择组件，并解释混合检索的代价。

To find those candidates, we combined keyword and rule matching with vector search in Faiss.

We then reranked the top ten candidates with BGE. The LLM used the user's intent to make the final judgment.

There were fewer than ten thousand stations, so we didn't add Elasticsearch just for this search.

**Keeping the index local reduced infrastructure needs, but we still had to maintain it.**

We also gave more weight to short station names, so longer location descriptions would not dominate the match.

The extra retrieval and ranking steps also added work and latency to the request.

## Page 7 — Task state

中文定位：区分 memory 与 draft；指表格中更新和保留的字段。

Next, we put the current task conditions into a JSON object called the draft.

Memory helps the model understand the conversation. The draft records the fields we need for execution.

Here, the date changes to the day after tomorrow, and the departure changes to Hongqiao. Hangzhou and the two p.m. limit stay the same.

**An explicit draft makes these changes visible and easier to check.**

The cost is maintaining a schema that matches the business. For the actual API call, we also need the proper date format and station identifiers.

## Page 8 — Tool execution

中文定位：信息不足就追问；执行由当前状态与门控结果共同约束。

Once we have the draft, we check the required fields.

If something is missing, we ask the user and update the draft again. If the information is ready and the input checks have passed, we can call the business tool through MCP.

PocketFlow and the state machine organize the flow. MCP provides access to the tools.

Asking a follow-up question adds another turn, but it helps avoid a call with missing information.

After the tool returns, we still need to prepare the response for the user.

## Page 9 — Returning the result

中文定位：图的重点是先检查再释放；左下内容只负责闭合案例。

We buffer a small piece of generated content in the BFF, check it, mask sensitive information, and then release it.

**This gives us a chance to check the content before the user sees it. The cost is extra waiting.**

So we need to look at both the time to the first visible response and the time to finish the task.

For our example, the response should show the updated travel conditions and the search result or current status.

That completes the request. Next, I'll cover a few engineering concerns across the flow.

## Page 10 — Localization

中文定位：区分语言选择策略与已经使用的标准文案系统。

Localization involves both choosing a language and managing the wording.

The strategy shown here uses the model to identify the language, with the app preference as a fallback when confidence is low.

For standard product messages, we used Shark. Translators prepared and reviewed the text, and the application retrieved it by key.

This gave us consistent wording across languages, but only for the messages covered by those templates. Generated answers still needed separate handling.

An extra model call for language selection would also add latency.

## Page 11 — Reducing latency

中文定位：并行只改变等待关系，执行前提仍然保留。

For performance, we looked at which steps really depended on each other.

For example, input checks and entity retrieval can run in parallel when neither needs the other's result.

Execution still waits until the checks pass and the parameters are ready.

This can reduce waiting, but if the checks reject the request, the retrieval work may be wasted.

We also looked at initialization and prompt size. The aim was to reduce unnecessary work while keeping the conditions for execution clear.

## Page 12 — Caching

中文定位：已有缓存对象可以讲；右侧条件是复盘检查清单。

We also cached query-related data, retrieval results, and intermediate MCP results.

A usable cache entry lets us skip repeated work at that stage. The rest of the request still follows the normal flow.

The review question is whether the cached result still fits the current task.

For example, if the user changes the travel date, an earlier result may no longer apply.

**Caching saves calls, but it creates responsibility for freshness and context.**

The table is a review checklist for those conditions.

## Page 13 — Handling failures

中文定位：按失败类型选择路径，不暗示固定的降级顺序。

When a tool fails, the next step depends on the failure.

We may retry or use a backup API. We may use a cached result if it still applies. We may also skip a step that does not block the main task.

These are different options, not a fixed sequence.

Each has a condition for continuing. If none is valid, we need to explain the current status to the user.

Recovery should help the user continue without presenting an unreliable result as a successful one.

## Page 14 — Evaluation

中文定位：trace 支持定位；固定测试集与真实请求回答不同问题。

To evaluate the agent, we need to connect all the steps in one request through a trace.

This helps us see whether a problem came from entity matching, state updates, tool execution, or the response.

We also look at response time and whether our tests cover real user needs.

A small golden test set gives us a quick way to check for regressions. Real queries show which cases matter in practice.

Both need clear expected results. Otherwise, a score is hard to interpret or use for a release decision.

## Page 15 — The technical feedback loop

中文定位：明确转入复盘建议；把采集接回样本与开发。

The final two slides are lessons I'd apply earlier if I built this again.

First, I'd connect development to a stronger feedback loop.

After a change, we run regression tests. Once the change meets the release criteria, we observe real requests and collect feedback.

Then we investigate failures, select useful cases, and add expected results. Those cases become tests for the next change.

**This is the data flywheel I want: real usage creates better test cases, and those cases guide development.**

The fix may belong in the data, the prompt, or the workflow.

## Page 16 — Alignment with the business

中文定位：收束到 AI Engineer 的系统判断；技术闭环需要共同成功标准。

The second lesson is to agree on success criteria with the business early.

What counts as a completed task? Which cases must work? What evidence do we need before release?

A small test set supports fast development. Testing against real query patterns helps us judge business coverage. We need both to inform the same release decision.

Feedback can then guide the next priorities using criteria that both sides understand.

For me, building an agent includes designing the execution flow and creating a shared way to judge whether each change makes it better.

## Page 17 — Q&A

中文定位：停下来，把时间交给提问。

That's the experience I wanted to share. Thanks for listening.

I'm happy to discuss the design choices or how they might apply to your projects.

---

## 演讲时的事实边界（不朗读）

- 第 3、5–9 页的取舍，是对现有设计的解释。不要临时扩展成“我们证明此方案优于所有其他方案”。
- 第 10 页：Shark 的使用已确认。模型语言判断加 App fallback 是当前材料中的策略口径，完整上线状态仍需核对。
- 第 11 页：图表达依赖关系，不是实测延迟数据。“门控拒绝时可能浪费检索工作”是并行设计的可能代价，不是已记录的事故。
- 第 12 页：缓存对象已确认；Key、TTL、隔离、失效与回写细节未核对。讲稿保留设计原则，不补具体数值。
- 第 14 页：讲评估框架，不宣称已有量化收益或覆盖率。
- 第 15–16 页：明确作为复盘建议，避免用 “we already had the complete loop” 等已完成口径。
- 如被追问暂时无法核实的细节，可说：“I don't have the exact configuration with me, so I don't want to guess. The design concern was…” 然后回到自己能确认的部分。

## 英文讲解的小提示（不朗读）

- “TOC Agent” 在口语里用 “an agent for customers” 更自然；开场无需解释 TOC 缩写。
- 首次说 “a JSON object called the draft”，之后直接用 “the draft”。
- “取舍” 可以自然说成 “This helps us…”, “The cost is…”, “But we still need to…”。不用每页重复 “trade-off”。
- 页与页之间直接沿用最后一句转场。讲解表格或图时可以稍停，指一下当前节点，再继续。
