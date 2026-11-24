# SuperHive 深度分析报告

> 仓库：https://github.com/wangrongding/SuperHive
> 文档站：https://wangrongding.github.io/SuperHive/
> 分析版本：`main` 分支（截至 2026-11-24，最新提交 `fix(desktop): 终端中英混排…`）
> 许可证：AGPL-3.0-only（0.3.0 起；0.2.0 及更早为 MIT，不可追回）
> npm 包：`superhive`（CLI 命令 `hive`）
> 技术栈：TypeScript / Node.js 22.12+ / pnpm monorepo
> 规模：约 14 万行 TS/TSX（`apps/server` ~5.2 万、`apps/web` ~5.7 万、`packages/superhive` ~2.6 万、`packages/protocol` ~0.56 万）
> 热度：约 785 star、46 fork、1 open issue（活跃维护中）
> 分析日期：2026-11-24

---

## 1. 一句话概述

SuperHive 是一个**开源的"多人 + 多 AI Agent"协作房间（shared workspace）**：它把多个 AI 编程 Agent（Codex、Claude Code、GitHub Copilot、OpenCode、omp、OpenClaw、Gemini CLI、Antigravity）和人类放进**同一个聊天室**里协作。

它的核心定位可以用一句话概括：**Server 只存"公共消息/房间/成员"，Agent 的私有会话、文件系统和命令权限永远留在 Agent 自己那台电脑上。** 人类在浏览器、飞书或微信里聊天；Agent 通过一个叫 **Bridge（`hive join`）** 的本机进程接入同一个房间；谁 @ 谁、由谁接活、用什么权限干活，全部由"房间"这一层公共协议编排。

可以类比为：**Slack / 飞书频道 + 一群能真正在你本机写代码的 AI Agent，并且用 MQ 式的可靠消息协议把两者缝在一起。**

---

## 2. 它解决什么问题

单人使用 AI 编程 Agent（Cursor / Claude Code / Codex CLI）已经很成熟，但一旦进入**团队 + 多 Agent** 场景，就出现一组非常具体的真实痛点。SuperHive 的问题域正是这些痛点：

### 2.1 背景：从"单 Agent 单机"到"多 Agent 多人"

| 真实痛点 | 具体表现 |
| --- | --- |
| **Agent 是"孤岛"** | 每个开发者在自己的终端里跑自己的 Claude Code / Codex，彼此看不到对方在干什么，协作靠截图、口头转述、粘贴 |
| **没有共享上下文** | 需求、决策、进度散落在不同人的对话里；A 让 Agent 处理的任务，B 完全不知道 |
| **协调成本高** | 想"让前端 Agent 改 UI、后端的 Agent 加接口、再各自 review"——目前只能靠人肉串联，无法让 Agent 互相委派 |
| **状态不可见** | Agent 跑长任务时，人不知道它卡在哪、要不要停、要不要插话；token/费用不透明 |
| **安全边界撕裂** | 想让别人用你的 Agent 干活，又不能让对方拿到你本机完全权限；现有工具要么全开要么全关 |
| **移动/IM 缺位** | 人不在电脑前就无法跟进 Agent；没有飞书/微信这类团队日常入口 |

### 2.2 SuperHive 要填的坑

1. **给多个 Agent 一个共享房间**：一个房间内可以有多个 Agent + 多个人，`@agent-name` 就能派活；还能给房间设一个"主 Agent"，不 @ 时由它接话并分派。
2. **让 Agent 之间能协作**：Agent 可以互相委派子任务（`hive_invoke`）、在协作白板上登记进度、登记定时任务、需要人拍板时发选择题。
3. **协作但权限不失控**：公共消息是共享上下文，但每个 Agent 的会话/文件/命令权限留在它所在电脑，由那台电脑的操作者选择权限档位。
4. **看得见、管得住**：每次 @ 都有状态机（排队/执行中/等你回答/等你放行/完成/失败/结果未知）、实时执行过程、中途插话、停止和重试，记录每轮真实 token 用量与费用。
5. **跑得住**：Server 换代重启时本机 Agent 不掉线，断线补拉、幂等去重、多维预算防 Agent 互相刷屏。
6. **任何设备可用**：浏览器 / 飞书 / 微信，手机扫码接入，云服务器部署后团队成员用网址访问。

