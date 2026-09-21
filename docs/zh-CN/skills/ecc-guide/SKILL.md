---
name: ecc-guide
description: "把任何“ECC 里该用哪个部分来做 X？”的问题路由到确切的组件——skill、命令、agent、hook、规则、MCP 连接器、安装配置档案——依据仓库实时状态回答，绝不凭记忆。一屏之内给出答案、规范路径和验证命令。TRIGGER：用户询问 ECC 包含什么、某个组件在哪里、哪个组件适合某项任务、如何安装/重置/迁移/卸载 ECC、某个 hook 或连接器为何这样表现，或命令、skills、agents、hooks 与规则之间的关系。DO NOT TRIGGER：用户想直接执行任务（应调用对应组件）、想要带执行顺序与停止条件的多命令流水线（用 ecc-recipes）、或想要交互式安装向导（用 configure-ecc）。"
argument-hint: "<主题 | find: 查询 | 组件名 | 留空=菜单>"
origin: community
metadata:
  version: "2.0.0"
  surface-baseline: "2027"
---

# ECC 指南

Everything Claude Code 的导航层。把模糊的意图变成一个明确的组件、它的规范文件，
以及一条能验证该答案的命令。

**约定：** 仅提供建议、只读。本技能负责定位和解释组件，从不安装、从不修改配置，
也从不代为运行它所指出的组件。

## 何时使用

- “ECC 包含什么？” / “用 ECC 怎么做 X？”
- 查找某个 skill、命令、agent、hook、规则、MCP 连接器或安装配置档案
- 在两个看起来功能重叠的组件之间做选择
- 理解安装路径、作用域、重复安装、重置与卸载
- 解释命令、skills、agents、hooks、规则与 MCP 之间的关系
- 排查“ECC 已安装，但 X 没出现”

### 不要使用的情况

| 情况 | 改用 |
|---|---|
| 用户希望立即执行任务 | 对应的 skill/命令 |
| 用户需要带执行顺序与停止条件的多命令流水线 | `ecc-recipes` |
| 用户想交互式安装、重新配置或迁移作用域 | `configure-ecc` |
| 用户想重写草稿提示词 | `prompt-optimizer` |
| 用户想统计 ECC 组件的 token/成本 | `context-budget`、`ecc-tools-cost-audit` |

## 首要原则

**依据当前文件回答，绝不凭记忆。** ECC 的目录每周都在变。任何写死的数量、
功能清单或安装参数，迟早都会变成错误答案。

由此得出三条规则：

1. 没有刚刚读到的数量，就不要说出口。
2. 没有落到文件系统上确认，就不要断言某个组件存在。
3. 如果没有可用的检出目录，请说明这一点，并给出结构性回答
   （“skills 位于 `skills/<name>/SKILL.md`”），而不是猜名字。

## 读取预算阶梯

只升级到问题真正需要的层级。大多数问题止步于 T1。

| 层级 | 问题形态 | 读取 | 目标开销 |
|---|---|---|---|
| **T0** | 概念性——“skill 和命令有什么区别？”“hook 配置档案是什么？” | 无 | ~0 token |
| **T1** | 存在/位置——“有 Rust reviewer 吗？” | 一次 `find` 或 `rg` | < 1k token |
| **T2** | 选择——“哪个适合我的仓库？” | 2-4 个候选的 frontmatter | < 4k token |
| **T3** | 全量普查——“列出所有 X”（仅在明确要求时） | `catalog.js --json` | 10k+ token |

按升级顺序的低成本探测：

```bash
# T1 —— 存在与位置（最快，无依赖）
ls skills/<name>/SKILL.md commands/<name>.md agents/<name>.md 2>/dev/null
rg -l "<query>" skills commands agents rules docs --max-count 1

# T2 —— 不读全文即可在候选之间决策
head -6 skills/<name>/SKILL.md            # 只看 frontmatter
rg -n "^description:" skills/*/SKILL.md | rg -i "<query>"

# T3 —— 完整实时目录（仅当明确要求“列出全部”）
node scripts/ci/catalog.js --json
node scripts/install-plan.js --list-profiles
node scripts/install-plan.js --list-components --json
```

