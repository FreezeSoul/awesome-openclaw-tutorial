# 附录A 命令速查表

> 💡 **本附录目标**：提供OpenClaw常用命令的快速参考。所有命令均基于官方CLI文档（https://docs.openclaw.ai/cli）验证，适用于v2026.3.7+版本。

## 📋 目录

-   A.1 安装与初始化
-   A.2 配置管理（config）
-   A.3 Gateway与守护进程（daemon）
-   A.4 状态与诊断
-   A.5 通道管理（channels）
-   A.6 模型管理（models）
-   A.7 Skills管理
-   A.8 插件管理（plugins）
-   A.9 日志与会话
-   A.10 定时任务（cron）
-   A.11 消息发送（message）
-   A.12 安全与备份
-   A.13 重置与卸载
-   A.14 常用场景组合
-   A.15 配置文件路径

## A.1 安装与初始化

    # 全局安装OpenClaw
    npm install -g openclaw@latest

    # 首次引导向导（推荐）
    openclaw onboard

    # 引导向导（高级模式，完整控制每个步骤）
    openclaw onboard --advanced

    # 重新运行引导向导（重置配置+凭据+会话）
    openclaw onboard --reset

    # 交互式配置向导（已安装后修改配置）
    openclaw configure

    # 查看版本
    openclaw --version

    # 查看帮助
    openclaw --help

    # 查看子命令帮助
    openclaw config --help

## A.2 配置管理（config）

> ⚠️ `openclaw config` 不带子命令等同于 `openclaw configure`（打开交互式向导）。
> config 仅支持 `get`、`set`、`unset`、`file`、`validate` 五个子命令。

    # 查看特定配置项
    openclaw config get <path>
    openclaw config get gateway.port
    openclaw config get agents.defaults.workspace
    openclaw config get agents.list[0].id

    # 设置配置项（值自动解析为JSON5，否则视为字符串）
    openclaw config set <path> <value>
    openclaw config set gateway.port 19001 --strict-json
    openclaw config set agents.defaults.heartbeat.every "2h"
    openclaw config set channels.whatsapp.groups '["*"]' --strict-json

    # 删除配置项
    openclaw config unset <path>
    openclaw config unset tools.web.search.apiKey

    # 查看配置文件路径
    openclaw config file

    # 校验配置文件
    openclaw config validate

> ⚠️ **不存在的命令**：`config list`、`config reset`、`config export`、`config import`、`config delete` 均不是有效子命令。查看全部配置请直接打开配置文件：`openclaw config file`。重置配置请使用 `openclaw reset`。

## A.3 Gateway与守护进程（daemon）

> ⚠️ Gateway的启停通过 `daemon` 命令管理，而非 `gateway start/stop`。

    # 安装系统服务（macOS: LaunchAgent / Linux: systemd）
    openclaw daemon install

    # 启动守护进程
    openclaw daemon start

    # 停止守护进程
    openclaw daemon stop

    # 重启守护进程（配置变更后执行）
    openclaw daemon restart

    # 查看守护进程状态
    openclaw daemon status

    # 卸载系统服务
    openclaw daemon uninstall

    # 查看守护进程日志
    openclaw daemon logs

    # 直接运行Gateway（前台模式，适合调试）
    openclaw gateway

    # Gateway运行参数
    openclaw gateway --port 18789 --verbose

    # 查询运行中的Gateway健康状态
    openclaw gateway health

    # 查询Gateway详细状态
    openclaw gateway status

    # 探测Gateway（附加检查）
    openclaw gateway probe

    # 发现局域网内的Gateway（Bonjour/mDNS）
    openclaw gateway discover

    # 调用Gateway RPC方法
    openclaw gateway call <method>

    # 打开控制面板（Web UI）
    openclaw dashboard

## A.4 状态与诊断

    # 查看整体运行状态
    openclaw status

    # 健康检查
    openclaw health

    # 综合诊断与修复建议
    openclaw doctor

    # 自动执行修复
    openclaw doctor --yes

    # 非交互模式诊断
    openclaw doctor --non-interactive

    # 深度扫描（检查系统服务等）
    openclaw doctor --deep

    # 启动TUI终端界面
    openclaw tui

    # 搜索官方文档
    openclaw docs <关键词>

