# 附录C 配置模板与自定义参考

> 💡 **本附录目标**：提供 openclaw.json 的常用配置片段，供你在引导向导完成后按需自定义。所有模板均基于官方文档（https://docs.openclaw.ai/gateway/configuration-examples）验证，适用于v2026.3.7+版本。
>
> ⚠️ **新手请注意**：你不需要手动编辑配置文件即可上手使用OpenClaw。直接运行引导向导，它会交互式引导你完成全部配置并自动生成配置文件。本附录的模板适用于向导完成后的进一步自定义。
>
> 配置文件路径：~/.openclaw/openclaw.json（JSON5格式，支持注释和尾逗号）

## 📋 目录

-   C.1 新手上手（推荐方式）
-   C.2 多模型配置
-   C.3 多平台集成配置
-   C.4 Skills配置
-   C.5 定时任务（Cron）
-   C.6 多Agent配置
-   C.7 安全配置
-   C.8 完整示例：超级个体配置
-   C.9 快速部署脚本

## C.1 新手上手（推荐方式）

### 1. 最快上手：直接运行引导向导（强烈推荐）

> ⚠️ **新手不要手动编辑配置文件。** OpenClaw采用严格的配置校验，一个字段名拼错或结构不对，Gateway就会拒绝启动。引导向导会自动生成正确的配置文件。

    # 第1步：运行引导向导（会引导你选择模型、输入API Key、配置通道等）
    openclaw onboard

    # 第2步：安装并启动守护进程
    openclaw daemon install
    openclaw daemon start

    # 第3步：打开控制面板，开始使用
    openclaw dashboard

向导会引导你完成以下全部配置：
- 模型选择与API Key输入（支持Anthropic/OpenAI/DeepSeek/Kimi等）
- 通道配置（WhatsApp/Telegram等）
- Gateway认证Token生成
- 工具权限设置
- Skills推荐安装

向导完成后，配置文件自动保存在 `~/.openclaw/openclaw.json`。如需进一步自定义，可使用以下方式修改：

    # 方式1：交互式配置向导（推荐）
    openclaw configure

    # 方式2：命令行单项修改
    openclaw config set agents.defaults.heartbeat.every "30m"
    openclaw config set session.reset.atHour 4

    # 方式3：打开配置文件直接编辑
    openclaw config file   # 显示配置文件路径，用编辑器打开即可

    # 方式4：通过控制面板Web UI修改
    openclaw dashboard     # 打开后在 Config 标签页可视化编辑

### 2. 向导完成后的常用自定义

以下是向导完成后，你可能想要额外调整的常见配置项。使用 `openclaw config set` 命令逐项修改即可，无需手动编辑JSON文件：

    # 设置中文身份
    openclaw config set identity.name "小龙虾"
    openclaw config set identity.theme "专业高效的AI助手"
    openclaw config set identity.emoji "🦞"

    # 开启心跳（每30分钟主动检查一次）
    openclaw config set agents.defaults.heartbeat.every "30m"
    openclaw config set agents.defaults.heartbeat.target "last"

    # 设置会话每日自动重置（凌晨4点，闲置2小时后）
    openclaw config set session.dmScope "per-channel-peer"
    openclaw config set session.reset.mode "daily"
    openclaw config set session.reset.atHour 4
    openclaw config set session.reset.idleMinutes 120

    # 确保工具权限为完整模式（否则只能聊天不能干活）
    openclaw config set agents.defaults.tools.profile "full"

    # 修改后重启生效
    openclaw daemon restart

## C.2 多模型配置

### 1. 国产模型组合（省钱方案）

