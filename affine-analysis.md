# AFFiNE 深度分析报告

> 仓库：https://github.com/toeverything/AFFiNE
> 分析版本：`v2026.11.18-canary.905`（monorepo 版本 `0.27.5`）
> 分析日期：2026-11-20
> 官网：https://affine.pro ｜ 文档：https://docs.affine.pro

---

## 1. 一句话概述

AFFiNE（读音 [əˈfaɪn]，取 "affine" 之意）是一个 **本地优先（local-first）、隐私优先的开源一体化知识工作台**：它把 **文档（Notion 式块编辑器）+ 无限白板（Miro 式无边画布）+ 多视图数据库（Airtable/Trello 式表格看板）+ AI 助手** 超融合（hyper-fused）在同一个应用里。官方口号是 *"Write, Draw and Plan All at Once"*——"There can be more than Notion and Miro"。

截至分析时，GitHub **约 75.2k Stars / 5.6k Forks**，主语言 TypeScript（约 70 万行，不含测试）+ Rust（约 11 万行，本地存储与原生绑定），2022 年 7 月开源，由新加坡公司 TOEVERYTHING PTE. LTD.（AFFiNE 团队，源自中国团队）持续维护，当前提交频率为日级（canary 版本每天多次发布）。

---

## 2. 基本信息

| 项目 | 内容 |
| --- | --- |
| 仓库 | `toeverything/AFFiNE`（monorepo，124 个 npm 包） |
| 技术栈 | TypeScript + React（前端）、NestJS + GraphQL + Prisma（服务端）、Rust（原生存储）、Electron（桌面端）、yjs/CRDT（协同） |
| 许可证 | **混合许可**：客户端与 BlockSuite 主体为 MIT；`packages/backend/server` 为 AFFiNE Enterprise Edition (EE) 许可（生产商用需订阅，开发测试免费） |
| 部署形态 | 云端 SaaS（app.affine.pro）、桌面客户端（macOS/Windows/Linux）、iOS/Android、Web、Docker 自托管 |
| 相关子项目 | BlockSuite（编辑器框架，已并入本仓库 `blocksuite/` 目录）、AFFiNE AI（付费增值）、AFFiNE Cloud |

---

## 3. 它解决什么问题

### 3.1 用户侧痛点：工具割裂与数据主权

现代知识工作中，同一个想法往往要跨越多个工具流转：

- 在 **Notion** 里写文档 → 想画张架构图要切到 **Miro/FigJam** → 排期管理又要切到 **Trello/Airtable** → AI 总结再切到 ChatGPT；
- 这些工具互不相通，内容在复制粘贴中变形、失联，"原子化的 building blocks"（段落、卡片、任务、便签）在每个应用里都是重复的孤岛；
- 云 SaaS 工具的数据存在厂商服务器上，**用户失去数据所有权**：隐私顾虑、服务关停风险、离线不可用、被锁定在私有格式里；
- Notion / Miro 等均不开源，无法自托管、无法定制、无法审计。

### 3.2 开发者侧痛点：缺少可复用的"块编辑"基础设施

做一个 Notion/Miro 级别的编辑器极难：块级数据模型、多人实时协同、无限画布渲染、数据库多视图……这些能力没有像 UI 组件库那样成熟的**开源基础框架**可复用。AFFiNE 团队为此把核心能力沉淀为独立的 **BlockSuite** 框架（编辑器框架）与本地优先存储层，让第三方也能构建自己的协同块应用。

### 3.3 归纳：三大问题

1. **融合问题**：文档、白板、数据库三种范式能否合并为"同一份数据的两种视图"，而非三个工具？
2. **主权问题**：数据能否默认存在本地磁盘（离线可用、格式开放、随时自托管），云端只是可选的同步层？
3. **开放问题**：核心编辑能力能否开源（MIT），让社区可 fork、可扩展、可二次开发？

---

## 4. 它是如何解决的（核心架构与原理）

### 4.1 总体分层架构

AFFiNE 是一个典型的大型 monorepo，自上而下分为四层（对应仓库目录）：

```
┌─────────────────────────────────────────────────────────┐
│ 应用层 packages/frontend/apps/*                          │
│   web / electron(桌面) / ios / android / mobile-shared    │
├─────────────────────────────────────────────────────────┤
│ 应用核心 packages/frontend/core（约 70+ 业务模块）         │
│   workspace-engine(本地/云工作区)、editor、navigation、    │
│   db、comment、share-doc、template、blocksuite/ai(Copilot)│
├─────────────────────────────────────────────────────────┤
│ 编辑器框架 blocksuite/（BlockSuite，MIT 独立可用）         │
│   framework/store(块树+yjs)、framework/sync(同步)、       │
│   framework/std(编辑器标准)、gfx(图形引擎)、model、        │
│   blocks(段落/数据库/图像/画布等 20+ 块类型)、             │
│   inlines(富文本)、widgets、fragments(大纲/框架面板)       │
├─────────────────────────────────────────────────────────┤
│ 存储与协同层                                              │
│   客户端: packages/common/nbstore(SQLite/IndexedDB/      │
│           广播通道)+ framework/sync(yjs CRDT)             │
│   服务端: packages/backend/server(NestJS+GraphQL+Prisma) │
│           + Postgres(pgvector) + Redis + S3 + socket.io  │
└─────────────────────────────────────────────────────────┘
```

