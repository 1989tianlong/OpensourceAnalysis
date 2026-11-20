# App Store Compliance Playbook 深度分析报告

> 仓库：https://github.com/mjmirza/app-store-compliance
> 分析版本：`v2.4.0`（2026-11-15 发布，commit `d2156ba`）
> 分析日期：2026-11-20
> 主语言：Shell / Python / Bash，约 1.9 万行脚本 + 30+ 篇文档
> 社区热度：⭐ 1,877 stars / 126 forks（截至分析日）
> 许可证：OpenRoots Agent License 2.3（特殊许可，商用注意，见第 9 节）

---

## 1. 一句话概述

App Store Compliance Playbook 是一个**面向 iOS / Android 应用上架审核的开源合规手册 + 自动化拦截器**：它把 Apple App Store 和 Google Play 上一百多种常见拒审（rejection）模式整理成机器可读数据，并用一个 Bash "守卫（Guard）"脚本挂进 Claude Code 等 13 种 AI 编码工具，在你执行 `fastlane`、`eas submit`、`xcrun altool` 等提交命令的**前一秒**扫描项目，发现致命拒审风险就直接**阻断上传**并给出条款编号和修复方法——把"提交 → 等几天 → 被拒 → 猜原因 → 再提交"的恶性循环，变成提交前的一次可设计的检查。

作者的定位标语：*"Stop getting your iOS and Android apps rejected. Turn App Store and Google Play rejection into a designed-out failure mode."*（别再被拒了，把拒审变成一种被设计掉的失败模式。）

---

## 2. 仓库基本信息

| 项 | 内容 |
| --- | --- |
| 作者 | mjmirza（Mirza Iqbal），独立开发者单人维护 |
| 创建时间 | 2026-06，迭代非常活跃（截至分析日已到 v2.4.0，2026-11-19 仍在推送） |
| 形态 | 纯文档 + 数据 + 脚本的"知识库仓库"，无需要编译的产物；安装即用 |
| 在线阅读 | https://mjmirza.github.io/app-store-compliance/（GitHub Pages，附 llms.txt 供 AI 抓取） |
| 核心交付物 | ① 123 条拒审模式数据 ② PreToolUse 守卫脚本 ③ Agent Skill + 安装器 ④ 一套监控/审计脚本 ⑤ 107 条全球监管截止日期数据库 |
| CI | 每次 push/PR 运行全部自测：数据校验、guard 测试、安装器测试、文档链接检查、引用校验、评估记分卡等十余个套件 |

---

## 3. 它解决什么问题

### 3.1 痛点：上架审核是一个昂贵的"黑盒对抗测试"

一个 App 提交被拒的真实代价链是：

```
提交 → 等待数天审核 → 收到一句模糊拒信 → 猜测原因修改 → 再提交 → 再等几天 → 可能再被拒
```

- 一次拒绝浪费一周是常态；Google 侧 careless resubmit（原样重提）还会把账号推向**封号**。
- Apple 每年审核约 777 万次提交，拒绝约 193 万次，**接近每 4 次就有 1 次被拒**；Google 2025 年拦截了 175 万+ Play 提交。
- 拒绝原因高度集中：审核备注里没有可用的演示账号 + 后端没开、隐私声明（Privacy Manifest / Data Safety 表）与运行时行为不符、权限申请无对应功能、数字商品绕过官方支付…… 这些几乎都是**提交前就能发现**的可预防问题。

### 3.2 时代背景：AI 编码让提交量暴增，但规则反而收紧

这是该仓库"为什么是现在（2026）"的核心论点：

- AI 编码工具（Claude Code 等）让"一个下午上架一个 App"成为现实，2026 Q1 双商店发布量同比 +60%（仅 iOS 约 +80%），单期新增提交近 60 万。
- 商店没有放水，而是**收紧了规则**：隐私清单（Privacy Manifest）强制执行、向外部 AI 发送用户数据必须弹同意框（2025 年底起）、2026 新版年龄分级问卷、EU AI Act 透明度义务等。
- 结果是大量 AI 快速拼出来的 App "死在见用户之前"——而用 AI 造 App 的人，往往根本没听说过 Privacy Manifest 这类规则。

### 3.3 问题归纳

该仓库要解决的是三个具体问题：