## A.5 通道管理（channels）

    # 列出已配置的通道
    openclaw channels list

    # 查看通道状态（含连接健康检查）
    openclaw channels status

    # 通道状态（附加探测）
    openclaw channels status --probe

    # 添加通道
    openclaw channels add <channel>

    # 移除通道
    openclaw channels remove <channel>

    # 通道登录
    openclaw channels login <channel>

    # 通道登出
    openclaw channels logout <channel>

    # 配对管理（WhatsApp/Telegram DM配对）
    openclaw pairing list <channel>
    openclaw pairing approve <channel> <code>

## A.6 模型管理（models）

    # 列出已配置的模型
    openclaw models list

    # 查看模型状态
    openclaw models status

    # 切换默认模型
    openclaw models set <model>
    openclaw models set anthropic/claude-sonnet-4-5

    # 设置图片模型
    openclaw models set-image <model>

    # 添加认证（API Key / OAuth / setup-token）
    openclaw models auth add
    openclaw models auth login --provider openai --set-default

    # 模型别名管理
    openclaw models aliases list
    openclaw models aliases add <alias> <model>
    openclaw models aliases remove <alias>

    # 备用模型管理
    openclaw models fallbacks list
    openclaw models fallbacks add <model>
    openclaw models fallbacks remove <model>
    openclaw models fallbacks clear

    # 图片模型备用
    openclaw models image-fallbacks list
    openclaw models image-fallbacks add <model>
    openclaw models image-fallbacks remove <model>

    # 扫描可用模型
    openclaw models scan

    # 认证优先级
    openclaw models auth order get
    openclaw models auth order set <providers...>

## A.7 Skills管理

> ⚠️ **2026.9 主线**：优先使用 `openclaw skills` 查看/检查/管理；`clawhub install …` 仅作历史参考，不要默认照抄。

### openclaw skills（查看与检查）

    # 列出所有Skills（内置+工作区+托管）
    openclaw skills list

    # 仅列出符合条件可加载的Skills
    openclaw skills list --eligible

    # 查看Skills详情
    openclaw skills info <skill-name>

    # 检查Skills依赖是否满足
    openclaw skills check

### clawhub（历史参考：安装/卸载/更新/搜索）

    # 全局安装ClawHub CLI
    npm install -g clawhub

    # 搜索Skills
    clawhub search <关键词>
    clawhub search browser
    clawhub search --sort downloads

    # 安装Skills
    # 历史参考（非 2026.9 默认推荐）：clawhub install <slug>
    # clawhub install brave-search  # 历史参考

    # 安装到指定目录
    # 历史参考（非 2026.9 默认推荐）：clawhub install <slug> --dir /path/to/skills

    # 查看Skills详情（不安装）
    clawhub inspect <slug>

    # 列出已安装Skills
    clawhub list

    # 更新单个Skills
    clawhub update <slug>

    # 更新所有Skills
    clawhub update --all

    # 卸载Skills
    clawhub uninstall <slug>

    # 同步Skills
    clawhub sync

## A.8 插件管理（plugins）

    # 列出插件
    openclaw plugins list

    # 查看插件详情
    openclaw plugins info <id>

    # 安装插件
    openclaw plugins install <id>

    # 启用插件（需重启Gateway）
    openclaw plugins enable <id>

    # 禁用插件
    openclaw plugins disable <id>

    # 插件诊断
    openclaw plugins doctor

## A.9 日志与会话

    # 查看日志
    openclaw logs

    # 实时跟踪日志
    openclaw logs --follow

    # JSON格式日志
    openclaw logs --json

    # 纯文本日志
    openclaw logs --plain

    # 限制日志行数
    openclaw logs --limit 100

    # 查看会话信息
    openclaw sessions