效率规则：先读 frontmatter 再读正文；本次会话内缓存已读内容；把彼此独立的探测
合并成一次调用；同一会话中绝不重复运行 `catalog.js`。

## 意图路由表

在读取任何文件之前，先把用户的用词映射到组件类别。

| 用户表述 | 组件类别 | 定位于 | 验证方式 |
|---|---|---|---|
| “工作流 / playbook / 怎么做 X” | skill | `skills/<name>/SKILL.md` | `ls skills/<name>/` |
| “斜杠命令 / `/x`” | 命令 | `commands/<name>.md` | `ls commands/` |
| “委派 / 子 agent / 并行” | agent | `agents/<name>.md` | `ls agents/` |
| “自动 / 每次编辑时 / 阻止我” | hook | `hooks/hooks.json`、`scripts/hooks/` | `cat hooks/README.md` |
| “始终遵守 / 规范 / 策略” | 规则 | `rules/` | `ls rules/` |
| “连接到某个外部系统” | MCP 或 skill | `mcp-configs/mcp-servers.json`、`docs/MCP-CONNECTOR-POLICY.md` | 见 MCP 预算 |
| “安装 / 配置档案 / 作用域 / 卸载” | 安装器 | `manifests/install-*.json`、`README.md` | `node scripts/install-plan.js --list-profiles` |
| “为这个仓库配置 ECC” | 接入 | `/project-init` | dry-run 计划 |
| “我的配置是否健康/安全” | 审计 | `/harness-audit`、`/security-scan` | `npm run harness:audit -- --format text` |

**两个组件都匹配时的优先级：** skill > agent > 命令 > hook。
skills 是主要的工作流层；命令是仍在维护的兼容入口；当工作应在独立上下文窗口中
运行时选 agent；只有当行为必须在无人主动调用时也触发，才选 hook。

## 组件模型

各用一句话解释——当用户搞不清层次时使用：

- **Skill** —— 模型在*相关时*才加载的工作流。渐进式披露：只有激活时才消耗上下文。
- **命令** —— 用户显式输入的入口。调用确定，内容性质与 skill 同类。
- **Agent** —— 在*独立*上下文窗口中执行的委派工作。适合大范围搜索、独立评审，
  以及任何会淹没主线程的任务。
- **Hook** —— 在生命周期事件上的确定性自动化。无论模型是否配合都会运行；
  这是强制执行层。
- **规则** —— 始终加载的指导。每行的上下文税最高，因此这里最短者胜。
- **MCP 连接器** —— 带会话状态的实时外部系统。无论用不用，工具 schema 都会
  加载进*每一个*会话。

## MCP 连接器预算（2027 姿态）

规范策略：`docs/MCP-CONNECTOR-POLICY.md`。要点如下：

一个连接器要占据默认位，必须既**通用**，*又*确实需要 MCP 才能提供的能力——
保持打开的会话状态、流式传输、认证握手或结构化浏览。无状态的请求/响应工作
应当是包装 CLI 或 REST API 的 skill，而不是 server。

2027 年主流 harness 的实际默认值仍是**零到两个默认连接器加上原生内置能力**。
ECC 只发布一个（`chrome-devtools`）；六个在 2026 年 6 月的审计中被降级为 skill，
并作为可选项保留在 `mcp-configs/mcp-servers.json` 中。该审计结论在 2027 年依然
成立——harness 原生的搜索、记忆与思考能力，只是吸收了更多这些 server 原本赖以
存在的理由。

当用户要求新增连接器时，按此顺序发问：

1. 已有的 CLI 或 REST API 能做到吗？→ 那就是 skill，不是 server。
2. 价值在于*保持的会话*，还是一次性调用？→ 一次性就是 skill。
3. 需要密钥吗？→ 通用性不达标；只能可选，绝不设为默认。
4. 对那些从不调用它的会话，schema 税是多少？

