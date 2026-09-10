> 📖 **纸质书《OpenClaw超级个体实操手册》已上市！** 清华大学出版社出版，在开源教程基础上全面重写+逐条验证。🛒 [京东专属购买链接（¥42，原价¥59.8）](https://item.jd.com/14669463.html)

# 第11章节 高级配置（模型、记忆、审批与性能）

> 本章目标：把 OpenClaw 当前稳定版里最重要的高级配置讲清楚，包括模型选择、认证、媒体默认模型、Active Memory、Memory Wiki，以及执行审批与安全边界。

---

## 版本基线

- **当前稳定版**：`v2026.9.3`（2026-09-08 发布）
- 本章默认按 `v2026.9.3` 写；遇到旧 `openai-codex/*` / OpenProse / `clawhub install …` 请先当历史参考

---

## 先给小白的阅读说明

### 这一章到底解决什么问题

很多新手一看到 `openclaw.json`、`models.json`、`AGENTS.md`、provider auth，就会马上开始手改配置。结果往往是：

- 不知道哪里才是当前生效值
- 改了配置却没改到真正的默认模型
- 认证、回退模型、媒体模型混在一起

这一章就是帮你把这些东西拆开。

### 如果你只想先把配置跑通，先看这几节

- **先看 `11.1`**：知道推荐顺序
- **再看 `11.2`**：把主模型、回退模型、认证配通
- **然后看 `11.3`**：分清会话模型和媒体模型
- **最后看 `11.6`**：知道安全边界，不要误配审批

### 开始前的最低前提

你不需要先懂所有 JSON 字段，但至少要满足：

1. 已跑过 `openclaw onboard`
2. 至少有一个 provider 能成功登录
3. 愿意先用 CLI 看状态，再决定是否手改配置文件

### 小白最容易犯的 3 个错

- 一上来就手改 JSON，不先看 `models status`
- 把 `imageModel` 和 `imageGenerationModel` 当成同一个东西
- 看到配置项很多，就以为“全都配上才算完整”

---

## 11.1 推荐配置路径：先向导，后精调

OpenClaw 2026.4 之后，高级配置的推荐顺序不是“先手改 JSON”，而是：

1. `openclaw onboard`
2. `openclaw models auth add|login`
3. `openclaw models status|list|set`
4. 需要时再手动改 `openclaw.json` / `models.json`

最短起步命令：

```bash
openclaw onboard
openclaw models status
openclaw models list
openclaw models set openai/gpt-5.4
```

如果你不知道当前到底配成了什么，先看状态，不要猜：

```bash
openclaw models status --probe
openclaw status
```

### 看到什么算当前配置已经健康

如果你看到下面这些现象，说明配置已经进入“可继续优化”的状态：

- `openclaw models status` 能看到 `primary` 和 `fallbacks`
- `openclaw models status --probe` 没报 provider 不可用或 token 失效
- `openclaw status` 没有明显的 gateway / auth 阻塞错误

`models status --probe` 会做真实探测，可能消耗 token，但它最适合确认以下问题：

- token 是不是过期了
- provider 看起来已配置，但是否真能用
- 当前 primary / fallbacks 最终到底解析成了什么

---

## 11.2 模型与认证：当前该怎么配

### 11.2.1 先理解三层关系

OpenClaw 现在的模型配置可以简化成三层：

1. **主模型**：`agents.defaults.model.primary`
2. **回退模型**：`agents.defaults.model.fallbacks`
3. **认证与 provider 状态**：通过 `models auth`、环境变量和 auth profile 管理

### 11.2.2 常用命令

如果你是第一次接触这些命令，可以把它们理解成下面 4 类：

- `status` / `list`：先看现状
- `set`：设置主模型
- `fallbacks add`：给主模型准备兜底
- `auth login`：解决“为什么看得到 provider 却用不了”

```bash
# 查看当前状态
openclaw models status
openclaw models list

# 设置主模型
openclaw models set openai/gpt-5.4

# 增加回退模型
openclaw models fallbacks add anthropic/claude-sonnet-4-5
openclaw models fallbacks add google/gemini-2.5-pro

# 设置图像理解兜底模型
openclaw models set-image openai/gpt-4.1-mini
openclaw models image-fallbacks add google/gemini-2.5-pro
```

### 11.2.3 当前值得注意的 provider 变化

根据官方 release notes（`2026.8.1` OpenClaw 2.0 → `v2026.9.3`）：

- **`2026.8.1` 起**：会话/转录迁入 SQLite；升级前务必备份 `~/.openclaw`
- **OpenAI 路由迁移（Breaking）**：旧的 `codex/*`、`openai-codex/*` 应迁移为规范的 `openai/*`；执行 `openclaw doctor --fix`
- **OpenProse 已移除（Breaking）**：内置 OpenProse 插件与 `/prose` 已下线；清配置用 `openclaw doctor --fix`，需要时再按官方文档安装上游 Agent Skill
- **`v2026.9.3`**：更新预演/恢复更稳；Skill Workshop 改为按 Agent 持久化集合；Node 要求提升到 **24.16+ / 26.1+**
- 具体可选模型 ID 请以 `openclaw models list` / Control UI 目录为准，不要照抄过期型号

如果你要做编程工作流，当前更推荐先走：

- `openai/...`（ChatGPT/Codex 订阅或 API；用 `models auth login --provider openai`）
- `anthropic/...`
- `google/...`

### 11.2.4 认证方式建议

```bash
# 交互式添加 provider 认证
openclaw models auth add

# 直接对某个 provider 发起登录
openclaw models auth login --provider openai --set-default
openclaw models auth login --provider anthropic --method cli --set-default
```

如果你是自托管 provider 或 OpenAI-compatible endpoint，优先保证：

- `baseUrl` 可达
- API key 已注入
- 仅在可信私网环境里启用 `models.providers.*.request.allowPrivateNetwork`

---

## 11.3 媒体默认模型：不要再把图片、视频、音乐混着配

官方现在把“会话模型”和“媒体生成模型”拆得很清楚。你至少要分清 5 类默认模型：

- `agents.defaults.model`
- `agents.defaults.imageModel`
- `agents.defaults.imageGenerationModel`
- `agents.defaults.videoGenerationModel`
- `agents.defaults.musicGenerationModel`

推荐配置示例：

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "openai/gpt-5.4",
        "fallbacks": [
          "anthropic/claude-sonnet-4-5",
          "google/gemini-2.5-pro"
        ]
      },
      "imageModel": {
        "primary": "openai/gpt-4.1-mini"
      },
      "imageGenerationModel": {
        "primary": "openai/gpt-image-1"
      },
      "videoGenerationModel": {
        "primary": "google/veo-3.1-fast-generate-preview",
        "fallbacks": [
          "qwen/wan2.6-r2v-flash"
        ]
      },
      "musicGenerationModel": {
        "primary": "google/lyria-3-clip-preview"
      }
    }
  }
}
```

几点说明：

1. `imageModel` 用于“主模型不能直接看图”时的图像理解兜底
2. `imageGenerationModel` 专门给 `image_generate` 用
3. `videoGenerationModel` 和 `musicGenerationModel` 只影响共享媒体工具
4. 如果你没有显式配置，OpenClaw 也会尝试根据已认证 provider 自动推断默认值，但生产环境不建议完全依赖自动推断

---

## 11.4 Active Memory：让记忆在回复前主动介入

`v2026.4.12` 的一个核心变化，是 **Active Memory plugin** 进入主线能力：它会在主回复前先跑一次受限的记忆子代理，用 `memory_search` / `memory_get` 拉回和当前会话相关的偏好、上下文和历史事实。

### 11.4.1 推荐起步配置

```json
{
  "plugins": {
    "entries": {
      "active-memory": {
        "enabled": true,
        "config": {
          "agents": ["main"],
          "allowedChatTypes": ["direct"],
          "modelFallbackPolicy": "default-remote",
          "queryMode": "recent",
          "promptStyle": "balanced",
          "timeoutMs": 15000,
          "maxSummaryChars": 220,
          "persistTranscripts": false,
          "logging": true
        }
      }
    }
  }
}
```

### 11.4.2 什么时候该开，什么时候别开

**适合开启**：

- 私聊型、长期关系型助手
- 高频重复协作
- 需要记住偏好、习惯、上下文的场景

**不适合默认开启**：

- 纯自动化 worker
- 一次性 API 任务
- 强确定性流水线
- 你不希望隐藏个性化影响输出的场景

### 11.4.3 调试方法

```bash
openclaw memory status --deep
```

在聊天里可以用 `/verbose on` 看 Active Memory 的状态行。调优优先从这几个参数入手：

- `queryMode`
- `promptStyle`
- `timeoutMs`
- `maxSummaryChars`

---

## 11.5 Memory Wiki：把长期记忆变成“可维护的知识层”

`memory-wiki` 是官方内建 plugin，它**不是用来替代 memory-core 的**，而是把长期记忆编译成结构化 wiki 层，适合：

- 项目知识沉淀
- 客户画像 / 产品知识整理
- 知识冲突排查
- 长周期研究类任务

### 11.5.1 官方推荐理解方式

- **memory-core / QMD / dreaming**：负责 recall、promotion、search、dreaming
- **memory-wiki**：负责把 durable memory 编译成可导航的 wiki 页面与结构化 claim/evidence

### 11.5.2 推荐配置示例

```json
{
  "plugins": {
    "entries": {
      "memory-wiki": {
        "enabled": true,
        "config": {
          "vaultMode": "isolated",
          "vault": {
            "path": "~/.openclaw/wiki/main",
            "renderMode": "obsidian"
          },
          "bridge": {
            "enabled": false,
            "readMemoryArtifacts": true,
            "indexDreamReports": true,
            "indexDailyNotes": true,
            "indexMemoryRoot": true,
            "followMemoryEvents": true
          },
          "ingest": {
            "autoCompile": true,
            "maxConcurrentJobs": 1,
            "allowUrlIngest": true
          },
          "search": {
            "backend": "shared",
            "corpus": "wiki"
          },
          "context": {
            "includeCompiledDigestPrompt": false
          },
          "render": {
            "preserveHumanBlocks": true,
            "createBacklinks": true,
            "createDashboards": true
          }
        }
      }
    }
  }
}
```

### 11.5.3 常用命令

```bash
openclaw wiki init
openclaw wiki status
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "customer onboarding"
openclaw wiki get entity.alpha
```

推荐工作流：

1. 先让 memory-core 跑稳
2. 再开 `memory-wiki`
3. 默认优先 `isolated` 模式
4. 如果你明确需要从现有 memory artifact 构建 wiki，再启用 `bridge`

---

## 11.6 执行审批、安全与自托管边界

### 11.6.1 不要只看 `exec`，要同时看 approval 文件与 tool policy

`v2026.4.12` 增加了本地 `exec-policy` 命令，目标是把 `tools.exec.*` 配置和本机审批文件同步起来。但实际落地时，你仍然要同时理解三层东西：

1. `tools.exec.*`
2. `~/.openclaw/exec-approvals.json`
3. agent 的 tool policy / allowlist

最实用的检查命令：

```bash
openclaw exec-policy show
openclaw approvals get
openclaw approvals get --gateway
```

如果你要给某些命令做 allowlist：

```bash
openclaw approvals allowlist add "~/Projects/**/bin/rg"
openclaw approvals allowlist add --agent main "/usr/bin/uname"
```

### 11.6.2 Hook / Webhook 的安全底线

- Hook token 和 gateway token 分开
- 不要把 hook 暴露在根路径 `/`
- `hooks.path` 保持独立子路径
- Webhook 路由尽量绑定最小 `sessionKey`
- Secret 优先走 `env` / `file` / `exec`，不要写死到仓库里

### 11.6.3 自托管 provider 的私网配置

`v2026.4.12` 官方加入了 `models.providers.*.request.allowPrivateNetwork`，用于你明确知道自己在访问可信私网 provider 时放开限制。这个开关非常有用，但也只应该用于**你完全控制的私网服务**。

适用场景：

- 自己的 LM Studio / OpenAI-compatible endpoint
- 内网部署的代理层
- VPN / Tailnet 内的推理网关

不适用场景：

- 公网随便开的代理地址
- 不明来源共享网关
- 混合代理环境里没有明确边界的 endpoint

---

## 11.7 性能调优：2026.4 值得关注的点

### 11.7.1 先做“结构性调优”，再做“参数性调优”

最有效的顺序通常是：

1. 先把主模型 / 回退链配清楚
2. 再把媒体模型单独拆开
3. 再开 Active Memory / Memory Wiki
4. 最后才调 thinking、context、fallback 数量

### 11.7.2 本地模型用户的新补充

历史 `v2026.4.15-beta.1` 里曾新增过一个本地模型实验参数：

```json
{
  "agents": {
    "defaults": {
      "experimental": {
        "localModelLean": true
      }
    }
  }
}
```

它会在弱本地模型场景下去掉一些重量级默认工具，降低提示词体积。**注意这是历史 beta 能力**，在 `v2026.9.3` 环境里请先查官方配置文档和 `openclaw config schema`，不要直接照抄到生产主环境。

### 11.7.3 本章推荐的检查顺序

```bash
openclaw status
openclaw models status --probe
openclaw memory status --deep
openclaw wiki status
openclaw approvals get
openclaw security audit
```

---

## 11.8 本章实践建议

如果你正在配一套长期可用的 OpenClaw 环境，最稳的顺序是：

1. 先配好 `models auth`、主模型和回退链
2. 再单独配图片 / 视频 / 音乐模型
3. 需要长期关系型助手时再开 Active Memory
4. 需要“可维护知识层”时再开 Memory Wiki
5. 最后再收紧 exec approvals、hook token 和私网 provider 边界

---

## 11.9 官方参考

- GitHub Releases：https://github.com/openclaw/openclaw/releases
- Models CLI：https://docs.openclaw.ai/cli/models
- Model Concepts：https://docs.openclaw.ai/concepts/models
- Inference CLI：https://docs.openclaw.ai/cli/infer
- Active Memory：https://docs.openclaw.ai/concepts/active-memory
- Memory Wiki：https://docs.openclaw.ai/plugins/memory-wiki
- Exec Approvals：https://docs.openclaw.ai/cli/approvals
