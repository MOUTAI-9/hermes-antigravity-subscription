---
name: antigravity-subscription-hermes
description: "用 Antigravity 订阅在 Hermes 跑模型，含资格排错。"
version: 1.0.0
author: CHEN (MOUTAI-9)
license: MIT
platforms: [macos, linux, windows]
metadata:
  hermes:
    tags: [antigravity, agy, gemini, subscription, model-provider, troubleshooting]
    related_skills: [hermes-agent, claude-code, codex]
---

# Antigravity 订阅接入 Hermes（含排错）

把 Google Antigravity 订阅（Gemini 4 Argon / Gemini 3.8 Flash / Claude 5.5 / GPT-OSS）
接成 Hermes 的模型提供方。核心是：**官方 `agy` CLI 负责认证与推理，Hermes 负责工具执行**。

> ⚠️ **先读风险再动手**。Google Antigravity Additional Terms 第 6 条原文把这件事定性为违约：
> "Using third party software, tools, or services to access the Service (e.g. using OpenClaw
> with Antigravity OAuth) is a breach of this Agreement. Such actions may be grounds for
> suspension or termination of your Antigravity and/or Gemini CLI accounts."
> 社区有大量自动封号实证（含纯官方用法被误伤者）。本人自担风险，勿用于敏感数据。

## 何时使用

- 想把 Google Antigravity 订阅的模型当 Hermes 的大脑（而不是另起一个 CLI）
- 已经装好 `agy` 但连接/资格/菜单有问题，需要按现象定位

## 三个独立的关卡（90% 的困惑来自把它们混为一谈）

| 关卡 | 含义 | 失败表现 |
|---|---|---|
| ① 登录 | `agy` 的 Google OAuth | `agy models` 报 "Please sign in" |
| ② 目录信任 | 允许 agy 读写**当前目录**的文件 | 每次启动都问 "Do you trust this folder?" |
| ③ 资格 | 账号是否有 Antigravity 使用权（按地区/风控） | `403 Verify your account` / `not eligible` |

**① 成功 ≠ 能用**。`agy models` 只需要登录（读的是账号元数据），真正的推理与额度查询要过
`daily-cloudcode-pa.googleapis.com` 的资格校验。所以"能列模型、一调用就 403"是常态，不是配置错。

## 第 0 步：前置检查

1. Hermes ≥ 0.21.4（`hermes --version`）
2. macOS / Linux / Windows（插件三平台都支持；`agy` 本身也三平台）
3. 有 Antigravity 资格的 Google 账号 + 能访问 Google 的网络（地区限制是硬门槛）

## 第 1 步：装 `agy`（官方安装脚本）

```bash
# macOS / Linux
curl -fsSL https://antigravity.google/cli/install.sh | bash
# Windows PowerShell
irm antigravity.google/cli/install.ps1 | iex
```

脚本行为（已核对）：拉 manifest → **校验 SHA512** → 装到 `~/.local/bin/agy` →
在 `~/.zshrc` / `~/.zprofile` 追加一行 `export PATH="/…/.local/bin:$PATH"` → 清 macOS quarantine 属性。

安装后若 PATH 里已有 `~/.local/bin`，会把那行**重复追加**，建议删掉新增的那行，保留幂等写法的旧行。

验证：

```bash
command -v agy && agy --version    # 例：1.5.1
```

## 第 2 步：登录 + 资格验证（失败就别往下走）

```bash
agy models          # 只验证登录；输出 13 条 model-effort slug
agy -p=/quota       # 验证资格 + 额度；失败会返回 eligibility 错误
```

`agy models` 输出形如 `gemini-4-argon-{low,medium,high}`、`claude-opus-5-5-{low,medium,high}`、
`gpt-oss-120b-medium`，共 4×3+1 = 13 条。

### 资格报错怎么处理

现象：`Eligibility check failed: Your current account is not eligible for Antigravity.`
日志位置：`~/.gemini/antigravity-cli/log/cli-*.log`，关键字 `al_alert`、`PERMISSION_DENIED 403`。

它给的 `https://accounts.google.com/v3/signin/continue?plt=…&authuser` 链接**通常打不开**：
`plt` 是一次性策略令牌，只在签发它的浏览器会话里有效，换浏览器/过几分钟就是
`500. That's an error.`。不要反复重试这条链接。

正确做法：

1. 用**平时登录该 Google 账号的同一个浏览器 profile** 打开 `https://myaccount.google.com/security`，
   看有没有"安全提醒 / 验证你的账号"，按提示完成（手机号那类）。**验证码不要交给任何 agent**。
2. 打开 `https://antigravity.google` 用同一账号确认地区资格；提示 not available in your country
   就是地区限制，验证救不了。