1. **信息不对称**：审核规则散落在 Apple/Google 文档、新闻、监管法规里，开发者不会主动读完；被拒时只拿到一句话，无从定位。
2. **时机太晚**：合规检查发生在审核方手里，而此时修复成本最高（已等待数天、账号风险已产生）。
3. **执行不落地**：即使有 checklist，人也会忘；需要把检查自动挂到"提交"这个动作之前，让 AI 助手"拒绝帮你发布一个会被拒的 App"。

作者的第一性原理表述得很精辟：**把商店审核员当作一个对抗性集成测试——它只在真机上跑一次、读不到你的意图、对配置缺失零耐心。审核员需要的一切，必须在提交那一刻就存在、可用、且显而易见。**

---

## 4. 它是如何解决的（核心原理）

### 4.1 整体架构：知识层 + 数据层 + 执行层 + 防幻觉质检层

```
┌─────────────────────────────────────────────────────────────────┐
│ 知识层  docs/（30+ 篇人工策展文档）                                 │
│   APPLE.md / GOOGLE-PLAY.md   拒审地图：每条 guideline → 触发条件 → 修复 │
│   EU/GLOBAL/PLATFORM-2026     法规硬规则（带日期与来源）               │
│   MISTAKE-PATTERNS.md         八大根因 + 申诉手册                     │
│   PRE-SUBMISSION-CHECKLIST.md 逐项可验证的提交前清单                   │
│   BY-APP-TYPE.md              按应用类型路由（订阅/社交/儿童/健康/AI/VPN…）│
├─────────────────────────────────────────────────────────────────┤
│ 数据层  data/（机器可读，驱动一切自动化）                             │
│   rejection-patterns.json     123 条拒审模式（severity/detection/fix）│
│   detection-recipes.json      107 条模式对应的自动检测命令             │
│   regulatory-deadlines.json   107 条全球监管截止日期（含 source_url）  │
│   ruling-register.json        14 条代码扫不出来的发布要求（法律依据）    │
├─────────────────────────────────────────────────────────────────┤
│ 执行层                                                            │
│   guard.sh（1752 行 Bash）    PreToolUse 守卫：提交前扫描，critical 即阻断│
│   install.sh + install_harness.py  一条命令装进 13 种 AI 编码工具        │
│   metadata-audit / release-audit / fleet-sweep / monitor 系列 /       │
│   accessibility-audit / deadline-checker / generate-timeline          │
│   agent-os/skill/SKILL.md     五步审计 Agent Skill + /app-store-audit  │
├─────────────────────────────────────────────────────────────────┤
│ 防幻觉质检层（这个仓库的独特之处）                                    │
│   verify-citations.py         机器验证 ~300 条引用 URL，专抓"软 404"    │
│   validate.py + CI            数据一致性与全量测试                     │
│   AGENTS.md 源信任层级         官方监管 > 权威媒体 > 学术 > 行业 > 社媒   │
│   每条 deadline 强制 source_url + verified_on 溯源字段                │
└─────────────────────────────────────────────────────────────────┘
```

`references/` 目录是由 `generate-references.py` 从数据层**自动生成**的 AI 可加载参考树（9 类规则 + 10 种应用类型），保证文档永远不会与数据漂移。

### 4.2 数据层：123 条拒审模式的分类学

`data/rejection-patterns.json` 是整个系统的"单一事实来源"，每条模式包含 `id / platform / guideline / title / severity / detection / signals / fix` 六要素：

- **平台分布**：Apple 61 条、Google Play 43 条、双平台 13 条、Web 6 条
- **严重度分布**：critical 38 条、high 65 条、medium 20 条
- critical 的典型条目直接对应最高频拒因：`APPLE-2.1-MISSING-DEMO-ACCOUNT`（缺演示账号）、`APPLE-2.1-STAGING-BACKEND`（后端是测试环境）、`APPLE-5.1.1-MISSING-USAGE-DESCRIPTION`（权限用途字符串缺失）、`APPLE-5.1.1-NO-ACCOUNT-DELETION`（无账号删除）、`APPLE-3.1.1-EXTERNAL-PAYMENT`（绕过 IAP）、`GOOGLE-DATASAFETY-MISMATCH`（Data Safety 表与实际不符）等

每条模式在 `detection-recipes.json` 里配套一条具体检测命令（114 条模式中 107 条可自动检测，剩余 7 条是商店列表/账号侧只能人工核验的项）。

### 4.3 执行层核心：Guard 守卫脚本的工作原理

`agent-os/hooks/app-store-compliance-guard.sh`（1752 行纯 Bash）是整个仓库的旗舰，支持**双模式**运行：

**模式一：独立扫描**（无任何依赖安装）

