> 📖 **纸质书《OpenClaw超级个体实操手册》已上市！** 清华大学出版社出版，在开源教程基础上全面重写+逐条验证。🛒 [京东专属购买链接（¥42，原价¥59.8）](https://item.jd.com/14669463.html)

# 第5章节 知识库与第二大脑（Active Memory / Memory Wiki / 研究归档）

> 本章目标：用 OpenClaw 当前官方主线，把“知识库”这件事讲清楚：什么交给 Active Memory，什么交给 Memory Wiki，什么只是临时资料，不要再把旧的第三方同步 Skill 当默认方案。

---

## 版本基线

- **当前稳定版**：`v2026.9.3`（2026-09-08 发布）
- 本章默认按 `v2026.9.3` 稳定版写

---

## 先给小白的阅读说明

### 这一章到底解决什么问题

很多人一提“知识库”，脑子里想到的是：

- 把网页存进 Notion
- 把笔记同步到 Obsidian
- 做一堆目录和标签

这些不是没用，但它们已经不是 OpenClaw 当前的主线。现在更推荐你把知识系统拆成 3 层：

1. **Active Memory**：回复前先回忆相关上下文
2. **Memory Wiki**：把长期知识整理成结构化、可检索的知识层
3. **临时资料处理**：网页抓取、会议纪要、研究摘录，先整理再决定要不要进入长期知识层

### 如果你只想先跑通，先看这几节

- **想让 OpenClaw 更“记得住你”**：看 `5.2`
- **想开始搭第二大脑**：看 `5.3`
- **想做文章、项目、论文资料沉淀**：看 `5.4`
- **你已经在用 Obsidian**：看 `5.5`

### 小白最容易犯的 3 个错

- 一上来就先折腾同步，而不是先把记忆主线配通
- 把“临时资料”直接塞进长期知识库，最后越存越乱
- 以为所有知识都必须结构化，结果迟迟不开始

### 本章一句话结论

先让 OpenClaw 能**记得住**，再让它能**查得到**，最后才考虑它要不要**同步到别的笔记工具**。

---

## 5.1 2026.4 之后，知识库的正确主线是什么？

旧版教程里，“知识库”主要靠网页归档、Notion 同步、Obsidian 同步来实现。但在当前官方主线里，更重要的是下面这套组合：

| 层级 | 负责什么 | 适合什么场景 |
|------|----------|--------------|
| `Active Memory` | 回复前先回忆相关偏好、事实、上下文 | 聊天、长期协作、偏好记忆 |
| `Memory Wiki` | 把长期知识编译成结构化知识层 | 项目知识、研究资料、长期主题沉淀 |
| `openclaw infer web fetch/search` | 收集外部资料 | 网页、文档、新闻、调研资料 |
| `wiki ingest / compile / search / get` | 资料入库、整理、检索 | 第二大脑、团队知识、项目档案 |

你可以这样理解：

- **Active Memory** 解决的是“回复时能不能想起来”
- **Memory Wiki** 解决的是“长期资料能不能整理好、查得准”
- **网页抓取 / 转写 / 摘要** 解决的是“怎么把外部信息先拿进来”

---

## 5.2 Active Memory：先让 OpenClaw 记得住你

官方文档对 Active Memory 的定义很明确：它是一个**在主回复之前运行的阻塞式记忆子代理**。它会在合适的会话里先用 `memory_search`、`memory_get` 找相关记忆，再把精简结果交给主回复。

### 5.2.1 什么时候该开

适合直接开启的情况：

- 你经常和同一个 agent 长期协作
- 你希望它记住你的习惯、偏好、工作背景
- 你希望它少问重复问题

先不要急着开的情况：

- 你只是偶尔临时问几句
- 你现在主要在排查模型/认证问题
- 你对延迟非常敏感，想先把基础链路跑通

### 5.2.2 官方推荐起步配置

根据官方文档，最安全的起步方式是：启用插件、只绑定一个对话 agent、只先在 direct chat 里使用。

把下面这段写到 `openclaw.json`：

```json5
{
  plugins: {
    entries: {
      "active-memory": {
        enabled: true,
        config: {
          agents: ["main"],
          allowedChatTypes: ["direct"],
          modelFallback: "google/gemini-3-flash",
          queryMode: "recent",
          promptStyle: "balanced",
          timeoutMs: 15000,
          maxSummaryChars: 220,
          persistTranscripts: false,
          logging: true
        }
      }
    }
  }
}
```

改完以后，重启你当前使用的 Gateway 进程或服务。

### 5.2.3 看到什么算 Active Memory 已经跑通

你可以先用最简单的方式判断：

- 同一个对话里，OpenClaw 开始更稳定地记住你的偏好和背景
- 开启 `/verbose` 或 `/trace` 时，能看到 Active Memory 的状态信息
- 没有明显的模型报错、超时报错或空转

### 5.2.4 小白怎么理解这些配置项

- `agents: ["main"]`：只让 `main` 这个 agent 用主动记忆
- `allowedChatTypes: ["direct"]`：先只在私聊/直接对话里使用，避免一上来全局打开
- `queryMode: "recent"`：优先看最近上下文，速度和效果更平衡
- `promptStyle: "balanced"`：默认是稳妥模式，适合大多数用户
- `timeoutMs: 15000`：超过 15 秒就放弃，避免拖慢回复太多

如果你是第一次用，不建议一开始就改得太激进。默认先跑稳，再谈优化。

---

## 5.3 Memory Wiki：把长期知识整理成“能查、能证据回溯”的知识层

官方对 `memory-wiki` 的定位不是普通笔记插件，而是一个**把 durable memory 编译成知识 vault 的 bundled plugin**。

它和 Active Memory 的关系是：

- Active Memory 负责“回复时先想起来”
- Memory Wiki 负责“把长期知识整理成稳定页面、claims、evidence、dashboards”

### 5.3.1 为什么它比“随手记笔记”更适合长期知识

Memory Wiki 的优势不只是“能存”，而是：

- 有专门的 wiki vault
- 支持结构化 `claim / evidence`
- 能编译出稳定页面和 digest
- 能用 `wiki_search` / `wiki_get` 精准检索
- 可以和活动记忆层做 shared search

这更像一个“知识层”，而不是一个“笔记堆”。

### 5.3.2 官方推荐起步配置

官方文档建议把配置放到 `plugins.entries.memory-wiki.config`。下面这份是比较适合新手的起步版本：

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "isolated",
          vault: {
            path: "~/.openclaw/wiki/main",
            renderMode: "obsidian"
          },
          obsidian: {
            enabled: true,
            useOfficialCli: true,
            vaultName: "OpenClaw Wiki",
            openAfterWrites: false
          },
          bridge: {
            enabled: false,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true
          },
          ingest: {
            autoCompile: true,
            maxConcurrentJobs: 1,
            allowUrlIngest: true
          },
          search: {
            backend: "shared",
            corpus: "wiki"
          },
          context: {
            includeCompiledDigestPrompt: false
          },
          render: {
            preserveHumanBlocks: true,
            createBacklinks: true,
            createDashboards: true
          }
        }
      }
    }
  }
}
```

### 5.3.3 小白先记住 4 个关键开关就够了

- `vaultMode: "isolated"`：最适合刚开始，先把 wiki 当独立知识层
- `renderMode: "obsidian"`：如果你已经在用 Obsidian，会比较顺手
- `search.backend: "shared"`：需要时可以和共享记忆检索打通
- `createDashboards: true`：让 wiki 自动生成更容易浏览的汇总页

### 5.3.4 什么时候用 `bridge` 模式

只有在你已经明确知道：

- 自己的 active memory backend 已经有公开 bridge artifacts
- 想把记忆层里的长期资料编译进 wiki

这时再考虑 `bridge`。如果你现在只是第一次上手，先用 `isolated`。

---

## 5.4 最适合小白先跑通的知识库流程

这一节不讲大而全的系统，只讲最短闭环。

### 5.4.1 流程 A：把一份本地笔记收进 Wiki

先初始化：

```bash
openclaw wiki init
openclaw wiki status
```

然后准备一份最简单的 Markdown 笔记，例如 `./notes/customer-onboarding.md`：

```md
# Customer Onboarding

