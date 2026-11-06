# Hermes × Antigravity 订阅：接入与排错

把 **Google Antigravity 订阅**（Gemini 4 Argon / Gemini 3.8 Flash / Claude 5.5 / GPT-OSS）
接成 [Hermes Agent](https://github.com/NousResearch/hermes-agent) 的模型提供方。
官方 `agy` CLI 负责认证与推理，Hermes 负责工具执行与审批。

本仓库是一个 **Hermes skill**（`SKILL.md`），内容是一份排错手册 —— 不做一键脚本，因为这条路
真正会卡人的地方是资格与登录，而不是命令。

## ⚠️ 使用前必读

Google **Antigravity Additional Terms 第 6 条**明文规定：

> Using third party software, tools, or services to access the Service (e.g. using OpenClaw with
> Antigravity OAuth) is a breach of this Agreement. Such actions may be grounds for **suspension
> or termination** of your Antigravity and/or Gemini CLI accounts.

社区有大量自动封号实证（2026 年起，含仅使用官方客户端却被误伤者）。**这不是"安全方案"**：
用第三方工具驱动订阅属违约，账号可能被停，数据会出境到 Google 且可能被人工审阅。
仅个人研究使用，风险自担。

## 它解决什么

- `agy` 装好了，但 `agy -p` 报 `403 Verify your account` / `not eligible`
- 官方给的验证链接打开就是 `500. That's an error.`
- 菜单里模型只有 5 个，不知道 `-high` 跑哪去了
- 改完 `.env` 之后，模型选择器里还挂着旧 provider

这些都写进 `SKILL.md` 的"三个独立关卡"和"常见故障"表里了。

## 安装为 Hermes 技能

```bash
git clone https://github.com/MOUTAI-9/<repo>.git ~/.hermes/skills/antigravity-subscription-hermes
```

之后相关任务会自动加载；也可 `skill_view(name='antigravity-subscription-hermes')` 直接读。

## 免责

与 Google、Nous Research 均无关联。文中名称仅用于描述兼容关系，商标归各自所有。
MIT License。