```bash
bash agent-os/hooks/app-store-compliance-guard.sh "/path/to/your/app"
```

**模式二：AI 工具 Hook**（PreToolUse 拦截）

脚本头声明 `@event: PreToolUse` / `@matcher: Bash`。安装器把它注册进各 AI 工具的 hook 配置后，AI 每次 要执行 shell 命令时，工具会把命令以 JSON 打到 guard 的 stdin，工作流如下：

```
AI 工具要执行提交命令（JSON payload → stdin）
        │
        ▼
① 解析 payload：jq → python3 → 降级正则，三级提取 .tool_input.command
   （兼容 Claude Code / Codex / Gemini / Cursor / Copilot / Grok 等各自不同的 schema）
        │
        ▼
② 命令匹配：超大正则 SUBMIT_RE 识别"提交类命令"
   fastlane deliver/pilot/supply、eas submit/build、xcrun altool|notarytool、
   transporter、gradlew bundleRelease/assembleRelease、xcodebuild archive、
   flutter build ipa、expo submit、cap sync、ionic/cordova build、
   unity -buildTarget、dotnet publish -f net*-ios、tauri build、asc uploads…
   甚至自定义 fastlane lane（lane 名含 submit 字样也会识别）
        │
        ▼
③ 平台判定：原生 iOS/Android + Flutter / React Native / Expo /
   Ionic / Capacitor / Cordova / Unity / .NET MAUI / Tauri / KMP，
   Android-only 的命令不跑 iOS 检查（避免噪音）
        │
        ▼
④ 扫描项目：按 123 条模式的 signals 扫源码树（vendor 目录除外，
   多 app 仓库会识别出每个 app root 分别扫），
   内嵌 deadline-checker 顺带报告 90 天内的监管截止日期
        │
        ├── 存在 critical ──▶ exit 2，stderr 输出 "BLOCKED." + 按severity排序的
        │                    发现清单（含文件:行号、条款编号、fix.），提交被阻断
        ├── 干净 ──────────▶ "CLEAR."，命令放行（exit 0）
        └── 找不到项目 ─────▶ "NOT CHECKED."——"没扫到永远不算通过"
```

关键设计细节：

- **"未检查"≠"通过"**：路径不存在、payload 为空、解析失败时宁可沉默退出，绝不把"没读到"当作"没问题"。
- **可逃生**：开发者可用 `APP_STORE_GUARD_OK=1` 显式覆盖阻断（而不是逼人卸载守卫）。
- **保守优于漏报**：官方明确说明，无法从代码确认的项（如 App Store Connect 里的审核备注、声明的数据用途）也会被"提示出来"而非默认干净——"漏掉一条警告的代价是一次拒审，多弹出一条警告的代价只是一分钟"。一位 Reddit 用户最初把这当成误报，后来发现正是这些提示让他首提过审。

### 4.4 周边工具矩阵

| 工具 | 作用 |
| --- | --- |
| `install.sh` | 一条命令把 guard + skill + `/app-store-audit` 装进 Claude Code、Codex、Gemini CLI、OpenCode、Cursor、Copilot、Grok、Qwen Code、Kimi Code、Factory Droid、Auggie、Windsurf（`--tool all` 全装）；`doctor` 子命令会真的喂一条提交命令给 hook、看到 BLOCKED 才打印 `Install verified.`；`uninstall` 精确移除 |
| `metadata-audit.py` | 审计商店元数据（名称/副标题/关键词/描述/URL），多语言目录逐个审计、字符上限、跨平台提及、"coming soon"式宣传、缺隐私政策/订阅条款链接；`--propose` 给出修改建议形成"检测→建议→复验"循环。大量拒审发生在列表文案而非代码 |
| `release-audit.py` | 发布就绪总闸：先自检 playbook 再审你的 App + 元数据，写 `RELEASE-READINESS-REPORT.md`；exit 0 清晰 / 2 被 critical/high 阻断 / 3 未完成，适合接 CI/CD |
| `fleet-sweep.py` | 企业场景：一次扫所有在运营 App，配套 **Waiver Ledger**（有期限的误报豁免台账）与 GitHub code-scanning 导出 |
| `monitor.py` 等 8 个监控器 | 跟踪 25 条 Apple 要求轨道、监管动态（按源信任层级分类）、Android/AI 政策/隐私/安全要求；可 `--simulate` 预演某项新要求的影响、生成迁移任务与 PR 草稿 |
| `deadline-checker.py` | 时区感知地报告滚动 90 天内的全球监管截止日期（EU AI Act、DSA、COPPA、美国各州年龄法、中国算法备案等 107 条） |
| `accessibility-audit.py` | 无障碍静态审计（VoiceOver 标签、Dynamic Type、对比度、TalkBack、触摸目标），对应 EU 欧洲无障碍法/EN 301 549 |
| `eval-scorecard.py` + `evals/gold-cases.json` | 32 个金标准场景对 Fleet Sweep/Waiver/Ruling Register 做回归评分 |
| `templates/REVIEW-NOTES-TEMPLATE.md` | 填空式审核备注模板—— 六个部分能解决大多数 2.1 拒审 |