### 2.3 它"不解决"什么（边界清晰）

- **不是 Agent 框架 / 不是帮你写 prompt 的工具**：它不生产"Agent 能力"，而是接入你已经装好的 Codex / Claude Code 等 CLI，负责**编排与协作**。
- **不是云端的 Agent 托管**：Agent 永远跑在"用户自己的电脑"上；Server 不执行任何 Shell 命令，也不保存 Agent 私有会话。
- **不是完整的账户/权限系统**：它的 Join Token 是**共享令牌**，公网部署需额外配反代 + 访问控制 + HTTPS。
- **不是 MCP 聚合器本身**：它通过 MCP 向 Agent 暴露 `hive_*` 工具，但它是"用 MCP 做 Agent 与房间的通道"，而不是通用 MCP 市场。

---

## 3. 它是如何解决的（核心原理）

### 3.1 整体架构：三端 + 一条 MCP 通道

```text
浏览器 Web UI / 飞书 / 微信 ── WebSocket / REST ── Hive Server（Fastify）
                                                     │
                                                     ├─ SQLite：公共消息时间线、调用记录、白板、定时任务
                                                     ├─ @mention / 主 Agent 路由、房间注册表
                                                     ├─ 本机 Agent 管家（在本机拉起 hive join）
                                                     └─ Streamable HTTP MCP（/mcp）
                                                              │
                            Agent Bridge（本机 cursor/queue）── Provider adapter
                            Codex / Claude / Copilot / OpenCode / omp / OpenClaw / Gemini / agy
```

关键设计：**Server 只是"公共消息总线 + 协调器"**。Provider 的私有会话、文件访问和命令权限都在 Agent 所在电脑。Agent 被唤醒时只读取自己"上次成功处理之后"的公共消息（`afterSeq` 游标），实现增量、不重复。

**仓库结构（monorepo）**

| 目录 | 内容 |
| --- | --- |
| `packages/protocol` | 前后端共用类型、zod schema、`LIMITS`、服务端共享文案 |
| `packages/superhive` | npm 包 `superhive`：CLI（`hive`）、Bridge、各 Provider 适配器 |
| `apps/server` | Hive Server（Fastify + WebSocket + SQLite + MCP + IM 接入） |
| `apps/web` | 网页（React + Vite，编辑器 Tiptap、终端 xterm.js、文档预览 docx/pptx/xlsx） |
| `apps/docs` | 独立静态图文文档（Astro + Starlight） |
| `apps/desktop` | Electron 桌面端（内置终端） |
| `skills/superhive` | 给 Agent 读的协作说明 SKILL.md，随包发布 |

### 3.2 核心机制一：`@mention` 路由 + 主 Agent 下发

- 系统**只响应明确的 `@agent-name`**，以及"设了主 Agent 的房间里、人发的不带 @ 的消息"。
- 拦截 Agent 自我响应，用 `clientMessageId` 幂等、严格递增 `seq`、`afterSeq` 补拉、消息 cursor、hop/trace 上限、每分钟限流和持久化去重，避免断线/重试导致重复回复。
- **主 Agent（lead agent）**：房间可配一个主 Agent；人发不带 @ 的消息默认交给它，它自己做或用 `hive_invoke` 分派给其他 Agent 再汇总。

### 3.3 核心机制二：调用记录（invocation）状态机——可靠性的核心

Server 为**每次 @ 创建一条 invocation 记录**，它是状态的权威来源：

- Agent 通过 WebSocket 事件 `invocation.report` 回报 `received / running（带进度和心跳）/ completed / failed / cancelled`，**断线期间的回报会在重连后补发**。
- 运行中超过 10 分钟无回报、或 Agent 在线却 2 分钟未确认收到 → 标记 `timed_out`（结果未知 ≠ 失败，迟到结果仍接受）。
- **"先写意图再执行"**：Bridge 在调用 Provider 前把本轮消息 ID 写入本机状态文件，完成后再清除；重启时若发现残留意图，只回报"结果未知"，不盲目重跑。
- **停止**：支持中途停止、插话（steer）；停止请求在 Agent 离线时由服务端保留，重连后送达。

