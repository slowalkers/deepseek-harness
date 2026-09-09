# 门户外接 DeepSeek Harness 执行 Skill 的可行性分析

> 目标场景：在一台服务器上部署 DeepSeek Harness（dsh），门户（Portal）作为上层业务入口，当用户触发某个 Skill（或其他任务）时，把任务转发给 dsh，由 dsh 加载对应 Skill 并执行，最后把结果回传给门户。

**结论先行：可行，且是合理的架构，但需要先澄清一个关键概念——「执行一个 Skill」在 dsh 里并不是调用一个「可执行单元」，而是「驱动一个 Agent 在加载了该 Skill 指令的上下文中完成任务」。** 因此门户与 dsh 之间真正要建立的是一条「远程驱动 Agent 会话」的通道，Skill 是在这条通道上传递的上下文之一。本文给出可行性判断、接入路径对比与推荐方案。

---

## 1. 关键概念澄清：Skill 是「指令」，不是「可执行单元」

在 DeepSeek Harness 中，Skill 是一段可复用的 Markdown 指令（`SKILL.md`），通过 skill 能力族（[`packages/skill`](../packages/skill)）提供给 Agent 使用：

- `ctx.skills` 注册表合并各 Provider 的目录，按名称解析「获胜」的 Skill；
- 模型侧通过 `skill({ name })` 工具**加载** Skill 全文进上下文（`<skill_content>`、`<skill_instructions>`），模型随后**按这些指令行动**；
- 用户侧可通过 `/skill` 命令直接调用。

也就是说，dsh 并没有一个「执行 Skill X」的独立 RPC/端点。**「执行一个 Skill」= 创建一个 Agent 会话 + 把 <Skill 指令 + 用户任务> 注入对话 + 让模型驱动工具/子代理等去完成 + 收集结果。** 方案设计时把握住这一点，就能把目标精确落成「建立远程 Agent 会话驱动通道」，而不是去寻找一个不存在的「Skill 执行 API」。

---

## 2. 可行性与价值判断

### 2.1 为什么可行

- dsh 本身就是为「可编程驱动」设计的。它有多个面向进程外/远程的接入面（见 §3），且 Agent、会话、工具、Skill 全部是插件化的能力，可被远端触发。
- 门户只需负责**编排与展示**（谁触发、带什么任务、结果如何回传），真正的**认知与执行**（理解任务、调用工具、读写文件、跑子代理）全部下沉到 dsh。职责划分清晰。
- 已有 Skill 与 Session 的远程暴露：`ctx.remote.skills.list`（见 §3.2）已经能远程列举某会话可见的 Skill 目录。

### 2.2 值得注意的边界

- **「执行 Skill」需要你自己拼装**：目前没有现成的 `POST /run-skill` 一类接口。标准做法是「列目录 → 开/复用一个会话 → 注入任务提示（内含希望加载的 Skill）→ 观察事件直到 Agent idle → 取最终回复」。这些原语都已具备，但组合逻辑要由门户侧（或你在 dsh 上加一层薄适配）完成。
- **部署形态决定接入方式**：dsh 的 SDK/ACP 是 **stdio 子进程**协议（门户外层进程 spawn 并持有 dsh 进程）；而 API Remote 层是 **HTTP**（常驻服务，门户后端通过 `/api` 调用）。两者适合不同的门户架构，见 §4。
- **没有“无需 Agent、只返回指令文本”的语义**：如果你只想让门户「获取某 Skill 的指令内容」，那是另一个需求（用 `ctx.skills.get` / `ctx.remote.skills.list` 即可，不启动 Agent）；只有当你要「按 Skill 完成任务」时才需要会话驱动。请先确认门户要的是哪一种。

---

## 3. dsh 提供的四种远程接入面

