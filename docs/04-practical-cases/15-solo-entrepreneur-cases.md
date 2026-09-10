> 📖 **纸质书《OpenClaw超级个体实操手册》已上市！** 清华大学出版社出版，在开源教程基础上全面重写+逐条验证。🛒 [京东专属购买链接（¥42，原价¥59.8）](https://item.jd.com/14669463.html)

# 第15章节 一人公司实战（选题、交付、分发、复盘）

> 本章目标：把“一个人做品牌、做产品、做分发”的工作流，改写成符合 OpenClaw 当前官方能力的版本：用 standing orders、cron、Task Flow、媒体能力和 Memory Wiki 形成闭环。

---

## 版本基线

- **当前稳定版**：`v2026.9.3`（2026-09-08 发布）
- 本章一律按 `v2026.9.3` 稳定版主线展开，避免把历史 beta / 已移除插件写成默认商业流程

---

## 先给小白的阅读说明

### 这章不是“全自动赚钱教程”

这一章更准确的定位是：**教你怎么把一个人的日常商业工作拆成可复用流程**。它不会让你完全不审稿、不确认、不复核就自动对外发布。

### 谁适合先读这一章

- 你已经把第 `13` 章自动化和第 `14` 章媒体能力跑通过
- 你自己一个人做内容、产品、咨询、课程或服务交付
- 你想把“研究、起草、产出、分发、复盘”做成半自动流水线

### 小白怎么读最省力

- **只想做内容系统**：先看 `15.1`
- **只想做线索跟进**：先看 `15.2`
- **想理解一人公司的整体操作系统**：再看 `15.3` 以后

### 最重要的原则

外部发布、对外报价、法律风险、品牌风险，默认都要保留人工审批。OpenClaw 擅长的是**帮你提速**，不是替你承担责任。

---

## 15.1 案例一：个人品牌内容流水线

### 15.1.1 目标

一个人完成下面这条链路：

1. 每天自动发现值得写的题目
2. 快速做素材研究
3. 产出文章 / 推文 / 视频脚本草稿
4. 自动生成配图、试音或短视频素材
5. 人工审核后再分发

### 15.1.2 先写 Standing Orders，而不是先堆工具

把 agent 的长期职责写进 `AGENTS.md`：

```md
## Program: Daily Content Desk

**Authority:** Research AI/productivity topics, draft content, prepare assets
**Trigger:** Daily 09:00 via cron, plus manual on-demand requests
**Approval gate:** External publishing always requires human approval
**Escalation:** If sources conflict, legal risk appears, or confidence is low

### Execution Steps

1. Search for the last 24h worth of relevant updates
2. Rank 3 content angles
3. Draft one article outline and one short-form script
4. Prepare image/video/audio asset suggestions
5. Save everything to the workspace and wait for approval

### What NOT to Do

- Do not auto-publish to external platforms
- Do not fabricate facts or quotes
- Do not reuse old claims when sources conflict
```

### 15.1.3 定时触发

这个例子最适合先跑，因为它能让你每天固定收到一批候选选题，你很容易判断系统有没有真正帮你省时间。

```bash
openclaw cron add   --name "Daily content desk"   --cron "0 9 * * *"   --tz "Asia/Shanghai"   --session isolated   --message "Research AI/productivity topics from the last 24 hours, rank 3 angles, draft one article outline and one short-form script, then summarize what needs approval."   --announce
```

### 15.1.4 素材生成

研究阶段优先走官方搜索与抓取：

```bash
openclaw infer web search --query "OpenClaw release notes April 2026" --json
openclaw infer web fetch --url https://github.com/openclaw/openclaw/releases --json
```

素材阶段优先走官方媒体能力：

```bash
openclaw infer image generate --prompt "一张白板风格配图：AI 内容生产流水线" --json
openclaw infer tts convert --text "今天的选题已经准备完成" --output ./topic-brief.mp3 --json
```

如果是短视频素材，则让 agent 调用 `video_generate`，并在 `tasks list` 里看进度。

#### 看到什么算这条内容流水线已经有价值

- 你每天都能稳定拿到候选题目和草稿
- 你不需要手动开十几个网页再拼素材
- 你保留最终审核权，而不是让系统自动外发

### 15.1.5 为什么这一套比旧教程稳

旧版一人公司案例里最大的问题，是默认把一大堆平台化发布工具、历史 Skill 名称和经验流写死在正文里。现在更稳的做法是：

- OpenClaw 负责研究、起草、素材准备、状态追踪
- 外部分发放在人工审批后进行
- 如果确实要自动分发，走 `hooks` / `webhooks` plugin 或自家中间层，不把它写死成某个旧平台依赖

---

## 15.2 案例二：线索收集与跟进自动化

### 15.2.1 场景

你有：

- 官网表单
- 邮件咨询
- Telegram / Slack / 飞书私信
- 产品试用申请

你想做到：

1. 新线索进来立刻被识别
2. 自动补齐基础画像
3. 按优先级分级
4. 生成跟进建议
5. 需要时升级为人工处理

### 15.2.2 推荐架构

```text
表单 / 邮件 / IM 事件
  -> hooks 或 webhooks plugin
  -> create_flow
  -> run_task（资格判断 / 摘要 / 回复建议）
  -> 写入 Memory Wiki / CRM
  -> 人工审批是否发送正式回复
```

### 15.2.3 外部触发最小实现

轻量场景直接用 `hooks/agent`：

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent   -H 'Authorization: Bearer SECRET'   -H 'Content-Type: application/json'   -d '{"message":"Classify this inbound lead, summarize intent, infer urgency, and draft a reply outline.","name":"Lead intake","model":"openai/gpt-5.4-mini"}'
```

复杂场景则用 `webhooks` plugin 创建 Task Flow：

- `create_flow`：建立该线索的流程对象
- `run_task`：拆出资格判断、资料补齐、回复建议等子任务

### 15.2.4 为什么要把线索写进 Memory Wiki

因为一人公司最怕的不是“回复慢”，而是：

- 前后说法不一致
- 客户背景记不住
- 承诺过的事没人记得
- 线索状态散落在多个地方

如果你把关键信息沉淀到 wiki，你会得到：

- 谁是这个人 / 公司
- TA 来自哪里
- TA 关注什么
- 之前谈到哪一步
- 还有哪些未决问题

这对小团队极其关键。

---

## 15.3 一人公司的 4 层操作系统

我更推荐把 OpenClaw 在一人公司里的角色拆成 4 层：

### 第一层：日常执行层

负责：

- 搜索资料
- 总结信息
- 生成图片 / 试音 / 视频素材
- 处理日常消息和待办

核心能力：

- `infer`
- `image_generate`
- `video_generate`
- `music_generate`
- `tts`

### 第二层：调度层

负责：

- 定时执行
- 记录任务状态
- 处理长任务
- 跟踪多步骤流程

核心能力：

- `cron`
- `tasks`
- `Task Flow`

### 第三层：长期授权层

负责：

- 规定 agent 长期负责什么
- 规定哪些事必须审批
- 规定何时升级给你处理

核心载体：

- `AGENTS.md`
- standing orders

### 第四层：知识层

负责：

- 沉淀客户、项目、产品、内容资产
- 记录 claim / evidence / contradiction
- 把经验变成可检索知识

核心能力：

- `memory-core`
- `active-memory`
- `memory-wiki`

---

## 15.4 一人公司应该怎么设审批边界

这是本章最重要的一节。

**可以自动做**：

- 研究与摘要
- 选题建议
- 资料抓取
- 素材初稿生成
- 线索分级建议
- 周报和日报草稿

**必须人工确认**：

- 对外正式发布
- 对客户作出承诺
- 发价格、合同、结算信息
- 处理敏感舆情和公关风险
- 修改生产数据或关键配置

OpenClaw 很适合做“前 80% 的准备工作”，但最后那一跳商业责任，必须保留给人。

---

## 15.5 最小可落地栈（推荐）

如果你就是一个人，不要一开始就搞“全家桶”。下面这套最实用：

### 必备

- `openclaw onboard`
- 主模型 + 回退链
- 一个 `cron` 日报 / 周报任务
- 一个 `AGENTS.md` standing order
- 一个 `hooks` 或 `webhooks` 接口

### 第二阶段再加

- `Active Memory`
- `Memory Wiki`
- 图片 / 视频 / 音乐默认模型
- `Task Flow`

### 暂时别急着加

- 一堆历史 Skill 生态兼容层
- 平台强耦合脚本
- 没有审批边界的自动外发
- 没有任务观测能力的多 agent 编排

---

## 15.6 每周复盘清单

建议每周固定看一次：

```bash
openclaw status
openclaw cron list
openclaw tasks audit
openclaw tasks flow list
openclaw memory status --deep
openclaw wiki status
openclaw wiki lint
```

重点看：

1. 任务有没有堆积
2. 哪些流程总是失败
3. 哪些承诺没有进入知识层
4. 哪些 standing orders 该收缩或扩权
5. 哪些自动化还没有审批边界

---

## 15.7 本章结论

一人公司不是“让 AI 代替你”，而是：

- 让 OpenClaw 接管研究、整理、调度、素材准备和状态追踪
- 让你只把精力放在判断、选择、品牌、产品和最终责任上

换句话说：

**把重复劳动交给系统，把最终判断留给你自己。**

---

## 15.8 官方参考

- GitHub Releases：https://github.com/openclaw/openclaw/releases
- Automation Overview：https://docs.openclaw.ai/automation/cron-vs-heartbeat
- Scheduled Tasks：https://docs.openclaw.ai/automation/cron-jobs
- Task Flow：https://docs.openclaw.ai/automation/taskflow
- Standing Orders：https://docs.openclaw.ai/automation/standing-orders
- Webhooks Plugin：https://docs.openclaw.ai/plugins/webhooks
- Memory Wiki：https://docs.openclaw.ai/plugins/memory-wiki
- Inference CLI：https://docs.openclaw.ai/cli/infer