### 4.5 防幻觉质检层：这个仓库最值得借鉴的工程实践

合规内容最大的风险是"编造的条款编号"，该仓库用四道机制把"引用真实性"变成机器可验证的：

1. **`verify-citations.py` 反软 404 校验**：Apple 的新闻站对任意未知 article id 都返回 HTTP 200（字节相同的兜底页），所以"链接能打开"不等于"引用是真的"。校验器对每个这类 host 先抓取一个**故意伪造的 control URL**建立指纹基线，任何内容指纹与 control 匹配的引用直接判为伪造；同时把反爬 4xx、主机故障 5xx 与真伪造区分开，保持信号噪音比。一次校验约 300 条链接。
2. **强制溯源字段**：`regulatory-deadlines.json` 每条必须带 `source_url`（读的是哪个页面）与 `verified_on`（哪天人工确认过），经镜像/二手来源转述必须写进 `source_note`；`validate.py` 强制执行，`verified_on` 超 180 天告警。
3. **AGENTS.md 源信任层级**：写进 agent 指令的硬规则——欧盟委员会/EUR-Lex/FTC/NIST 等 P1 官方来源 > 路透/AP/Bloomberg P2 > 学术 P3 > 行业博客 P4 > 社交媒体与 AI 摘要 P5；P4/P5 来源的合规主张必须被 P1 官方佐证才能进 PR。法规延期/收窄的 deadline 只能原地更新并保留 history，不许删除。
4. **CI 全量自测**：guard 测试套件、安装器真实验证、文档中每个链接/路径/脚本名存在性检查、数据一致性校验、评估记分卡，全部在每次 push 时运行。

### 4.6 知识层精华：八大拒审根因

`docs/MISTAKE-PATTERNS.md` 把上百种表面拒因收敛成 8 个可执行的根因，每条附"纪律（discipline）"：

| # | 根因 | 一句话纪律 |
| --- | --- | --- |
| 1 | 不可见的功能（缺演示账号/后端没开/功能藏在付费墙后） | 审核员够不到的功能等于不存在 |
| 2 | 声明与运行时不符（隐私标签/Data Safety 与 SDK 行为不一致） | 声明是合同，App 必须严格兑现 |
| 3 | 支付捷径（数字商品绕开 IAP/Play Billing） | 把抽成当作分发成本，不是要绕过的 bug |
| 4 | 薄壳应用（网页套壳无增值） | 没有超出网页的原生价值就不该上架 |
| 5 | 垃圾重复（模板化/换皮/饱和品类克隆） | 一个概念一个 App，差异做进内容 |
| 6 | 权限抢夺（敏感权限无对应核心功能） | 每个权限都要能追溯到可见功能+具体用途字符串 |
| 7 | 缺少出口（有账号无删除、无隐私政策、无撤回同意） | 收数据的每个 App 都要给用户退路 |
| 8 | 复利式重提（被拒后原样重交碰运气） | Google 上这是从拒审通向封号的最快路径 |

文档还提供完整的**申诉手册**（Apple Resolution Center / Google Play Console 申诉流程与话术）和真实拒审案例复盘。

---

## 5. 适用场景

| 场景 | 说明 |
| --- | --- |
| **① 提交前一次性体检** | 任何开发者/非开发者，提交前花 2 分钟跑一次 guard，不安装任何东西 |
| **② AI 辅助开发的"发布保险丝"** | 用 Claude Code/Cursor 等工具开发 iOS/Android App，装上 hook 后每次提交命令前自动拦截——这是仓库的主打场景 |
| **③ 已被拒后的止损** | 按拒信里的 guideline 编号在数据层定位模式 → 应用 fix → 走申诉手册；明确禁止原样重提 |
| **④ 非开发者/外包验收** | 甲方只读 checklist，逐项问"这个勾是真的吗"，验收外包或 AI 生成的 App |
| **⑤ 企业多 App 舰队** | Fleet Sweep 一次给全舰队发布结论，误报走带过期时间的 Waiver Ledger 台账 |
| **⑥ 合规趋势追踪** | 用 8 个 monitor + deadline 数据库持续跟踪监管变化（EU AI Act、DMA、COPPA、各州年龄法、Play 政策迁移），可模拟新规影响并生成迁移 PR 草稿 |
| **⑦ CI/CD 门禁** | release-audit.py 退出码语义清晰（0/2/3），可直接做流水线质量门 |