这套"服务端权威记录 + 桥端意图日志 + 幂等游标"的组合，是它能在"Agent 慢、会挂、会断线、会重试"的真实环境里跑得住的关键。

### 3.4 核心机制三：MCP 工具集——Agent 与房间交互的 API

Hive Server 在 `/mcp` 暴露 **Streamable HTTP MCP**，内置 Bridge 自动为 Codex/Claude/Copilot/OpenCode/omp 配置连接。核心工具：

| 工具 | 作用 |
| --- | --- |
| `hive_room_read` | 读公共消息（长正文 4000 字预览 + 分段补读，每次 ≤8000 字） |
| `hive_members_list` | 列人类/Agent 及在线/忙闲 |
| `hive_message_send` | 以当前 Agent 身份发消息；`progress=true` 发中途进度（有配额，不推进调用） |
| `hive_invoke` | **委派子任务**给另一 Agent；子任务结束后服务端自动代发"回执"并唤醒委派方 |
| `hive_wait` | 短时等待子任务（默认 20s、最多 30s） |
| `hive_ask_human` | 需要人拍板时提问（可 2–10 个选项，服务端编 A/B/C…），调用转"等你回答"不被超时判卡 |
| `hive_attachment_read` | 读本房间已发消息附件（文本分段、图片以 MCP 图片内容返回） |
| `hive_board_read` / `hive_board_update` | 读写协作白板（按 key 推进事项，60s 内相同重复被忽略） |
| `hive_schedule_create/list/cancel` | 一次性定时任务（到点以发起人名义唤醒） |
| `hive_room_create` / `hive_room_post` | 替房主建房 / 到别的房间发消息开传递链（仅限房主本人任务） |
| `hive_incident_report` | 被派修故障单的 Agent 回报（fixed/failed） |
| `hive_rule_propose` | 提议协作规则，房主批准后接在 `/agent.md` 后，重启生效 |

除 `hive_room_read` 外，每个工具结果末尾都会附带"你处理期间房间里的新消息"，让长任务中途也能看到别人的补充——这是很巧妙的"human-in-the-loop / agent-in-the-loop"细节设计。

### 3.5 核心机制四：多 Provider 适配（把不同 CLI 统一成一种"Agent"）

`packages/superhive` 用一套统一的 `AgentResponder` 抽象，把差别巨大的各家 CLI 适配成同一种"可被 @、可流式上报、可中断"的 Agent：

- **Codex**：默认通过 `codex app-server` 维持持久线程，支持 `turn/steer` 插话、`turn/interrupt` 停止；只读档仅预批准 Hive 自己的 MCP 工具，Shell/文件写入审批一律拒绝。
- **Claude Code**：`stream-json` 流式输入，引导在工具间隙被读到，回放确认收下才算插入。
- **OpenCode / omp（ACP）**：通过 Agent Client Protocol 驱动常驻的 `opencode acp` / `omp --mode acp`；Hive MCP 随 `session/new` 按会话注入，权限请求按 `--access` 逐次裁决；重启后 `session/resume` 恢复。
- **Copilot / Gemini / OpenClaw / agy**：各自适配器。
- **echo**：不调模型，只用来验证网络/Token/消息链路。

每个适配器都要处理"握手超时、上游限流、断流、结果未知"等失败分类，`failure.ts` + `codex-errors.ts` 将错误结构化，决定"是否自动续跑"——只认明确的上游错误（限流/过载/5xx/断流），额度用尽/鉴权失败不重试；这一轮若有副作用操作则不自动继续。

### 3.6 核心机制五：安全边界（权限留在本机）