3. 两步验证（2FA）**不等于**这里的账号验证——实测设了 2FA 仍 `not eligible`，不要白折腾。
4. 完成后再 `agy -p=/quota`，看到 `Weekly/Five Hour Limit Remaining 100%` 即通过。

## 第 3 步：装 Hermes 侧 provider 插件

Hermes 官方 curated 目录里有三个同形状的插件，选维护中的那个：

| 插件 | 维护 | 关键差异 |
|---|---|---|
| `antigravity-subscription-directsdk`（soyelmismo） | 活跃 | 持久 agy worker，只送增量消息，长会话不截断 → **推荐** |
| `antigravity-oauth`（neerazz） | 有更新 | 把 SOUL.md 写进 GEMINI.md，工具调用重新映射 |
| `antigravity-agy`（Realtyxxx） | 已归档 | 整段对话压成一条消息，超 ~65k token 被 agy 静默截断 → 长会话丢轮次 |

```bash
hermes plugins install antigravity-subscription-directsdk --yes-deps --no-enable
hermes plugins enable antigravity-subscription-directsdk
```

启用后无需改 `config.yaml`（插件通过 `plugin.yaml` 的 `kind: model-provider` 注册）。

## 第 4 步：验证（必须两条都跑）

```bash
# 1) 单轮
hermes chat -q "只回复三个字：接入成功" \
  --provider antigravity-subscription-directsdk --model gemini-4-argon-medium

# 2) 工具执行（证明工具由 Hermes 跑，而不是 agy 自己跑）
hermes chat -q "用终端执行 whoami，只回答用户名" \
  --provider antigravity-subscription-directsdk --model gemini-4-argon-medium
```

菜单里应出现一行 `Antigravity Subscription DirectSDK`。

## 模型菜单只有 5 行是正常的

agy 列 13 条 = 4 个系列 × 3 档 + gpt-oss 1 档；插件按 `-low/-medium/-high` 后缀折叠成 5 个基础模型，
档位通过 `--effort` 发送。换档三种写法（优先级：显式参数 > 模型名后缀 > 配置 > 默认 medium）：

```bash
/reasoning high                       # 会话内
hermes chat -q "…" --reasoning low    # 单次
/model gemini-4-argon-high            # 带后缀，后缀优先
```

## 成本与限制

- 每次请求固定约 **6.9k input tokens** 打底（agy 自带 agent 提示词），空问一句也要付。
- 只吃文本：图片会被替换成占位符。
- `reasoning_effort` 之外的 `temperature` / `max_tokens` / `response_format` 基本无效。
- 并行工具调用会被串行成多次请求。
- 额度分两类窗口：Gemini 模型、Claude/GPT 模型各算各的（5 小时 + 每周）。

## 撤回

```bash
hermes plugins disable antigravity-subscription-directsdk   # 从 Hermes 摘掉
agy /logout                                                # 彻底收回授权（交互式）
```

## 常见故障

| 现象 | 原因 | 处理 |
|---|---|---|
| 菜单里还显示已删掉的 provider（如 Google AI Studio） | 网关进程内存里仍有旧环境变量，改 `.env` 不影响运行中进程 | 退出 Hermes Desktop 重开，或在**网关外**执行 `hermes gateway restart`（在网关进程内执行会被拦） |
| `agy: command not found` | `~/.local/bin` 不在 PATH（launchd / systemd / 网关进程常见） | 用绝对路径，或给服务显式设 PATH |
| 每次启动都问 trust | 在父目录（如 `$HOME`）启动 | `cd` 到子目录再启动，**永远不要把 `$HOME` 交给它信任** |
| `not eligible` 反复出现 | 地区/账号资格 | 换支持地区网络 + 重新授权；或改走 API key 路线 |
| 长会话后模型开始"失忆"/循环 | 单条 user 消息超 ~65k token 被 agy 截断 | 换用 directsdk（增量发送），或把任务切块 |

## 边界（强烈建议）

- **只喂脱敏内容**：代码、模板、文案、结构化占位数据。店铺经营数据、客服原始记录、财报一律不进。
  退出 `agy` 时在 onboarding 里**取消勾选**「允许 Google 收集并人工审阅 Interactions」。
- **不要用别人的账号**：Antigravity 按登录账号判资格与风控，家庭组共享的是权益不是身份；
  封号会连坐该账号的邮箱/云盘，且共享凭据本身违反 Google 条款。
- 想长期、合规地跑 Gemini：用 `provider: gemini` + 可用的付费 API key（免费档对 frontier 模型
  常是 `limit: 0`，实测直接 429）。