## 现状
- 新用户第一次接触产品时，最容易卡在权限配置
- 现有帮助文档太散，入口不统一

## 我的判断
- 应该做一份统一 onboarding checklist
- 首屏应该先给 3 个最常见动作
```

把它导入 Wiki：

```bash
openclaw wiki ingest ./notes/customer-onboarding.md
openclaw wiki compile
openclaw wiki lint
```

搜索并读取：

```bash
openclaw wiki search "onboarding"
openclaw wiki get <lookup>
```

### 5.4.2 看到什么算 Wiki 已经跑通

至少满足下面 4 条：

- `openclaw wiki status` 能看到 vault 状态正常
- `openclaw wiki ingest` 能成功吃进你的文件
- `openclaw wiki compile` 没有报结构错误
- `openclaw wiki search` 能搜到你刚导入的主题

### 5.4.3 流程 B：把网页资料变成长期知识

更推荐的顺序不是“直接同步到第三方笔记工具”，而是：

1. 先抓网页资料
2. 让 OpenClaw 摘要成你自己的笔记
3. 再决定是否进入 Wiki

先抓取网页：

```bash
openclaw infer web search --query "OpenClaw Active Memory use cases" --json
openclaw infer web fetch --url https://docs.openclaw.ai/concepts/active-memory --json
```

然后你可以让 OpenClaw 帮你整理成一份 Markdown 摘要，再保存到 `./notes/`，最后继续：

```bash
openclaw wiki ingest ./notes/active-memory-summary.md
openclaw wiki compile
```

### 5.4.4 流程 C：项目知识沉淀

项目知识最适合进 Wiki 的内容是：

- 架构决策
- 常见故障排查
- 客户/用户共性问题
- 术语解释
- 上线复盘

不适合一股脑塞进去的内容是：

- 一次性的临时聊天记录
- 未经整理的超长日志
- 还没确认真假的外部信息

最好的习惯是：**先整理，再入库**。

---

## 5.5 如果你已经在用 Obsidian，应该怎么理解它和 Wiki 的关系

这一章不再把 Obsidian 当成默认主线，而把它放回它更适合的位置：**展示层 / 编辑层**。

### 更推荐的关系是

- OpenClaw 负责记忆、编译、检索
- Wiki 负责知识结构
- Obsidian 负责你的人类阅读和手工编辑体验

如果你已经在用 Obsidian，可以重点关注这些命令：

```bash
openclaw wiki obsidian status
openclaw wiki obsidian search "onboarding"
openclaw wiki obsidian open syntheses/alpha-summary.md
openclaw wiki obsidian daily
```

### 不再推荐当默认主线的旧写法

下面这些旧路线，现在不再建议当成入门默认方案：

- `clawhub install notion-sync`
- `clawhub install obsidian-sync`
- 先装同步 Skill，再把第三方笔记工具当记忆主系统

它们不是完全不能做，而是更适合作为**你已经有现成工作流时的补充集成**，而不是当前官方知识主线。

---

## 5.6 本章最容易踩的坑

### 坑 1：把所有资料都当长期知识

不是所有东西都值得进 Wiki。一个很实用的判断标准是：

- 这条信息以后还会反复被用到吗？
- 它是否已经整理到足够清楚？
- 它是否值得被检索、引用和复盘？

### 坑 2：一上来就开太多模式

新手建议顺序：

1. 先开 Active Memory
2. 再初始化 Wiki
3. 先用 `isolated` 模式
4. 跑通后再考虑 `bridge`

### 坑 3：把同步当成目标，把知识本身忘了

Notion、Obsidian、备忘录都只是载体。真正该优先解决的，是：

- OpenClaw 能不能想起来
- 资料能不能查得到
- 结论有没有证据来源

---

## 5.7 给小白的最短落地顺序

如果你今天只想花 30 分钟先把“知识库”跑起来，按这个顺序做：

1. 打开 Active Memory
2. 初始化 Memory Wiki
3. 写一份自己的本地 Markdown 笔记
4. `wiki ingest` + `wiki compile`
5. `wiki search` 看能不能搜到
6. 再决定要不要接 Obsidian

---

## 5.8 官方参考

- Active Memory：https://docs.openclaw.ai/concepts/active-memory
- Memory Wiki：https://docs.openclaw.ai/plugins/memory-wiki
- Wiki CLI：https://docs.openclaw.ai/cli/wiki
- Inference CLI：https://docs.openclaw.ai/cli/infer