各层职责单一、可独立复用：BlockSuite 可以脱离 AFFiNE 用于任何块编辑应用（它有自己的文档站与 playground）；nbstore 是通用本地存储抽象；服务端只承担"同步、鉴权、协作、搜索、AI"等云能力。

### 4.2 关键设计一：一切皆 Block，"同一个数据的两种视图"

这是 AFFiNE 与 Notion（只有文档）、Miro（只有画布）最本质的差异。实现要点：

- **块树模型**（`blocksuite/framework/store`）：每个文档（Doc）是一棵 block 树，根节点固定为 `affine:page`（见 `blocksuite/affine/model/src/blocks/root/root-block-model.ts` 的 `defineBlockSchema({ flavour: 'affine:page', role: 'root' })`）；
- **页面内容**挂在 `affine:note` 块下（段落、列表、代码、图片、数据库等 20+ 种块类型，见 `blocksuite/affine/blocks/`）；
- **画布内容**存放在 `affine:surface` 块（`blocks/` 下的 `surface` 与 `gfx/` 图形引擎负责）：形状、连线、frame、便签、嵌入网页等"无边画布元素"；
- 因此 **"页面模式（Page View）"与"白板模式（Edgeless View）"只是同一棵 block 树的两个渲染器**：一段文字在页面里是段落，切到画布上就是可以拖拽排版的便签；数据库块（`data-view` 包支持表格/看板等多种视图）既能嵌在文档里，也能整块放上画布。模式切换不需要数据转换，也不产生副本。

这直接回答了 3.1 的"融合问题"：不是做三个工具再打通，而是**从数据模型层面就让它们是同一个东西**。

### 4.3 关键设计二：本地优先 + CRDT 实时协同

AFFiNE 的存储模型是"本地数据库为主、云端为可选同步层"：

- **本地数据**：文档状态以 **yjs（CRDT，冲突可合并复制类型）** 持久化。Web 端写 IndexedDB，桌面端通过 **Rust 原生模块（`packages/frontend/native/nbstore`，约 11 万行 Rust 中的核心）** 写入本地 SQLite；二进制资源（图片/附件）存本地 blob 存储（`nbstore/src/impls/` 下有 `sqlite / idb / disk / broadcast-channel / cloud` 等实现）；
- **多端同步**：`framework/sync` 提供 doc/blob/awareness 三类同步通道；本地写完后后台增量同步，断网照常编辑，恢复后 CRDT 自动合并，无冲突弹窗；
- **多人实时协同**：云端开启的 workspace 通过服务端 `core/sync/gateway.ts`（socket.io 网关）广播 yjs 更新与 awareness（光标/选区）；本地 workspace 则用浏览器 BroadcastChannel 在多标签页间实时同步；
- **数据所有权**：因为数据格式是开放的 CRDT 快照 + 本地文件，用户随时可以把云端 workspace 导出为本地，或完全只用本地模式（不需要登录任何账号）。

这回答了"主权问题"：**默认数据在本地，云是增强而非依赖**——与 Notion（纯云）形成根本区别。

### 4.4 关键设计三：可自托管的服务端（协同/搜索/AI）

`packages/backend/server` 是一个标准的企业级 NestJS 服务（GraphQL API 见 `schema.gql`，数据模型见 `schema.prisma`），核心模块：

| 模块 | 作用 |
| --- | --- |
| `core/sync` + `core/realtime` | socket.io 网关，承载 yjs 增量同步与实时协作 |
| `core/doc` / `core/workspaces` / `core/permission` | 文档、工作区、成员权限管理 |
| `core/storage`（S3 兼容） | blob 二进制存储（自托管可用本地/MinIO 类方案） |
| `core/indexer` + Postgres **pgvector** | 全文/向量检索，为搜索与 AI 嵌入提供支撑 |
| `plugins/copilot` | AFFiNE AI 服务端（对话、幻灯片生成、思维导图、原型代码等，按订阅付费） |
| `plugins/oauth / payment / license / quota / features` | 登录、支付、许可与配额门控（付费墙逻辑独立成 `paywall` 模块） |