| 接入面 | 传输 | 常驻/进程 | 适合谁 | 是否已暴露 Skill |
|---|---|---|---|---|
| **SDK**（`dsh --profile sdk`） | JSON-RPC over stdio | 子进程（spawned） | 能 spawn 进程的应用（Node/Python 后端） | 通过会话内 `skill` 工具间接使用 |
| **API Remote / Typert Gateway** | HTTP `POST /api/<ns>/<method>` | 常驻服务（`dsh web` 的 BFF） | 门户后端通过 HTTP 调用 | ✅ `ctx.remote.skills.list` |
| **ACP**（`dsh --profile acp`） | JSON-RPC over stdio | 子进程（spawned） | 自动化/无人工介入的程序 | 会话内加载，间接 |
| **Webhook runtime**（`ctx.webhookRuntime`） | HTTP 触发 → 生成普通 Session | 常驻服务内注册规则 | 外部事件驱动型门户 | 可把任务（含要用的 Skill）作为提示投递 |

下面分别说明。

### 3.1 SDK — 「从另一个进程驱动一个 dsh 运行时」

[SDK 族](../packages/sdk) 的设计意图就是「让另一个进程驱动一个完整的 dsh runtime」：客户端 spawn 一个 `dsh --profile sdk` 子进程，通过 stdio JSON-RPC 打开会话、发送提示、订阅会话事件/Agent 状态，直到 Agent idle 收最后回复。TypeScript 客户端（`@deepseek-ai/dsh-sdk-client`）与 Python SDK 是同一协议的设计孪生。

这适合**门户的后端直接是 Node/Python 进程**的场景：后端为每个任务 spawn（或复用一个）dsh 子进程，做「列 Skill → 开会话 → 注入任务 → 等 idle → 回传结果」。

### 3.2 API Remote / Typert Gateway — 「通过 HTTP 驱动常驻 dsh 的会话」

[API Gateway](../docs/api-gateway.md) 是 dsh 的 BFF：Host 用 `@Remote`/`@RemoteScope` 声明业务方法，生成 Host 契约与 Client 调用，通过共享 Connection 的 `/api` 路由做 `POST /api/<namespace>/<method>`。会话控制器（[`packages/api/session-controller`](../packages/api/session-controller)）已经提供：

- 会话生命周期（创建/回复/续会话/历史/跟随）、提示、取消；
- `ctx.remote.skills.list` —— **远程列举某 Session 的用户可调用 Skill 目录**（`SkillListRequest`/`SkillListValue`）。

这条路径适合**门户后端只通过 HTTP 与一台常驻 dsh 服务通信**的架构：门户后端是 dsh 的「Client 装配」（import `@deepseek-ai/dsh-api-remotes`），或直接按 `/api/<ns>/<method>` 协议发 HTTP。Skill 的执行同样要组合「开会话 + 提示注入 + 事件跟随」完成。

### 3.3 ACP — 「标准的 Agent 自动化协议」

[ACP](../packages/acp) 是「自动化专用」协议：程序可以不需人工介入地创建/恢复/关闭会话、发送文本与图片提示、接收语义更新、应答权限提示、取消工作。它也是 stdio JSON-RPC。如果你的门户/系统想用**标准化的 Agent Client Protocol** 遥控 dsh 的持久 Agent，这是最契合的；缺点是没有内置「运行某个 Skill」的 RPC，仍需会话内驱动。

### 3.4 Webhook runtime — 「把外部投递变成普通 Session」

[Webhook 子系统](../docs/subsystems/webhook.md) 把已认证的外部投递转成可选的普通根会话：注册一条 `WebhookRule`，收到 `WebhookDelivery` 后返回一个 `WebhookSessionRequest`（含 workspace、标题、提示文本、agent preset、权限 preset），运行时据此创建 Agent 会话并投递该提示。它是**从 HTTP 外到 dsh Session** 的现成入口；注意它是 **fire-and-forget**（无队列、无完成回写、可能重复投递），所以若门户需要同步拿结果，要么自己加一层同步回传，要么把它当作「投递即成功」的异步任务。

---

## 4. 方案对比与推荐