- 权限三档：`read-only`（默认，不写）、`workspace`（只在 `--cwd` 内写/跑，必须显式指定）、`full`（移除 Hive 配置的沙箱和审批，高危）。
- **"在本机启动"只对同机房主开放**：需同时满足回环/本机网卡地址、无反向代理头、Host 为 localhost/本机 IP（防 DNS rebinding）、Origin 属本机（防 CSRF）、带 Server 每次启动生成的房主凭证。**Join token 不能用来启动进程**。
- Join token 通过**环境变量**传给子进程，不出现在进程参数里；以 realpath 存储工作目录。
- Server **不接受 Shell 命令**，远端成员无法让 Server 执行任何程序；本机房主也只能用固定参数拉起 `hive join`。
- 桌面端才有内置终端；浏览器/局域网打开的网页碰不到。

### 3.7 核心机制六：防循环 + 多维预算 + 故障自愈

- **多维预算**：hop 深度、人类缺席时 Agent 连续轮数、每对 Agent 每日次数；被拦下的调用状态为 `blocked`，人可 `budget.extend` 放行或单条 `invocation.retry`。
- **Prompt 节流与清洗**：新会话或每 8 轮/30 分钟才附成员列表/白板/完整规则，其余只发增量；进入 Prompt 的聊天内容去掉控制字符和 bidi 字符，中和伪造的 `<trigger_message>` 标签。
- **故障单（incident）系统**：失败先分"外部原因（模型/凭证/离线，只计数）/ 我们的问题（Bridge 与 Server 对不上账、Server 5xx、协议校验失败）/ 说不清"。我们的问题到阈值后可**自动派修**给某个修复 Agent（先写复现测试再修、提交信息加 `Incident: <短号>`、用 `hive_incident_report` 回报后交 review），修复经验入库，`git diff` 核对是否动了受保护路径。这是一个相当"认真"的生产级工程闭环。
- **本机 Agent 管家（LocalAgentSupervisor）**：Server 换代重启时旧 Server `detach()` 只停止跟踪不结束进程，新 Server 接管，本机 Agent 不中断；崩溃按指数退避重启，10 分钟内 5 次熔断。

### 3.8 核心机制七：IM 接入（飞书 / 微信）

- **飞书**：用 `@larksuiteoapi/node-sdk` 扫码接入，推送为消息卡片（回复/提问/失败/待批准/合并推送各有配色），可点选作答、原地更新。
- **个人微信**：经 `@wechatbot/wechatbot`（腾讯 iLink 通道），仅支持本人在私聊发文字，默认不启用（可能违反平台规则，文档明确提示风险）。
- IM 里支持 `/to`（选对话对象）、`/room`、`/level`、`/scope`、`/web` 等命令，实现"手机上也能派活、收进度"。

---

## 4. 在什么场景下使用

### 4.1 典型场景

| 场景 | 用法 |
| --- | --- |
| **小团队"AI 结对"** | 团队部署一个 Hive Server（局域网/云服务器），每人把本机 Claude Code/Codex 接进同一个房间，共享需求与进度，`@frontend-agent`、`@backend-agent` 分工 |
| **多 Agent 流水线** | 房间设主 Agent 统筹，前端 Agent 改 UI、后端 Agent 加接口、review Agent 审查，Agent 之间用 `hive_invoke` 自动委派、用白板登记进度 |
| **移动跟进长任务** | 连飞书后，Agent 跑长任务时把进度卡片推到手机私聊，提问直接在飞书里点选作答 |
| **远程/异地协作** | 云服务器 + HTTPS 部署，团队成员和手机用网址访问；房主在别的设备扫码登录管理 |
| **"我让我的 Agent 帮别人干活"但不给全权** | 访客 @ 房主 Agent 时默认要房主批准；访客可带自己的 Agent 进来，权限按档位限制 |
| **个人开发者自用** | `npm i -g superhive && hive`，本机起一个房间，把自己几个 Agent 和一个浏览器界面缝起来 |

### 4.2 不太适合的场景

- 需要**云端托管 Agent、多人共享一个账户**的 SaaS 形态（它刻意让 Agent 留在本机）。
- 需要**强账户体系/多租户隔离**的企业级权限（它只有共享 Join Token + 房主/访客）。
- 完全不懂 CLI 的用户（需先自行安装并登录 Codex/Claude Code 等）。

---

## 5. 如何使用

### 5.1 安装与启动（无需克隆仓库）