**自托管只需三条命令**（`.docker/selfhost/compose.yml`）：

```bash
docker compose -f .docker/selfhost/compose.yml up -d
# 编排了三个服务：
#   affine_server  ghcr.io/toeverything/affine:stable (端口 3010)
#   postgres       pgvector/pgvector:pg16（文档 + 向量索引）
#   redis          会话/队列/广播
# 先跑 affine_migration 容器执行 self-host-predeploy.js 迁移脚本
```

官方也提供 `config.example.json` / JSON Schema 用于配置 SMTP、OAuth、存储等，社区版免费，生产规模化商用需 EE 订阅。

### 4.5 关键设计四：AI Copilot 深度嵌入编辑器

AI 不是外挂聊天框，而是长在 BlockSuite 编辑器里的能力（`packages/frontend/core/src/blocksuite/ai/`）：

- **动作式 AI**：选中文本/画布元素后调用改写、翻译、总结、续写等 action（`actions/`）；
- **结构化生成**：大纲 → 幻灯片（`slides/`）、文章 → 思维导图（`mini-mindmap/`）、一句话 → 可运行的原型页面；
- **双模式入口**：聊天面板（chat-panel）与画布/文档上下文菜单（`entries/edgeless`、`entries/...`）双入口；
- 服务端 `plugins/copilot` 负责模型编排，配合 pgvector 的 workspace 索引实现"基于你的知识库回答"。

### 4.6 工程质量与生态

- **严格的工程化**：Node 22 + Yarn 4 + oxlint/oxfmt + Vitest + changesets 式 canary 发版（每日 canary、稳定版按月节奏）；PR 必须 Conventional Commits + CLA；
- **代码规模**：约 70 万行 TS（不含测试）+ 11 万行 Rust，124 个 npm 包，模块边界清晰（frontend/backend/common/blocksuite 四大 workspace）；
- **生态双仓战略**：BlockSuite（编辑器框架，MIT）既服务 AFFiNE，也作为独立开源框架供第三方构建块应用，形成"基础设施 + 应用"的双层开源策略（同门项目还有 OctoBase 历史仓库，现已演进为仓库内的 Rust nbstore）。

---

## 5. 在什么场景下使用（适用人群与典型用法）

### 5.1 个人知识管理

- **本地优先笔记/日记**：不想把隐私笔记放到云端的用户；journal 模块内置每日笔记；
- **模板生态**：官方模板库提供视觉板、数字规划器、ADHD 规划器、康奈尔笔记、读书清单等；
- **全格式导入**：支持 Notion/Markdown/HTML/Notion 导出包迁移（`modules/import`、`import-clipper` 浏览器剪藏）。

### 5.2 团队协作

- **头脑风暴 → 文档 → 排期一条龙**：白板里贴便签讨论 → 一键转成结构化文档 → 文档中嵌数据库排任务（表格/看板视图），全程不换工具；
- **实时多人协同** + 评论 + 分享链接（`share-doc` / `comment` 模块），自托管后数据完全在自己服务器；
- **中小团队自建 Wiki/知识库**：Docker 一键部署，替代"Notion + Miro + Airtable"三件套，兼顾私有化合规要求。

### 5.3 开发者与二次开发

- **基于 BlockSuite 构建自己的块编辑器/协同应用**（框架层 MIT 独立可用，含 playground 与类型完备的扩展 API）；
- **fork AFFiNE 做行业定制**：改块类型、接自有 AI 模型、嵌入既有系统；
- **桌面端本地 SQLite**：需要"数据必须在本地"的场景（合规、离线工厂、内网环境）。

### 5.4 不适合的场景（客观说明）

- 需要成熟的企业权限体系（SSO/审计/细粒度权限矩阵）时，开源版功能有限（属 EE 付费能力）；
- 对移动端体验要求很高的场景（移动端仍较新）；
- 已深度绑定 Notion 数据库公式生态且不愿迁移的团队。

---

## 6. 如何使用

### 6.1 普通用户（推荐路径）

```bash
# 方式一：官网下载桌面客户端（macOS / Windows / Linux）
#   https://affine.pro/download
# 方式二：直接使用云端版（免安装）
#   https://app.affine.pro
```

首次进入默认创建 **本地 workspace**（数据在本地）；如需跨设备同步/多人协作，登录 AFFiNE Cloud 并把 workspace 切换为云 workspace。核心操作：左上角新建 Doc（页面模式 ⇄ Edgeless 白板模式一键切换），`/` 呼出块菜单插入段落、数据库、图片、代码块等；画布模式下可放任意块并自由排版；右上角 Share/Publish 分享。

### 6.2 自托管（团队私有部署）