---

## 6. 如何使用

### 6.1 最快上手（2 分钟，零安装）

```bash
git clone https://github.com/mjmirza/app-store-compliance ~/repositories/app-store-compliance
bash ~/repositories/app-store-compliance/agent-os/hooks/app-store-compliance-guard.sh "/path/to/your/app"
```

看输出的最后一行：`BLOCKED.` → 逐条修复 `[CRITICAL]` 再重跑；`CLEAR.` → 转去做 checklist 里代码扫不到的账号/列表项；`NOT CHECKED.` → 路径不对（Expo 项目先 `npx expo prebuild` 补出原生目录）。

### 6.2 装进 AI 编码工具（推荐，1 分钟）

```bash
bash ~/repositories/app-store-compliance/scripts/install.sh --tool all
# 或 --tool claude|codex|gemini|opencode|cursor|copilot|grok|qwen|kimi|droid|auggie|windsurf
```

装完重启工具（Codex 需 `/hooks` 人工批准一次）。之后在 Claude Code / Gemini CLI / OpenCode 里可用 `/app-store-audit` 触发完整五步审计（定位项目 → 自动扫描 → 人工补查 → 按 severity 出修复表 → 被拒时走申诉流程）。

### 6.3 让 Agent 自己完成安装 + 首次审计

README 提供了一段可直接粘贴给任意 AI 编码工具的提示词，Agent 会：克隆仓库 → 识别自己是什么工具并运行安装器 → 报告安装后动作 → 对当前项目跑 guard 出排名发现 → 带你走账号就绪清单和 90 天监管截止日期。提示词里明确写入了安全约束："不得替我操作 GitHub 账号（不 star/fork/follow），只给链接由我决定"。

### 6.4 完整发布门禁（团队/CI）

```bash
bash scripts/pull-metadata.sh apple                          # 拉取线上商店列表
python3 scripts/metadata-audit.py ./metadata --propose       # 元数据审计+修复建议
python3 scripts/release-audit.py /path/to/app --metadata ./metadata   # 总闸门
python3 scripts/fleet-sweep.py                               # 多 App 舰队扫描
```

### 6.5 引用可信度自检

```bash
python3 scripts/verify-citations.py --files docs/ data/ references/   # 需网络，几分钟
```

---

## 7. 目录结构导览

```
app-store-compliance/
├── README.md                    # 入口：按角色分流的使用指南（475 行，质量极高）
├── AGENTS.md                    # 给 AI agent 的 15 项发布审查清单 + 源信任层级 + PR 硬规则
├── data/                        # 机器可读数据层（一切的驱动源）
│   ├── rejection-patterns.json      # 123 条拒审模式分类学
│   ├── detection-recipes.json       # 107 条自动检测配方
│   ├── regulatory-deadlines.json    # 107 条监管截止日期（强制溯源字段）
│   └── ruling-register.json         # 14 条代码扫不出的发布要求（法律地位+证明方式）
├── agent-os/
│   ├── hooks/app-store-compliance-guard.sh   # ★ 1752 行守卫（核心）
│   └── skill/SKILL.md                        # 五步审计 Agent Skill
├── docs/                        # 30+ 篇知识文档（Apple/Google 拒审地图、EU/全球法规、
│                                #   平台机制、跨平台框架、八大根因+申诉、清单、舰队扫描…）
├── references/                  # generate-references.py 从数据自动生成的 AI 参考树
│   ├── rules/                   #   按类别（隐私18/Android37/性能34/支付11/元数据11…）
│   └── guidelines/by-app-type/  #   按应用类型（AI/订阅/儿童/健康/游戏/VPN/macOS…）
├── scripts/                     # ~1.95 万行工具（安装器/审计/监控/校验，每个都有测试）
├── evals/gold-cases.json        # 32 个金标准回归用例
├── templates/                   # 审核备注模板、fleet/waivers 配置示例
└── .github/                     # CI、PR/Issue 模板、反 AI 署名检查
```

