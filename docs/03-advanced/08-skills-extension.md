> 📖 **纸质书《OpenClaw超级个体实操手册》已上市！** 清华大学出版社出版，在开源教程基础上全面重写+逐条验证。🛒 [京东专属购买链接（¥42，原价¥59.8）](https://item.jd.com/14669463.html)

# 第8章节 Skills扩展（什么时候该用、怎么查、怎么装、怎么管）

> 本章目标：把 Skills 放回正确位置。它仍然重要，但已经不是“所有能力的唯一入口”。你会学到：什么时候该用 Skills，什么时候该用内建能力，如何安全地搜索、安装、检查和维护 Skills。

---

## 版本基线

- **当前稳定版**：`v2026.9.3`（2026-09-08 发布）
- 本章默认按 `v2026.9.3` 稳定版写；所有命令以官方 `openclaw skills` CLI 为准（不要默认 `clawhub install`）

---

## 先给小白的阅读说明

### 这一章最重要的结论

现在的 OpenClaw，不是“没有 Skills 就什么都干不了”。

很多能力已经有官方主线：

- 模型与 provider：`openclaw models`
- 无头能力调用：`openclaw infer`
- 自动化：`cron / tasks / Task Flow / webhooks`
- 记忆：`Active Memory / Memory Wiki`
- 媒体：`image_generate / video_generate / tts / music_generate`

**Skills 现在更适合做“可复用的工作方法和 SOP”**，而不是当成所有能力的默认入口。

### 如果你只想先学会最实用的部分，先看这些

- **想先装一个能用的 Skill**：看 `8.2`
- **想知道 Skills 和其他能力的分工**：看 `8.1`
- **想自己写一个本地 Skill**：看 `8.4`
- **最担心安全问题**：看 `8.5`

### 小白最容易犯的 4 个错

- 看到旧教程就直接运行 `clawhub install ...`
- 把 Skills、Tools、Plugins、MCP 混成一回事
- 没看源码就装第三方 Skill
- 明明该用 `infer` 或内建能力，却硬找一个 Skill 来做

---

## 8.1 Skills 在 2026.4 之后到底是什么？

### 8.1.1 一句话理解

你可以把 Skill 理解成：

- 一份可复用的操作说明书
- 一套会在合适场景下按需加载的 SOP
- 一段教 agent“遇到这类任务该怎么做”的专业经验

它最适合解决的问题是：**同一类任务你会反复做，而且做法相对稳定**。

### 8.1.2 什么场景该优先用 Skill

适合用 Skill 的情况：

- 固定写作流程
- 固定研究流程
- 固定交付模板
- 固定排查步骤
- 固定格式转换流程

不一定要用 Skill 的情况：

- 临时问答
- 一次性 prompt
- 单条命令就能完成的事
- 官方已经有明确主线能力的事（如 `infer`、媒体生成、Task Flow）

### 8.1.3 现在更实用的判断表

| 你的需求 | 更推荐什么 |
|----------|------------|
| 跑一个模型命令 | `openclaw infer` |
| 切换默认模型 | `openclaw models` |
| 做图片/视频/语音 | 官方媒体能力 |
| 做自动化编排 | `cron / tasks / Task Flow / webhooks` |
| 让 agent 学会一套固定 SOP | Skill |
| 连外部 API / 数据库 / 服务 | Plugin / MCP / Webhooks |

如果你不确定，就先问自己一句：

**“我现在缺的是一个功能入口，还是一套做事方法？”**

- 缺功能入口：先看官方内建能力
- 缺做事方法：再考虑 Skill

---

## 8.2 最快上手：现在官方该怎么查、怎么装、怎么管

官方当前推荐的 CLI 入口是 `openclaw skills`，不是旧教程里的 `clawhub install`。

### 8.2.1 最常用命令

```bash
openclaw skills search "calendar"
openclaw skills search --limit 20 --json
openclaw skills install <slug>
openclaw skills install <slug> --version <version>
openclaw skills install <slug> --force
openclaw skills update <slug>
openclaw skills update --all
openclaw skills list
openclaw skills list --eligible
openclaw skills list --json
openclaw skills list --verbose
openclaw skills info <name>
openclaw skills info <name> --json
openclaw skills check
openclaw skills check --json
```

### 8.2.2 小白先记住 5 个就够了

如果你第一次用，其实先会这 5 个就够：

```bash
openclaw skills search "写作"
openclaw skills install <slug>
openclaw skills list --eligible
openclaw skills info <name>
openclaw skills check
```

### 8.2.3 这些命令到底各自干什么

- `search`：去找有没有合适 Skill
- `install`：把 Skill 装到当前工作区
- `list --eligible`：看当前工作区里哪些 Skill 真正可用
- `info`：看某个 Skill 的详细信息
- `check`：检查本地 Skill 有没有结构问题或可见性问题

### 8.2.4 Skills 装到哪里

官方文档明确说明：`search / install / update` 走 ClawHub，但会把 Skill 安装到**当前活动工作区的 `skills/` 目录**。

也就是说，对大多数用户来说，你可以直接理解成：

- 当前项目里装的 Skill，会跟着这个项目走
- 不同工作区可以有不同 Skill 组合
- `list / info / check` 看的是当前工作区和当前配置下可见的本地 Skill

### 8.2.5 看到什么算已经装成功

至少满足下面 3 条：

- `openclaw skills list --eligible` 能看到你刚装的 Skill
- `openclaw skills info <name>` 能读到它的说明
- 你在合适任务里调用它时，agent 能真正识别并使用

如果只装上了，但 `eligible` 里看不到，先别急着怪模型，先跑：

```bash
openclaw skills check
```

---

## 8.3 Skills、Tools、Plugins、MCP 到底怎么分

### 8.3.1 最不容易混的理解方式

- **Skill**：一套做事方法
- **Tool**：一个具体工具动作
- **Plugin**：给 OpenClaw 增加一整类能力
- **MCP**：把外部系统接入为可调用能力

### 8.3.2 小白版例子

假设你想让 OpenClaw 帮你做“技术文章改写成公众号版本”：

- Skill：告诉它这类文章应该怎么拆结构、怎么改风格、怎么出标题
- Tool：读取文件、搜索网页、生成图片
- Plugin / MCP：连接某个知识库、外部文档系统或发布系统

所以它们不是互相替代，而是分工不同。

---

## 8.4 自己写一个最小本地 Skill

如果你已经反复做同一类事，就可以开始写自己的 Skill。

### 8.4.1 最小目录结构

```text
skills/
└── my-writing-helper/
    └── SKILL.md
```

### 8.4.2 最小示例

```markdown
---
name: my-writing-helper
description: 把技术笔记整理成公众号文章大纲
---

# my-writing-helper

## 什么时候用

当用户要把技术笔记改成更适合公开发布的中文文章时使用。

## 步骤

1. 先提取原始笔记里的核心观点
2. 再重写成更清晰的文章结构
3. 最后输出标题、摘要、小标题和结尾行动建议
```

### 8.4.3 写完之后怎么检查

```bash
openclaw skills list --eligible
openclaw skills info my-writing-helper
openclaw skills check
```

### 8.4.4 什么时候值得自己写 Skill

最值得写 Skill 的情况是：

- 你已经重复做过 5 次以上
- 这件事有稳定步骤
- 你每次都在重复解释同样要求
- 你希望不同 agent 或不同项目都复用这套做法

如果只是一次性需求，先别写 Skill，直接用 prompt 更省时间。

---

## 8.5 安全：第三方 Skill 应该怎么装才稳

官方文档对这一点说得很直接：**把第三方 Skills 当成不受信任代码处理**。

### 8.5.1 小白版安全规则

安装前先做到这 4 件事：

1. 先看 `openclaw skills info <name>`
2. 尽量阅读源码或至少阅读 `SKILL.md`
3. 对不熟悉的 Skill，优先在沙箱或低风险项目里试
4. 装完先跑 `openclaw skills check`

### 8.5.2 为什么不能无脑装

官方文档提到：

- 第三方 Skill 应该视为不受信任代码
- Gateway 侧的依赖安装流程会做危险代码扫描
- 但这不等于你可以不看、不判断、不复核

所以更稳的习惯是：

- 先装你看得懂用途的 Skill
- 先装数量少、目标明确的 Skill
- 一次只加一个，确认没问题再加下一个

### 8.5.3 哪些信号要提高警惕

- 描述很模糊，但权限要求很多
- 需要你额外执行可疑脚本
- 要求访问你不相关的目录或系统能力
- 你根本不知道它为什么需要这些权限

---

## 8.6 给小白的最短落地顺序

如果你今天只想花 15 分钟把 Skills 跑起来，按这个顺序做：

1. `openclaw skills search "你要做的事"`
2. 选一个最简单、用途最明确的 Skill
3. `openclaw skills install <slug>`
4. `openclaw skills list --eligible`
5. `openclaw skills info <name>`
6. `openclaw skills check`
7. 在一个真实任务里试一次

---

## 8.7 本章最容易踩的坑

### 坑 1：把 Skills 当成所有能力的入口

现在很多能力已经有官方更稳定的主线，不需要先找 Skill。

### 坑 2：直接照搬旧教程里的 `clawhub install`

当前官方命令以 `openclaw skills` 为准。旧写法不应该再作为默认主路线。

### 坑 3：一上来就装很多 Skill

最稳的方式永远是：

- 先装一个
- 先看它能不能用
- 先确认它到底解决了什么问题

### 坑 4：明明是工作方法问题，却去换模型

有些问题不是模型不够强，而是你缺一套稳定 SOP。那才是 Skill 真正适合出场的时候。

---

## 8.8 官方参考

- Skills CLI：https://docs.openclaw.ai/cli/skills
- Skills 工具说明：https://docs.openclaw.ai/tools/skills
- Plugins CLI：https://docs.openclaw.ai/cli/plugins
- Hooks CLI：https://docs.openclaw.ai/cli/hooks
- Models CLI：https://docs.openclaw.ai/cli/models