用 `ECC_DISABLED_MCPS="chrome-devtools"` 关闭已发布的连接器。

## 安装指引

始终先出计划、再 dry-run、最后应用。当托管安装器支持该目标时，绝不手工复制
组件文件。

```bash
node scripts/install-plan.js --list-profiles
node scripts/install-plan.js --profile minimal --target claude --json
node scripts/install-apply.js --profile minimal --target claude --dry-run

# 安装单个 skill 而不是整个配置档案
node scripts/install-plan.js --skills <skill-id> --target claude --json
```

值得了解的参数：`--modules`、`--with`、`--without`、`--family`、`--config`、
`--target`。目标涵盖 Claude Code、Codex、Cursor、OpenCode、Kimi、Gemini、
CodeBuddy、JoyCode、Qwen——请用 `--list-components --json` 查看实时支持情况，
而不是凭记忆断言支持矩阵。

**重复安装告警：** 插件安装*叠加*完整手工安装或配置档案安装，会让每个组件出现
两份。先确认用户是否有意为之。

## 故障排查

按此顺序分诊；在第一个能解释症状的层级停止。

| 症状 | 首先检查 |
|---|---|
| 安装后组件缺失 | 安装作用域与目标目录——`.claude/`、`.codex/`、`.cursor/`、`.opencode/`、`.gemini/`、`.kimi-code/`、`.codebuddy/`、`.joycode/`、`.qwen/` |
| 组件出现两份 | 插件安装叠加了手工/配置档案安装 |
| hook 不触发 | `hooks/hooks.json` 的 matcher，然后是 `ECC_HOOK_PROFILE` / `ECC_DISABLED_HOOKS` |
| hook 触发过多 | hook 配置档案为 `strict`；降到 `standard` 或 `minimal` |
| 脚本报 `Cannot find module` | 依赖未安装——`npm ci` |
| 连接器工具不存在 | `ECC_DISABLED_MCPS`，然后是 harness 自身的 MCP 配置 |
| 会话变慢 / 上下文吃紧 | `context-budget`，优先精简规则与连接器 |

仓库健康检查，按开销升序：

```bash
npm run harness:audit -- --format text
npm run observability:ready
npm test
```

## 回答模板

先给答案。一屏之内讲完，再提供深入路径。

```text
用 <组件>。适合的原因：<一条理由>。
规范文件：<path>
验证：    <command>
下一步：  <一个具体动作>
```

搜索场景：

```text
最佳匹配：
- <path> —— <为什么重要>
- <path> —— <为什么重要>
先从这个开始：<某个> 因为 <理由>。
```

安装场景：

```text
检测到：<技术栈证据>
目标：  <harness>  作用域：<user|project|local>
计划：  <profile/modules/skills>
Dry run：<command>
将变更：<paths>
应用前需要批准：<yes/no>
```

## 反模式

- 用户只要一条路径，却抛出整个目录
- 凭记忆报出数量、版本或配置档案名
- 已有 skill 优先路径时，仍推荐已退役的命令入口
- 安装器支持该目标时，仍给出手工 `cp` 指令
- 为 T1 问题升级到 T3
- 用户只问其中一类，却把六类组件全讲一遍
- 不套用四条预算问题就推荐新增 MCP 连接器

## 相关组件

| 需求 | 组件 |
|---|---|
| 带执行顺序与停止条件的命令流水线 | `ecc-recipes` |
| 交互式安装 / 重新配置 / 作用域迁移 | `configure-ecc` |
| 面向目标仓库的技术栈感知接入 | `/project-init` |
| 确定性就绪度评分卡 | `/harness-audit` |
| skill 质量评审 | `/skill-health` |
| 从本地 git 历史生成 skill | `/skill-create` |
| 配置安全评审 | `/security-scan` |
| token/成本核算 | `context-budget`、`ecc-tools-cost-audit` |