| 评估维度 | SDK 子进程 | API Remote (HTTP) | ACP (stdio) | Webhook (HTTP) |
|---|---|---|---|---|
| 门户需直接持有进程？ | 是（spawn 子进程） | 否（连常驻服务 HTTP） | 是 | 否 |
| 网络友好（跨机器/容器） | 中（本地 stdio） | 高（HTTP） | 中 | 高 |
| 现成 Skill 列出 | 间接 | ✅ `remote.skills.list` | 间接 | 间接 |
| 同步拿结果 | ✅ 等 idle 取最终回复 | ✅（事件跟随） | ✅ | ❌ fire-and-forget |
| 会话持久/复用 | ✅ | ✅ | ✅ | ✅（普通 Session） |
| 标准化程度 | 私有协议（设计孪生两端都有） | 私有 BFF 协议 | 标准 ACP | 事件自定 |

### 推荐（分两种门户形态）

**形态 A — 门户后端就是 Node/Python 服务，且愿意持有 dsh 进程：推荐 [SDK](#31-sdk--从另一个进程驱动一个-dsh-运行时)。**
最简单直接，`DeepSeekHarness.run('...')` / `session()` 原生支持「开会话 → 发提示 → 等结果」，文档完备，TypeScript/Python 都有客户端。Skill 通过会话内的 `skill` 工具在模型侧加载。缺点：dsh 进程生命周期挂在门户后端上，水平扩展要做进程池。

**形态 B — 门户后端通过 HTTP 调一台常驻 dsh 服务：推荐 [API Remote / Typert Gateway](#32-api-remote--typert-gateway--通过-http-驱动常驻-dsh-的会话)。**
门户后端作为 dsh 的 Client 装配连 `/api`，用 `ctx.remote.skills.list` 列目录、session 控制器开会话/发提示/跟随事件。契合「一台 dsh 服务被多个上游 HTTP 消费」的部署。若想完全标准化，可再在 HTTP 外层包 ACP 客户端。

**Webhook** 适合**异步/批处理**触发（不要求同步返回），可作补充入口，不适合作为唯一结果回传通道。

### 关于「执行 Skill」的落地建议

由于没有现成的 `run-skill` RPC，建议两种做法二选一：

1. **门户侧编排（零改动 dsh）**：门户先 `list` 拿到 Skill 目录，再打开/复用会话，把「请加载 `<skill>` 并完成以下任务：…」作为提示注入，跟随事件到 idle 取最终回复。改动最小、最快验证。
2. **在 dsh 上加一层薄适配（推荐中长期）**：以插件/Remote 方法的方式，在现有能力之上暴露一个你自己的 `//run-task(sessionId, skillName, task, opts)` 一元方法（借助 `@Remote` 与 Typert Gateway，或 SDK client 层的封装），内部完成「加载 Skill + 提示注入 + 等待 idle + 返回结果」。这符合 dsh 的插件化扩展点，把「执行 Skill」的编排固化到产品侧。

---

## 5. 结论

- **总体可行，且职责划分合理**：门户负责人机编排与结果展示，dsh 承担认知与执行。
- 但要先定义「执行 Skill」= **驱动 Agent 会话完成任务（Skill 作为上下文）**，而不是调用一个独立「Skill 执行 API」。
- **最小可行路径（快）**：SDK（持进程）或 API Remote（HTTP 常驻）+ 门户侧编排「列目录 → 开会话 → 注入任务 → 等 idle → 回传」。
- **产品化路径（稳）**：在 dsh 上通过插件/Remote 加一层 `run-task` 适配，固化编排。
- 若门户仅需「读取 Skill 指令文本」，用 `ctx.remote.skills.list` 即可，不必启动 Agent。

---

## 6. 相关仓库文档索引

- [架构总览](../docs/architecture.md)
- [Skill 子系统参考](../docs/subsystems/skills.md) / [`packages/skill`](../packages/skill)
- [API Gateway 参考](../docs/api-gateway.md) / [`packages/api`](../packages/api)
- [SDK 族](../packages/sdk) / [TypeScript 客户端](../packages/sdk/client/README.md)
- [会话控制器](../packages/api/session-controller/README.md)
- [ACP 族](../packages/acp)
- [Webhook 子系统](../docs/subsystems/webhook.md)
