> 📖 **纸质书《OpenClaw超级个体实操手册》已上市！** 清华大学出版社出版，在开源教程基础上全面重写+逐条验证。🛒 [京东专属购买链接（¥42，原价¥59.8）](https://item.jd.com/14669463.html)

# 第13章节 高级自动化工作流（Cron / Tasks / Task Flow / Hooks / Standing Orders）

> 本章目标：建立一套符合 OpenClaw 当前官方设计的自动化观念，不再混淆 cron、heartbeat、tasks、Task Flow、hooks 和 standing orders 的职责边界。

---

## 版本基线

- **当前稳定版**：`v2026.9.3`（2026-09-08 发布）
- 自动化章节默认按 `v2026.9.3` 稳定版行为说明；升级后先 `openclaw doctor --fix`

---

## 先给小白的阅读说明

### 这章最重要的不是命令，而是顺序

很多人一上来就学 `Task Flow`，最后反而越学越乱。更适合小白的顺序是：

1. 先学 `cron`
2. 再学怎么看 `tasks`
3. 最后再上 `Task Flow`、`hooks`、`standing orders`

### 如果你只想先做一个能用的自动化

请先完成这一条最小路径：

- 用 `cron` 建一个每天早上跑的任务
- 用 `tasks list` 看它有没有进入后台账本
- 用 `tasks show` 看一次任务详情

如果这 3 步你都能完成，再往后看多步骤编排。

### 这章适合谁

- 已经能稳定使用 OpenClaw，但想把重复工作交给它
- 想把“提醒、报表、扫描、汇总”变成固定流程
- 想接外部触发，但还分不清 `hooks` 和 `webhooks`

---

## 13.1 先把 6 个概念分清楚

| 机制 | 适合什么 | 什么时候用 |
|------|----------|------------|
| `cron` | 精确时间触发 | 每天、每周、一次性提醒、定时报表 |
| `heartbeat` | 近似定期检查 | 需要持续“巡检主会话”而不是严格时刻 |
| `tasks` | 记录后台工作 | 看 detached work 跑了什么、卡在哪 |
| `Task Flow` | 多步骤可恢复编排 | A→B→C 这类 durable 流程 |
| `hooks` | 外部轻量触发 | 外部系统来一个事件，叫醒主会话或跑 isolated job |
| `standing orders` | 长期授权规则 | 把“你拥有哪些固定职责”写进 `AGENTS.md` |

官方文档的核心观点非常一致：

- `cron` 是调度器
- `tasks` 是后台工作账本
- `Task Flow` 是位于 tasks 之上的耐久编排层
- `standing orders` 是 agent 的长期运营授权，不是调度器本身

---

## 13.2 `cron`：时间驱动自动化的第一选择

### 13.2.1 一次性提醒

这是最适合新手先练手的例子，因为它结构最简单：到了某个时间点，发一次提醒，然后结束。

```bash
openclaw cron add   --name "Reminder"   --at "2026-05-01T16:00:00Z"   --session main   --system-event "Reminder: review the launch checklist"   --wake now   --delete-after-run
```

### 13.2.2 每日定时报表

```bash
openclaw cron add   --name "Morning brief"   --cron "0 7 * * *"   --tz "Asia/Shanghai"   --session isolated   --message "Summarize overnight updates, key tasks, and calendar priorities."   --announce
```

### 13.2.3 运维常用命令

#### 看到什么算 `cron` 已经跑通

你至少要会检查这 3 件事：

- `openclaw cron list` 里能看到你刚创建的任务
- `openclaw cron runs --id <job-id>` 能看到实际运行记录
- 如果任务是长时间运行的，`tasks list` 里也能看到对应后台任务

```bash
openclaw cron list
openclaw cron runs --id <job-id>
openclaw cron run <job-id>
openclaw cron edit <job-id> --message "Updated prompt"
openclaw cron remove <job-id>
```

### 13.2.4 现在应该怎么选 session 模式

- `main`：适合提醒、系统事件、继续主会话上下文
- `isolated`：适合日报、扫描、报告、批处理
- `current`：适合绑定当前会话的重复任务
- `session:custom-id`：适合每天都延续同一份上下文的固定流程

---

## 13.3 `tasks`：先学会看账本，再谈复杂编排

很多人把自动化做复杂了，最后却不知道“到底什么在跑、哪里失败了”。这就是 `tasks` 的意义。

### 13.3.1 你最该会的命令

```bash
openclaw tasks list
openclaw tasks show <task-id>
openclaw tasks cancel <task-id>
openclaw tasks audit
```

### 13.3.2 `tasks audit` 最有价值的地方

它会直接告诉你是否存在：

- `stale_queued`
- `stale_running`
- `lost`
- `delivery_failed`
- `missing_cleanup`
- `inconsistent_timestamps`

如果你已经在用：

- 视频生成
- 音乐生成
- isolated cron
- subagent / ACP 子任务

那 `tasks audit` 应该成为你的固定巡检动作。

---

## 13.4 `Task Flow`：多步骤流程的标准编排层

Task Flow 适合：

- 多步骤串行或分支流程
- 要求持久化状态
- 需要在 gateway 重启后继续
- 需要统一查看流程进度

### 13.4.1 两种模式

**Managed mode**：Task Flow 自己驱动每一步任务

适合：

- 周报流水线
- 内容生产流水线
- 审批后自动交付

**Mirrored mode**：Task Flow 只观察外部任务并同步状态

适合：

- 已经有 cron / CLI / 外部系统在生成任务
- 你只是想把多个任务收敛成一个流程视图

### 13.4.2 CLI 命令

```bash
openclaw tasks flow list
openclaw tasks flow show <lookup>
openclaw tasks flow cancel <lookup>
```

### 13.4.3 什么时候不要上 Task Flow

以下情况先别急着上：

- 只是单次后台任务
- 只是每天一个固定提醒
- 你还不会看 `tasks list` / `tasks audit`
- 工作流步骤还没稳定，今天改一版、明天改一版

这时先用 cron 或 isolated job 更稳。

---

## 13.5 `hooks` 与 `webhooks` plugin：把外部系统接进来

### 13.5.1 轻量触发：`hooks`

官方 `hooks` 适合外部事件进来后：

- 唤醒主会话
- 触发一次 isolated agent run

配置：

```json
{
  "hooks": {
    "enabled": true,
    "token": "replace-with-dedicated-hook-token",
    "path": "/hooks"
  }
}
```

调用 `wake`：

```bash
curl -X POST http://127.0.0.1:18789/hooks/wake   -H 'Authorization: Bearer SECRET'   -H 'Content-Type: application/json'   -d '{"text":"New invoice received","mode":"now"}'
```

调用 `agent`：

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent   -H 'Authorization: Bearer SECRET'   -H 'Content-Type: application/json'   -d '{"message":"Summarize the new invoice and extract payable date","name":"Finance","model":"openai/gpt-5.4-mini"}'
```

### 13.5.2 多步骤编排：`webhooks` plugin

如果你是 Zapier、n8n、CI、表单系统来驱动复杂流程，直接用 `webhooks` plugin 绑定 Task Flow。官方配置形态：

```json
{
  "plugins": {
    "entries": {
      "webhooks": {
        "enabled": true,
        "config": {
          "routes": {
            "ops": {
              "path": "/plugins/webhooks/ops",
              "sessionKey": "agent:main:main",
              "secret": {
                "source": "env",
                "provider": "default",
                "id": "OPENCLAW_WEBHOOK_SECRET"
              },
              "controllerId": "webhooks/ops",
              "description": "Ops TaskFlow bridge"
            }
          }
        }
      }
    }
  }
}
```

**最常用两个动作**：

- `create_flow`
- `run_task`

这比旧教程里一堆 shell glue code 更稳定，也更方便排障。

---

## 13.6 `standing orders`：自动化不是“计划”，而是“长期授权”

OpenClaw 官方现在把 standing orders 讲得很清楚：它们不是某个定时器，而是**写在 agent workspace 里的长期授权规则**。推荐直接放在 `AGENTS.md` 里，因为这个文件会自动注入到每次会话。

### 13.6.1 一个合格的 standing order 长什么样

```md
## Program: Weekly Status Report

**Authority:** Compile data, generate report, deliver to stakeholders
**Trigger:** Every Friday at 4 PM (enforced via cron job)
**Approval gate:** None for standard reports. Flag anomalies for human review.
**Escalation:** If data source is unavailable or metrics look unusual

### Execution Steps

1. Pull metrics from configured sources
2. Compare to prior week and targets
3. Generate report in Reports/weekly/YYYY-MM-DD.md
4. Deliver summary via configured channel
5. Log completion to Agent/Logs/

### What NOT to Do

- Do not send reports to external parties
- Do not modify source data
- Do not skip delivery if metrics look bad
```

### 13.6.2 为什么它重要

没有 standing orders：

- 你每次都要重复下达同样的管理指令
- agent 只能“被动等你叫”
- routine work 很容易中断

有 standing orders：

- 责任边界更清楚
- 人工审批点更清楚
- agent 的长期行为更可控
- cron 只负责“何时跑”，AGENTS.md 负责“跑什么、什么时候停”

---

## 13.7 推荐的自动化组合，不推荐的自动化组合

### 推荐组合

**组合 1：定时报表**

- `cron`
- `tasks list / audit`
- `AGENTS.md` standing order

**组合 2：外部系统驱动的多步流程**

- `hooks` 或 `webhooks` plugin
- `Task Flow`
- `tasks flow show`

**组合 3：长期研究 / 知识运营**

- `cron`
- `infer web search / fetch`
- `Memory Wiki`

### 不推荐组合

- 只用 shell while true + sleep 模拟调度
- 用一个超大 prompt 代替 standing orders
- 还不会看 `tasks audit` 就上很多 subagent 和 webhook
- 把外部系统 token 直接写进仓库

---

## 13.8 本章实践建议

如果你要从 0 到 1 搭一套稳定自动化，顺序应该是：

1. 先用 `cron` 跑通一个固定任务
2. 再学会看 `tasks list` 和 `tasks audit`
3. 多步骤时再上 `Task Flow`
4. 外部系统接入时再开 `hooks` / `webhooks` plugin
5. 最后把长期职责沉淀进 `AGENTS.md`

---

## 13.9 官方参考

- GitHub Releases：https://github.com/openclaw/openclaw/releases
- Automation Overview：https://docs.openclaw.ai/automation/cron-vs-heartbeat
- Scheduled Tasks：https://docs.openclaw.ai/automation/cron-jobs
- Background Tasks：https://docs.openclaw.ai/automation/tasks
- Task Flow：https://docs.openclaw.ai/automation/taskflow
- Standing Orders：https://docs.openclaw.ai/automation/standing-orders
- Webhooks Plugin：https://docs.openclaw.ai/plugins/webhooks