> ⚠️ 模型认证通过 `openclaw models auth add` 命令交互式配置，API Key 不直接写在配置文件中。以下配置设置模型选择和备用策略。

    {
      agents: {
        defaults: {
          model: {
            // 主模型：DeepSeek（最便宜）
            primary: "deepseek/deepseek-chat",
            // 备用模型：Kimi长文档 → GLM兜底
            fallbacks: [
              "moonshot/moonshot-v1-128k",
              "zhipu/glm-4-flash",
            ],
          },
          models: {
            "deepseek/deepseek-chat": { alias: "ds" },
            "moonshot/moonshot-v1-128k": { alias: "kimi" },
            "zhipu/glm-4-flash": { alias: "glm" },
          },
        },
      },
    }

**配置API Key（命令行执行）**：

    # 添加DeepSeek认证
    openclaw models auth add
    # 选择 deepseek → 输入 API Key

    # 添加Kimi认证
    openclaw models auth add
    # 选择 moonshot → 输入 API Key

    # 添加智谱GLM认证
    openclaw models auth add
    # 选择 zhipu → 输入 API Key

**在对话中切换模型**：

    /model ds      # 切换到DeepSeek
    /model kimi    # 切换到Kimi
    /model glm     # 切换到GLM

**成本估算**：
- 日常对话：DeepSeek（约0.001元/1K tokens）
- 长文档：Kimi 128K（约0.012元/1K tokens）
- 月均成本：5-30元

### 2. 国际模型配置

    {
      agents: {
        defaults: {
          model: {
            primary: "anthropic/claude-sonnet-4-5",
            fallbacks: [
              "openai/gpt-5.2",
              "anthropic/claude-opus-4-6",
            ],
          },
          imageModel: {
            primary: "anthropic/claude-sonnet-4-5",
          },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-5": { alias: "sonnet" },
            "openai/gpt-5.2": { alias: "gpt" },
          },
        },
      },
    }

### 3. 中转API配置

> 中转API使用OpenAI兼容格式，通过环境变量设置Key和BaseURL。

    {
      env: {
        vars: {
          OPENAI_API_KEY: "your-relay-api-key",
          OPENAI_BASE_URL: "https://tryallapi.com/v1",
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "openai/gpt-4o-mini",
            fallbacks: ["openai/gpt-4o"],
          },
        },
      },
    }

**优势**：
- ✅ 一个API密钥访问多个模型
- ✅ 国内访问速度快
- ✅ 成本更低

### 4. 本地模型（完全免费）

    {
      agents: {
        defaults: {
          model: {
            primary: "ollama/qwen2.5:32b",
            fallbacks: ["ollama/llama3.1:8b"],
          },
        },
      },
    }

**前提**：需先安装Ollama并拉取模型：

    curl -fsSL https://ollama.ai/install.sh | sh
    ollama pull qwen2.5:32b

## C.3 多平台集成配置

### 1. 飞书Bot

    {
      channels: {
        feishu: {
          enabled: true,
          appId: "cli_your_app_id",
          appSecret: "your_app_secret",
          dmPolicy: "pairing",
        },
      },
    }

> 飞书接入需安装插件：`openclaw plugins install @m1heng-clawd/feishu`，详见本书第12章。

### 2. 企业微信Bot

    {
      channels: {
        wework: {
          enabled: true,
          corpId: "ww_your_corp_id",
          agentSecret: "your_agent_secret",
          dmPolicy: "pairing",
        },
      },
    }

> 企业微信接入需安装插件：`openclaw plugins install @m1heng-clawd/wework`，详见本书第13章。

### 3. 钉钉Bot

    {
      channels: {
        dingtalk: {
          enabled: true,
          appKey: "your_app_key",
          appSecret: "your_app_secret",
          dmPolicy: "pairing",
        },
      },
    }

> 详见本书第13章。

### 4. Telegram Bot

    {
      channels: {
        telegram: {
          enabled: true,
          botToken: "1234567890:ABCdefGHIjklMNOpqrsTUVwxyz",
          dmPolicy: "pairing",
          allowFrom: ["your_telegram_user_id"],
          groups: { "*": { requireMention: true } },
        },
      },
    }