```bash
mkdir affine && cd affine
# 从仓库获取 .docker/selfhost/compose.yml 与 config
docker compose up -d          # 启动 affine + postgres(pgvector) + redis
# 访问 http://<host>:3010 完成管理员初始化
```

进阶配置（`config.json`）：SMTP 邮件、GitHub/Google OAuth、S3 存储、AI 凭据等；升级只需拉新镜像并跑 predeploy 迁移容器。

### 6.3 开发者（构建/贡献）

```bash
git clone https://github.com/toeverything/AFFiNE.git
cd AFFiNE && yarn            # 需要 Node 22(.nvmrc) + Rust 工具链；postinstall 自动初始化
yarn dev                     # Web 开发服务器
yarn affine @affine/server-native build && yarn dev   # 本地起后端（GraphQL :3000/graphql）
yarn build && yarn test      # 构建 / Vitest 测试；桌面端另见 docs/building-desktop-client-app.md
```

贡献流程：fork → 从 `canary` 拉分支 → Conventional Commits → PR（需签 CLA）。

---

## 7. 竞品对比

| 维度 | AFFiNE | Notion | Miro | AppFlowy | 思源笔记 SiYuan |
| --- | --- | --- | --- | --- | --- |
| 开源 | 客户端 MIT / 服务端 EE 源码可得 | 否 | 否 | AGPL | AGPL |
| 文档 | ✅ 块编辑器 | ✅ 标杆 | 弱 | ✅ | ✅ |
| 无限白板 | ✅ 与文档同源 | ❌（外挂 Figma 式） | ✅ 标杆 | 有限 | ❌ |
| 数据库/看板 | ✅ 多视图 | ✅ | ❌ | 有限 | ❌ |
| 本地优先 | ✅ 默认本地 | ❌ 纯云 | ❌ 纯云 | ✅ | ✅ |
| 自托管 | ✅ Docker | ❌（企业版贵） | ❌ | ✅ | ✅ |
| 实时协同 | ✅ CRDT | ✅ | ✅ | 弱 | 有限 |
| 内置 AI | ✅（订阅） | ✅（订阅） | ✅（订阅） | 接外部 | 插件 |
| 桌面离线 | ✅ | 弱 | ❌ | ✅ | ✅ |

定位总结：**AFFiNE 是唯一同时把"文档+白板+数据库+本地优先+自托管+开源"做齐的产品**；单点能力上 Notion 的生态/打磨、Miro 的画布手感仍领先，但 AFFiNE 的差异化（数据主权 + 模式融合 + 可编程的 BlockSuite）是明确的护城河。

---

## 8. 商业模式与许可证解读

- **社区版（CE）**：客户端全功能 MIT 开源，自托管服务端免费但受 EE 许可约束（开发测试自由、生产商用需订阅席位、禁止转售/再分发服务端）——属于典型的 **Fair-Source / open-core** 策略；
- **付费点**：AFFiNE AI（Copilot 订阅）、云协作席位与存储配额、自托管企业功能（SSO、审计等，license 插件门控）；
- **对个人用户**：本地功能完全免费且无账号要求，是当前最"无套路"的用法。

---

## 9. 总结：优点与风险

**优点**

1. 数据模型层面的三合一（doc = note + surface），页面与画布真正共享同一份数据，这是架构级而非功能级的创新；
2. 本地优先 + CRDT（yjs）+ 开放格式，数据主权与离线能力做得扎实；
3. BlockSuite 框架独立开源，编辑器基础设施可复用，社区想象力大；
4. 工程质量高：NestJS/Rust/React 分层清晰、测试与 CI 完备、发版节奏快；
5. 自托管体验好：Docker Compose 三服务开箱即用。

**风险与短板**

1. 混合许可使"完全开源替代品"的叙事打折扣：服务端商用受限，重度依赖云端 AI/协同功能时仍被锁定在官方服务；
2. 体量大（80 万+ 行代码），外部贡献者入门成本高，核心创新集中在 BlockSuite，应用层迭代由官方主导；
3. 移动端、企业权限等外围能力仍在追赶 Notion/Miro 的成熟度；
4. AI 能力依赖订阅与官方服务端，自托管社区版 AI 配置门槛较高。

**一句话结论**：AFFiNE 用"块级统一数据模型 + CRDT 本地优先 + 开源框架（BlockSuite）"三层架构，正面回答了"知识工具能否既融合又开放"这个问题——对个人是免费强大的本地知识库，对团队是可私有部署的 Notion+Miro 平替，对开发者是一套可编程的协同编辑基础设施。

---

*分析方法：源码浅克隆（commit `dc2b221`）+ 目录/关键源文件走读（root-block-model.ts、nbstore、sync gateway、compose.yml、LICENSE 等）+ 官网/README/社区资料交叉验证。*