```bash
# 1. 全局安装（每台电脑一次，装好就有 hive 命令）
npm i -g superhive

# 2. 运行（启动 Server 并打开网页）
hive
```

启动后终端打印 Web UI 地址（如 `http://192.168.1.20:8787`），本机默认打开 `http://127.0.0.1:8787`，即房主身份。数据/消息/token/日志保存在 `~/.superhive/`。

常用命令：

```bash
hive stop                    # 停止（也可 hive off）
hive restart                 # 沿用原配置重启
hive update                  # 更新到最新版（运行中自动重启）
hive autostart on            # 登录后自动启动
hive --skill                 # 打印随包发布的 SKILL.md
```

### 5.2 浏览器加入

1. 打开终端显示的地址 → 输入用户名 → 选/建房间 → 聊天，`@agent-name` 邀请指定 Agent。

### 5.3 接入第一个 Agent

在右侧成员列表下方点「接入 Agent」→ 选 Provider → 选权限档（默认工作区档）→ 二选一：
- **在本机启动**（Server 就在本机，后台拉起，重启自动恢复）；
- **复制命令**（到另一台电脑终端运行）：

```bash
npx --yes superhive@latest join \
  --provider claude \
  --access read-only \
  --server http://192.168.1.20:8787 \
  --name my-agent \
  --token YOUR_JOIN_TOKEN
```

然后在房间发 `@my-agent 你好` 验证。

### 5.4 从源码运行（开发者）

```bash
pnpm install
pnpm dev        # 启动 Hive Server(:8787) + Vite Web(:5188)
pnpm lint && pnpm test && pnpm build
```

CLI 关键参数：

```text
hive start   --host --port --data-dir --public-url --token --no-local-agents --foreground --no-open
hive join    --server --room --name --token --provider --access --cwd --persona --model ...
```

常用环境变量：`HIVE_PORT`、`HIVE_DATABASE`、`HIVE_TOKEN`、`HIVE_PUBLIC_URL`、`HIVE_LOCAL_AGENTS=0`、`HIVE_SANDBOX=1`（沙箱预览：不连 IM、不起本机 Agent、不跑定时、不建删任务目录）等。

### 5.5 桌面端

`apps/desktop` 是 Electron 桌面端：双击打开即在本机起 Hive，窗口里是同一个网页 + 一个内置终端（默认在当前房间工作区目录）。macOS 可用，Windows/Linux 适配中。

### 5.6 许可与商业化注意

AGPL-3.0-only。个人/团队自部署、修改、内部使用免费；**把 SuperHive 或其修改版作为网络服务提供给他人，或嵌入商业产品，需遵循 AGPL 开源义务或获取商业授权**。作者保留商业授权权，因此合入外部贡献前需先补 CLA 或 DCO 式声明。

---

## 6. 架构与模块地图（源码级）

| 模块 | 职责 | 关键文件 |
| --- | --- | --- |
| `apps/server` | Hive Server：Fastify + WebSocket + SQLite + MCP + IM | `app.ts`（路由/WS/MCP）、`hive-mcp.ts`（工具）、`local-agents.ts`（管家）、`incidents*.ts`（故障单）、`im/`（飞书微信）、`agent-instructions.ts`（规则权威源） |
| `packages/superhive` | CLI + Bridge + Provider 适配 | `cli.ts`、`bridge.ts`（核心循环）、`codex-*/claude-*/opencode-*/omp-*/copilot-*/gemini-*/agy-*/openclaw-client.ts`、`hive-protocol.ts`、`state-store.ts`、`prompt-builder.ts` |
| `packages/protocol` | 共享类型 + zod + LIMITS + 服务端文案 | `server-text.ts` 等 |
| `apps/web` | React + Vite 网页 | `App.tsx`、`chat-header`、`activity`、`artifacts`、`terminal-session`、i18n |
| `skills/superhive` | 给 Agent 的协作说明 | `SKILL.md`（"房间规则"一节由服务端生成并校验一致） |
| `scripts` | 开发 supervisor、沙箱预览、截图 | `dev-supervisor.mjs`、`preview-sandbox.mjs`、`screenshot.mjs` |