### 5. 多平台同时接入

    {
      channels: {
        telegram: {
          enabled: true,
          botToken: "your_telegram_token",
          dmPolicy: "pairing",
          groups: { "*": { requireMention: true } },
        },
        whatsapp: {
          dmPolicy: "pairing",
          allowFrom: ["+86138xxxxxxxx"],
          groups: { "*": { requireMention: true } },
        },
        discord: {
          enabled: true,
          token: "your_discord_token",
          dm: { enabled: true },
        },
      },
    }

## C.4 Skills配置

> ⚠️ Skills通过 `clawhub install <slug>` 安装，不在配置文件中列出安装列表。配置文件中只对已安装的Skills进行个性化配置（如API Key、启停等）。

    {
      skills: {
        entries: {
          "nano-banana-pro": {
            enabled: true,
            env: {
              GEMINI_API_KEY: "your-gemini-key",
            },
          },
          "brave-search": {
            enabled: true,
            env: {
              BRAVE_API_KEY: "your-brave-key",
            },
          },
          "tavily-search": {
            enabled: true,
            env: {
              TAVILY_API_KEY: "your-tavily-key",
            },
          },
        },
      },
    }

**安装Skills（命令行执行）**：

    clawhub install brave-search nano-banana-pro summarize \
      find-skills self-improving proactive-agent skill-vetter

    # 查看已安装Skills
    openclaw skills list

## C.5 定时任务（Cron）

> ⚠️ 定时任务通过 `openclaw cron add` 命令创建，不在配置文件中定义任务列表。配置文件中只设置Cron的全局参数。

**配置文件中的Cron全局设置**：

    {
      cron: {
        enabled: true,
        maxConcurrentRuns: 2,
        sessionRetention: "24h",
      },
    }

**创建定时任务（命令行执行）**：

    # 每天早上9点推送AI行业日报
    openclaw cron add \
      --name "daily-ai-report" \
      --cron "0 9 * * *" \
      --tz "Asia/Shanghai" \
      --session isolated \
      --message "生成今天的AI行业日报" \
      --deliver --channel feishu

    # 每周五18点生成周报
    openclaw cron add \
      --name "weekly-summary" \
      --cron "0 18 * * 5" \
      --tz "Asia/Shanghai" \
      --session isolated \
      --message "总结本周工作，生成周报" \
      --deliver --channel feishu

    # 查看/删除定时任务
    openclaw cron list
    openclaw cron remove <job-id>

**Cron表达式速查**：
- `0 9 * * *` — 每天9:00
- `0 18 * * 5` — 每周五18:00
- `0 */2 * * *` — 每2小时
- `*/30 * * * *` — 每30分钟
- `0 9 * * 1-5` — 工作日每天9:00

## C.6 多Agent配置

    {
      agents: {
        defaults: {
          workspace: "~/.openclaw/workspace",
          model: {
            primary: "anthropic/claude-sonnet-4-5",
          },
        },
        list: [
          {
            id: "main",
            default: true,
            workspace: "~/.openclaw/workspace-main",
          },
          {
            id: "content",
            workspace: "~/.openclaw/workspace-content",
            model: {
              primary: "anthropic/claude-opus-4-6",
            },
          },
          {
            id: "code",
            workspace: "~/.openclaw/workspace-code",
            model: {
              primary: "deepseek/deepseek-coder",
            },
          },
        ],
      },
      bindings: [
        { agentId: "content", match: { channel: "telegram" } },
        { agentId: "code", match: { channel: "discord" } },
      ],
    }

## C.7 安全配置

### Gateway认证（v2026.3.7+必须配置）

    {
      gateway: {
        port: 18789,
        auth: {
          mode: "token",
          token: "your-secret-token-here",
        },
      },
    }

**生成安全Token**：

    openssl rand -hex 32

> ⚠️ 从v2026.3.7起，Gateway认证为强制要求。

