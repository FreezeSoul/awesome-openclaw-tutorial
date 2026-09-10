> 📖 **纸质书《OpenClaw超级个体实操手册》已上市！** 清华大学出版社出版，在开源教程基础上全面重写+逐条验证。🛒 [京东专属购买链接（¥42，原价¥59.8）](https://item.jd.com/14669463.html)

# 第10章节 API 与外部能力集成（Infer / Webhooks / 媒体工作流）

> 本章目标：按 OpenClaw 官方最新主线，把模型推理、媒体能力、Webhook 接入和外部系统联动这几件事讲清楚，并替换掉旧的第三方 Skill 默认路线。

---

## 版本基线（请先统一口径）

- **当前稳定版**：`v2026.9.3`（2026 年 6 月 16 日发布）
- 本章默认按 **`v2026.9.3` 稳定版** 写；历史 beta / 旧模型路由只作为旧案例参考

> 如果你机器上还停在 `v2026.4.12` 或更早版本，先升级再看这一章，不然你会在命令名、能力入口和配置路径上反复踩坑。

---

## 先给小白的阅读说明

### 这章适合谁

- 你已经把 OpenClaw 装好，想让它调用模型、图片、音频、视频或网页能力
- 你想把 Notion、表单、Webhook、自动化平台接进 OpenClaw
- 你看过旧教程，发现很多 Skill 名称已经装不上，想知道现在到底该怎么做

### 开始前要先准备什么

在继续之前，至少先确认这 3 件事：

1. `openclaw onboard` 已跑过
2. `openclaw models status` 能看到你已经登录的 provider
3. 你知道自己现在是在做哪一类事：**命令行推理**、**会话内工具调用**，还是**外部系统触发**

### 如果你只想先跑通，按这个顺序看

- **只想先让命令跑通**：看 `10.2` + `10.3`
- **只想接外部系统**：看 `10.4`
- **只想知道旧 Skill 为什么不该再照抄**：看 `10.1` + `10.5`

### 先记住 3 句话

- `openclaw infer` 负责**无头、脚本化、命令行调用**
- agent 工具负责**会话里自动调用能力**
- `hooks` / `webhooks` / `Task Flow` 负责**外部事件接入和多步骤编排**

---

## 10.1 2026.4 之后，API 集成的正确主线是什么？

旧版教程的主要问题，不是“不会接 API”，而是默认把很多**已经过时的第三方 Skill 名称**和**不再是官方主路线的命令**当成首选方案。现在更推荐的路径是：

| 需求 | 旧写法常见问题 | 现在推荐的官方路径 |
|------|----------------|--------------------|
| 文本推理 | 零散脚本、手工拼 provider | `openclaw infer model run` |
| 图片生成 | 依赖历史第三方 Skill | `openclaw infer image generate` 或 agent 工具 `image_generate` |
| 视频生成 | 旧命令、旧 Skill 名称失效 | `openclaw infer video generate` 或 agent 工具 `video_generate` |
| 语音转写 | 手工脚本多、格式易错 | `openclaw infer audio transcribe` |
| TTS 合成 | 旧命令路径不统一 | `openclaw infer tts convert` 或 agent 工具 `tts` |
| Web 搜索 / 抓取 | 自己维护爬虫 | `openclaw infer web search` / `openclaw infer web fetch` |
| Embedding | 各 provider 自己写一套 | `openclaw infer embedding create` |
| 外部系统触发 | 靠 cron + shell 拼接 | `hooks` / `webhooks` plugin / `Task Flow` |
| 本地媒体工作流 | 零散脚本 + 图形界面切换 | 官方 `ComfyUI` provider/plugin |

一句话总结：

1. **“推理类”需求**优先走 `openclaw infer`
2. **“会话内自动调用”**优先交给 agent 工具（`image_generate`、`video_generate`、`music_generate`、`tts`）
3. **“外部系统驱动”**优先走 `hooks`、`webhooks` plugin 和 `Task Flow`
4. 只有这些都不满足时，再考虑自定义 plugin 或外部中间层

---

## 10.2 先把 provider 和模型能力配好

在开始接入任何 API 前，先确保 OpenClaw 自己已经能正常访问模型与媒体能力。最短路径：

```bash
# 1) 推荐：先走引导向导
openclaw onboard

# 2) 查看模型与认证状态
openclaw models status
openclaw models list

# 3) 按 provider 登录
openclaw models auth login --provider openai --set-default
openclaw models auth login --provider anthropic --method cli --set-default

# 4) 设置主模型与图像理解兜底模型
openclaw models set openai/gpt-5.4
openclaw models set-image openai/gpt-4.1-mini
```

如果你是多 provider 环境，建议同时配好主模型和回退链：

```bash
openclaw models set openai/gpt-5.4
openclaw models fallbacks add anthropic/claude-sonnet-4-5
openclaw models fallbacks add google/gemini-2.5-pro
```

**什么时候需要 `models status --probe`？**

- 你怀疑 token 已过期
- provider 列表看着有，但实测不通
- 刚切换了 OAuth / API key，需要确认可用性

```bash
openclaw models status --probe
```

### 看到什么算配置成功

如果下面这些都成立，说明你已经可以继续往下学：

- `openclaw models status` 能看到主模型和已登录 provider
- `openclaw models status --probe` 没有明显的 auth / token 错误
- 你已经知道自己默认要走哪条模型路线

如果这里都不通，先别急着接外部系统。先把模型认证和默认模型配通，再往后走。

---

## 10.3 `openclaw infer`：当前最重要的统一入口

官方文档已经明确：`openclaw infer` 是当前**标准的无头能力入口**。它覆盖：

- 文本推理
- 图片生成 / 编辑 / 描述
- 音频转写
- 语音合成
- 视频生成 / 描述
- Web 搜索 / 抓取
- Embedding 创建

### 10.3.1 常用命令速览

```bash
openclaw infer model run --prompt "Reply with exactly: smoke-ok" --json
openclaw infer image generate --prompt "friendly lobster illustration" --json
openclaw infer audio transcribe --file ./memo.m4a --json
openclaw infer tts convert --text "hello from openclaw" --output ./hello.mp3 --json
openclaw infer video generate --prompt "cinematic sunset over the ocean" --json
openclaw infer web search --query "OpenClaw docs" --json
openclaw infer embedding create --text "friendly lobster" --json
```

对小白来说，不要一口气全跑。最稳的顺序是：

1. 先跑 `model run`，确认文本模型能用
2. 再跑 `web search`，确认联网推理路径没问题
3. 需要哪种媒体能力，再单独测哪一个

这样一旦失败，你更容易知道是**模型认证问题**、**文件路径问题**，还是**媒体 provider 没配好**。

### 10.3.2 文本推理：把零散脚本换成标准命令

```bash
openclaw infer model run   --prompt "用 5 条 bullet 总结 OpenClaw v2026.9.3 的主要变化"   --json
```

适合：

- shell 脚本里做一跳摘要
- CI 里做 release note 总结
- 给下游自动化产出稳定 JSON

### 10.3.3 图片生成：默认走官方能力

```bash
openclaw infer image generate   --prompt "一张手写白板风格的 OpenClaw 工作流示意图"   --json
```

如果你是从现有文件继续改图，用 `image edit`；如果你要读图说明内容，用 `image describe`：

```bash
openclaw infer image describe   --file ./ui-screenshot.png   --model openai/gpt-4.1-mini   --json
```

> 注意：`image describe` 这类命令的 `--model` 必须写成完整的 `<provider/model>` 形式。

### 10.3.4 音频转写：不要再手写 whisper 脚本

```bash
openclaw infer audio transcribe   --file ./team-sync.m4a   --language zh   --prompt "重点提取人名、决策与行动项"   --json
```

适合：

- 会议纪要
- 播客拆解
- 微信语音 / 飞书语音整理

### 10.3.5 TTS：统一走 `tts convert`

```bash
openclaw infer tts convert   --text "今天的日报已经生成完成"   --output ./daily-report.mp3   --json
```

如果你是在 agent 对话里，需要回复直接带语音，优先让 agent 自动调用 `tts` 工具；如果你是在脚本、批处理或自动化流水线里，优先用 `infer tts convert`。

### 10.3.6 视频生成：现在是异步任务，不是同步截图脚本

```bash
openclaw infer video generate   --prompt "一段 5 秒的电影感镜头：小龙虾在日落海边冲浪"   --json
```

需要注意两点：

1. 视频生成通常是**异步长任务**，底层 provider 会先返回任务 id
2. OpenClaw 会把视频任务纳入 task ledger，必要时你可以配合 `openclaw tasks list` 查看进度

### 10.3.7 Web 搜索与抓取：先用官方再谈爬虫

```bash
openclaw infer web search --query "OpenClaw v2026.9.3 release notes" --json
openclaw infer web fetch --url https://docs.openclaw.ai/cli/infer --json
```

这套命令特别适合：

- 自动化日报
- 竞品监控
- 资料初筛
- 内容采编前的资料抓取

### 10.3.8 Embedding：统一走 `embedding create`

```bash
openclaw infer embedding create   --text "客户反馈：物流延迟、赔付说明不清晰"   --json
```

适合：

- FAQ 聚类
- 工单语义归类
- 外部知识库入库前向量化

---

## 10.4 外部系统怎么接？用 Hooks、Webhooks Plugin 和 Task Flow

### 10.4.1 轻量触发：`hooks`

如果只是让外部系统“叫醒” OpenClaw 或启动一次 isolated agent run，最轻的方案是 `hooks`。

配置示例：

```json
{
  "hooks": {
    "enabled": true,
    "token": "replace-with-dedicated-hook-token",
    "path": "/hooks"
  }
}
```

**唤醒主会话**：

```bash
curl -X POST http://127.0.0.1:18789/hooks/wake   -H 'Authorization: Bearer SECRET'   -H 'Content-Type: application/json'   -d '{"text":"New email received","mode":"now"}'
```

**启动一次 isolated agent run**：

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent   -H 'Authorization: Bearer SECRET'   -H 'Content-Type: application/json'   -d '{"message":"Summarize inbox","name":"Email","model":"openai/gpt-5.4-mini"}'
```

适合场景：

- 表单提交后触发摘要
- 新邮件 / 新工单到来后做初筛
- CI 成功后让 OpenClaw 生成更新说明

### 10.4.2 复杂编排：`webhooks` plugin + Task Flow

如果你需要**多步骤、可追踪、能继续推进**的工作流，直接上 `webhooks` plugin。

官方配置示例：

```json
{
  "plugins": {
    "entries": {
      "webhooks": {
        "enabled": true,
        "config": {
          "routes": {
            "zapier": {
              "path": "/plugins/webhooks/zapier",
              "sessionKey": "agent:main:main",
              "secret": {
                "source": "env",
                "provider": "default",
                "id": "OPENCLAW_WEBHOOK_SECRET"
              },
              "controllerId": "webhooks/zapier",
              "description": "Zapier TaskFlow bridge"
            }
          }
        }
      }
    }
  }
}
```

创建 flow：

```bash
curl -X POST https://gateway.example.com/plugins/webhooks/zapier   -H 'Content-Type: application/json'   -H 'Authorization: Bearer YOUR_SHARED_SECRET'   -d '{"action":"create_flow","goal":"Review inbound queue"}'
```

在 flow 中再创建子任务：

```json
{
  "action": "run_task",
  "flowId": "flow_123",
  "runtime": "acp",
  "childSessionKey": "agent:main:acp:worker",
  "task": "Inspect the next message batch"
}
```

适合场景：

- Zapier / n8n / Make 触发多步任务
- 客服工单分诊
- 线索筛选 + 跟进 + 汇总
- 周报流水线、内容审核流水线

### 10.4.3 Notion 现在怎么接才对？

本章旧内容里那一大段“Notion Skill 全套配置”最大的问题，不是 Notion 不能接，而是**默认路线已经不对了**。

现在更稳的做法是：

1. **如果只是把 OpenClaw 结果写入 Notion**：优先用 Zapier / n8n / 自家中间层，通过 webhook 接入
2. **如果要形成可持续的内部工作流**：用 `webhooks` plugin 把外部事件绑定到 Task Flow
3. **如果你需要深度定制**：自己写 plugin，不要依赖历史第三方 Skill 名称

也就是说：

- **Notion 依然能接**
- 但它不再应该占据“官方默认主线”的位置
- 现在的默认主线是 **Infer + Hooks/Webhooks + Task Flow + Plugin**

---

## 10.5 这一章最容易踩的坑

### 坑 1：继续照着旧 Skill 名称安装

处理方式：

- 先看官方 docs / release notes
- 优先确认能力是否已经内建到 `infer` 或 agent 工具中
- 只有官方路线没有时，才继续搜社区插件

### 坑 2：把媒体能力当同步脚本理解

- 图片和 TTS 多数是同步返回
- 视频和音乐常常是后台任务
- 这两类长任务最好配合 `openclaw tasks list`、`openclaw tasks show` 观察状态

### 坑 3：`--model` 没写 provider 前缀

下列命令场景里，建议始终写全：

- `image describe`
- `audio transcribe`
- `video describe`
- 任何你明确指定 provider 的脚本

正确示例：

```bash
openclaw infer audio transcribe --file ./memo.m4a --model openai/whisper-1 --json
```

### 坑 4：把 secret 直接写死在仓库里

官方文档已经明确支持 SecretRef。优先顺序：

1. `env`
2. `file`
3. `exec`

不要把 webhook secret、provider token、第三方 API key 直接写进公开仓库。

---

## 10.6 本章实践建议

如果你现在就要把 OpenClaw 接到业务里，推荐按下面的顺序走：

1. **先跑通 `openclaw onboard` 与 `openclaw infer`**
2. **再配图片 / 视频 / TTS 的默认模型**
3. **轻量触发用 `hooks`**
4. **多步骤流程用 `webhooks` plugin + `Task Flow`**
5. **本地多媒体深度编排再上 `ComfyUI`**

---

## 10.7 官方参考

- GitHub Releases：https://github.com/openclaw/openclaw/releases
- Inference CLI：https://docs.openclaw.ai/cli/infer
- Models CLI：https://docs.openclaw.ai/cli/models
- Webhooks Plugin：https://docs.openclaw.ai/plugins/webhooks
- Scheduled Tasks：https://docs.openclaw.ai/automation/cron-jobs
- Task Flow：https://docs.openclaw.ai/automation/taskflow