---

## 8. 亮点与局限（客观评价）

### 8.1 亮点

1. **"提交命令前 1 秒拦截"的时机选择极准**：不改变开发流程，只在损失不可逆的动作前加一道闸，且 exit code 语义（0/2/3）对人和 CI 都友好。
2. **对 AI 工具生态的适配深度罕见**：一个 Bash 脚本兼容 13+ 工具各不相同的 hook payload schema（jq/python3/正则三级降级解析），安装器带"真阻断才叫装好"的 doctor 验证。
3. **数据驱动而非文档驱动**：123 条模式是唯一事实源，文档、参考树、检测命令全部生成自它，杜绝漂移；其他项目可直接复用这份 JSON。
4. **反幻觉工程是同类项目少见认真**：软 404 control 探测、强制 source_url/verified_on、源信任层级进 agent 指令、CI 全量测试——这套机制对任何"LLM 生成合规/法律内容"的项目都有参考价值。
5. **诚实的边界声明**：明确告诉用户代码扫描看不到什么（账号状态、审核备注、声明的数据用途），并用"提示而非默认干净"的保守策略覆盖盲区；监控器默认用样例数据并打印 `Data.` 行声明数据来源，防止把样例当新闻。

### 8.2 局限与注意

1. **静态扫描的天花板**：guard 是正则级静态分析，不执行代码、不算真实对比度、看不到运行时 SDK 行为；README 自己承认无障碍审计的多种误报/漏报形态。
2. **"故意保守"带来的噪音**：无法验证的项也会弹出来（作者视之为特性，用户需自行判断），首次使用可能被"疑似误报"淹没。
3. **监控器多数默认是内置样例数据**（除 monitor.py 拉 Apple RSS 外，需 `--live` 或无实时源的说明），把它当"新规播报"用会踩坑——文档对此有明示，但容易被忽略。
4. **单人维护 + 激进迭代**：规则时效性极强是优点也是风险，合并社区 PR 的可持续性待观察。
5. **许可证不是 MIT**（见下节），商用与二次分发前必须读条款。

---

## 9. 许可证与商用注意事项（重要）

仓库使用 **OpenRoots Agent License 2.3**（ORA-2.3，非 OSI 标准许可证）：

- **个人/非营利/学校/政府**：任何规模免费。
- **公司年营收 ≤ 2000 万美元**：免费商用、修改、分发（阈值每年通胀调整）。
- **公司年营收 > 2000 万美元**：对"使用该 playbook 的产品"超出阈值部分的营收收 **0.5% 版税**，封顶 25 万美元/年，季度自报；只从跨过阈值那天起算，不追溯。
- **必须保留署名**：版权声明、许可证引用、作者信息要放进**包清单（manifest）**，不能只写在 README。
- **不得转售本身**：在自己的 agent 里运行、改造、串联、服务客户都可以；把 playbook 重新打包成出售的产品需另行书面授权。
- **AI 训练不包含**：用本仓库训练模型需要单独的 Compute 许可；阅读与搜索索引不受影响。
- 历史版本：2026-08-24 前发布内容永久保持 MIT（代码）+ CC BY 4.0（内容）。

---

## 10. 总结

这是一个把"上架被拒"当作工程问题来求解的仓库：**用分类学（123 条模式数据）固化知识，用守卫脚本（Guard）把检查移动到损失发生前的最后一刻，用 AI hook 让"忘记检查"在物理上不可能，再用一套反幻觉机制保证知识本身不出假**。它对三类读者都有价值：

- **移动开发者**：直接可用——2 分钟体检，或一条命令给 AI 工具装上"发布保险丝"。
- **Agent 工程师**：`guard.sh` 是一个教科书级的 PreToolUse hook 实现（多工具 payload 兼容、三级降级解析、"未检查≠通过"、逃生阀），`AGENTS.md` 的源信任层级与溯源字段是构建任何"LLM + 事实型内容"系统的优秀范本。
- **平台/合规团队**：deadline 数据库 + Fleet Sweep + Waiver Ledger 提供了企业级多 App 合规治理的轻量方案雏形。

一句话收尾：在 AI 让"造 App"变得廉价、而商店审核没有变廉价的 2026 年，这个仓库把最贵的那次失败（审核拒绝）前置成了最便宜的一次检查（提交前扫描）——这正是它一年拿到 1800+ star 的原因。