### 沙箱配置（Docker隔离）

    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",
            scope: "agent",
          },
        },
      },
    }

### 工具权限控制

    {
      agents: {
        defaults: {
          tools: {
            profile: "full",     // full | coding | messaging
          },
        },
      },
    }

## C.8 完整示例：超级个体配置

    // ~/.openclaw/openclaw.json
    {
      identity: {
        name: "小龙虾",
        theme: "专业高效的AI超级个体助手",
        emoji: "🦞",
      },
      gateway: {
        port: 18789,
        auth: { mode: "token", token: "替换为你的随机Token" },
      },
      agents: {
        defaults: {
          workspace: "~/.openclaw/workspace",
          userTimezone: "Asia/Shanghai",
          model: {
            primary: "deepseek/deepseek-chat",
            fallbacks: ["moonshot/moonshot-v1-128k", "zhipu/glm-4-flash"],
          },
          models: {
            "deepseek/deepseek-chat": { alias: "ds" },
            "moonshot/moonshot-v1-128k": { alias: "kimi" },
            "zhipu/glm-4-flash": { alias: "glm" },
          },
          heartbeat: { every: "30m", target: "last" },
          tools: { profile: "full" },
        },
      },
      channels: {
        feishu: {
          enabled: true,
          appId: "cli_your_app_id",
          appSecret: "your_app_secret",
          dmPolicy: "pairing",
        },
      },
      skills: {
        entries: {
          "brave-search": {
            enabled: true,
            env: { BRAVE_API_KEY: "your-brave-key" },
          },
        },
      },
      session: {
        dmScope: "per-channel-peer",
        reset: { mode: "daily", atHour: 4, idleMinutes: 120 },
      },
      cron: { enabled: true, maxConcurrentRuns: 2 },
    }

**配置完成后执行**：

    openclaw models auth add
    clawhub install skill-vetter find-skills self-improving proactive-agent \
      brave-search summarize nano-banana-pro
    openclaw cron add --name "daily-report" --cron "0 9 * * *" \
      --tz "Asia/Shanghai" --session isolated \
      --message "生成今日AI行业日报" --deliver --channel feishu
    openclaw daemon start
    openclaw dashboard

## C.9 快速部署脚本

### 一键配置脚本（Mac/Linux）

    #!/bin/bash
    set -e
    echo "🦞 OpenClaw 快速配置开始..."
    mkdir -p ~/.openclaw/workspace
    TOKEN=$(openssl rand -hex 32)

    cat > ~/.openclaw/openclaw.json << EOF
    {
      gateway: { port: 18789, auth: { mode: "token", token: "$TOKEN" } },
      identity: { name: "小龙虾", theme: "专业高效的AI助手", emoji: "🦞" },
      agents: {
        defaults: {
          workspace: "~/.openclaw/workspace",
          userTimezone: "Asia/Shanghai",
          tools: { profile: "full" },
        },
      },
      session: { dmScope: "per-channel-peer", reset: { mode: "daily", atHour: 4 } },
      cron: { enabled: true },
    }
    EOF

    echo "✅ 配置文件已生成（Token: $TOKEN）"
    openclaw onboard
    clawhub install skill-vetter find-skills self-improving proactive-agent
    openclaw daemon install
    openclaw daemon start
    echo "✅ 完成！运行 openclaw dashboard 打开控制面板"

## 📚 相关资源

-   官方配置文档：https://docs.openclaw.ai/gateway/configuration
-   官方配置示例：https://docs.openclaw.ai/gateway/configuration-examples
-   官方配置字段参考：https://docs.openclaw.ai/gateway/configuration-reference
-   本书第2-4章：安装部署与配置详解
-   附录A：命令速查表

**提示**：本模板基于v2026.3.7+版本验证。OpenClaw配置采用严格校验，未知字段会导致Gateway拒绝启动。如遇启动失败，运行 `openclaw doctor` 查看具体问题。