**工程细节亮点**：

- **i18n 严格**：中文原文作 key，`coverage.test.ts` 扫描源码校验漏翻、占位符一致。
- **沙箱预览**：`preview:sandbox` 起只监听 127.0.0.1 的沙箱 Server，剔除 IM/凭据/token/访客，服务端还有 `HIVE_SANDBOX=1` 硬开关兜底，配合 `screenshot.mjs` 做 UI 前后对比。
- **Git 钩子**：`pnpm install` 设 `.githooks`，提交前拒绝 `node_modules` 路径。
- **文档即产品**：`development.md` 详尽到安全边界、故障分类表、i18n 术语表、状态词表——这种文档密度在开源项目里很少见。

---

## 7. 设计哲学与可借鉴之处

1. **"控制面 / 数据面分离"**：Server 只管公共消息与协调，Agent 私有状态（会话、文件、权限）留在本机。这既解决了"团队共享上下文"，又避免了"把所有人的机器和密钥集中到一个 Server"的安全噩梦。
2. **幂等 + 游标 + 意图日志**：把"分布式消息系统的可靠性套路"（seq 严格递增、afterSeq 补拉、clientMessageId 去重、先写意图再执行）用在 Agent 编排上，非常务实。
3. **状态机作为 truth**：invocation 是权威状态，桥端只是上报；"结果未知 ≠ 失败"的区分避免了盲目重跑带来的副作用重复。
4. **失败分类驱动策略**：外部原因只计数、自己的问题才派修——不是所有错误都该改代码，这个判断很成熟。
5. **MCP 作为"Agent 与外部世界"的天然边界**：用标准协议把工具暴露给 Agent，兼容多家 CLI，而不是自己造一套 Agent 运行时。
6. **权限"默认最小 + 逐次确认"**：read-only 默认、workspace 需显式 cwd、full 每次确认、"在本机启动"严格同机校验。

---

## 8. 快速结论

**它是什么**：一个开源自托管的"多人 + 多 AI 编程 Agent"共享协作房间（Server + Bridge + Web/IM/Desktop）。

**解决什么问题**：AI 编程 Agent 在团队/多 Agent 场景下的"孤岛、无共享上下文、协调靠人肉、状态不可见、安全边界撕裂、移动端缺位"。

**怎么解决**：公共消息总线（Fastify+WS+SQLite）+ 严格 `@mention`/主 Agent 路由 + invocation 状态机与幂等游标保证可靠 + `/mcp` 暴露 `hive_*` 工具（委派/白板/提问/定时/附件）+ 多 Provider 适配把各家 CLI 统一 + 权限留在本机 + 故障自愈与多维预算。

**什么场景用**：小团队 AI 结对、多 Agent 流水线分工、长任务移动跟进（飞书/微信）、异地/云部署协作、"让我的 Agent 帮别人干活但不给全权"。

**怎么用**：`npm i -g superhive && hive` 起服务 → 浏览器加入 → 网页/命令行接入你已登录的 Codex/Claude Code 等 → `@agent-name` 派活。开发则 `pnpm install && pnpm dev`。

**风险/注意**：AGPL-3.0（商用需授权）、Join Token 是共享令牌（公网需反代+HTTPS+访问控制）、个人微信通道有封号风险、依赖用户已装好并登录各家 Agent CLI。

---

## 附：参考来源

- GitHub 仓库：https://github.com/wangrongding/SuperHive
- 文档站：https://wangrongding.github.io/SuperHive/
- 中文 README：[README.zh-CN.md](https://github.com/wangrongding/SuperHive/blob/main/README.zh-CN.md)
- 开发者文档：[docs/development.md](https://github.com/wangrongding/SuperHive/blob/main/docs/development.md)
- 日常使用：[docs/usage.md](https://github.com/wangrongding/SuperHive/blob/main/docs/usage.md)
- Agent 协作说明：[skills/superhive/SKILL.md](https://github.com/wangrongding/SuperHive/blob/main/skills/superhive/SKILL.md)
- GitHub API 元数据：https://api.github.com/repos/wangrongding/SuperHive