## A.10 定时任务（cron）

    # 添加一次性定时任务
    openclaw cron add \
      --name "发送提醒" \
      --at "2026-03-15T18:00:00Z" \
      --session main \
      --system-event "提醒：提交费用报告"

    # 添加循环定时任务
    openclaw cron add \
      --name "早间状态" \
      --cron "0 7 * * *" \
      --tz "Asia/Shanghai" \
      --session isolated \
      --message "总结今天的收件箱和日历" \
      --deliver \
      --channel whatsapp

    # 列出定时任务
    openclaw cron list

    # 删除定时任务
    openclaw cron remove <job-id>

## A.11 消息发送（message）

    # 发送消息
    openclaw message send --channel <channel> --target <target> "消息内容"

    # 发送投票
    openclaw message poll --channel discord --target channel:123 \
      --poll-question "今晚吃什么？" --poll-option 火锅 --poll-option 烧烤

    # 其他消息操作
    openclaw message react
    openclaw message edit
    openclaw message delete
    openclaw message pin
    openclaw message search

    # 运行单次Agent对话
    openclaw agent --message "你好"

## A.12 安全与备份

    # 安全审计
    openclaw security audit

    # 深度安全审计
    openclaw security audit --deep

    # 创建备份
    openclaw backup create

    # 仅备份配置
    openclaw backup create --only-config

    # 校验备份
    openclaw backup verify <backup-id或路径>

    # 列出备份
    openclaw backup list

    # 恢复备份
    openclaw backup restore <文件路径>

    # 管理密钥
    openclaw secrets

## A.13 重置与卸载

    # 重置（配置+凭据+会话）
    openclaw reset

    # 卸载
    openclaw uninstall

    # 全自动卸载
    openclaw uninstall --all --yes --non-interactive

    # 模拟卸载（仅显示结果）
    openclaw uninstall --dry-run

    # 软件更新
    openclaw update

    # 查看更新状态
    openclaw update status

    # 更新到指定版本
    openclaw update --tag <版本号>

    # 更新到指定通道
    openclaw update --channel stable
    openclaw update --channel beta

## A.14 常用场景组合

### 场景1：初次安装后的配置

    # 1. 运行引导向导
    openclaw onboard

    # 2. 安装守护进程
    openclaw daemon install

    # 3. 启动
    openclaw daemon start

    # 4. 打开控制面板
    openclaw dashboard

### 场景2：切换模型

    # 1. 查看可用模型
    openclaw models list

    # 2. 切换模型
    openclaw models set anthropic/claude-sonnet-4-5

    # 3. 重启守护进程
    openclaw daemon restart

### 场景3：安装新Skills

    # 1. 搜索Skills
    clawhub search 截图

    # 2. 安装Skills
    # clawhub install peekaboo  # 历史参考

    # 3. 确认已安装
    openclaw skills list

    # 4. 重启守护进程
    openclaw daemon restart

### 场景4：故障排查

    # 1. 查看运行状态
    openclaw status

    # 2. 综合诊断
    openclaw doctor

    # 3. 查看日志
    openclaw logs --follow

    # 4. Gateway健康检查
    openclaw gateway health

    # 5. 安全审计
    openclaw security audit

## A.15 配置文件路径

    # 查看配置文件路径
    openclaw config file

    # 主配置文件（默认位置）
    ~/.openclaw/openclaw.json

    # Skills目录（工作区级）
    <workspace>/skills/

    # Skills目录（全局级）
    ~/.openclaw/skills/

    # 人设文件
    ~/clawd/SOUL.md
    ~/clawd/USER.md
    ~/clawd/AGENTS.md

    # 记忆目录
    ~/clawd/memory/

## 📚 相关资源

-   OpenClaw CLI完整参考：https://docs.openclaw.ai/cli
-   ClawHub CLI文档：https://docs.openclaw.ai/tools/clawhub
-   配置参考：https://docs.openclaw.ai/gateway/configuration

**提示**：本速查表基于v2026.3.7+版本验证。命令可能随版本更新而变化，遇到报错请先运行 `openclaw update` 更新到最新版本，或查阅官方文档。
