> 📖 **纸质书《OpenClaw超级个体实操手册》已上市！** 清华大学出版社出版，在开源教程基础上全面重写+逐条验证。🛒 [京东专属购买链接（¥42，原价¥59.8）](https://item.jd.com/14669463.html)

# 第14章节 创意应用实战（图片、视频、音乐、TTS、ComfyUI）

> 本章目标：把旧教程里零散的“画图 / 视频 / 语音 / 第三方 Skill”写法，收束成 OpenClaw 当前官方支持的媒体工作流。

---

## 版本基线

- **当前稳定版**：`v2026.9.3`（2026-09-08 发布）
- 本章默认按 `v2026.9.3` 稳定版写；所有创意能力优先参考官方媒体能力矩阵

---

## 先给小白的阅读说明

### 如果你只是想先做出一个结果，别整章都看

- **想先出一张图**：直接看 `14.2`
- **想先生成语音**：直接看 `14.4.2`
- **想先试视频**：直接看 `14.3`，但要知道视频是异步任务
- **想做本地可控工作流**：最后再看 `14.5` 的 ComfyUI

### 开始前要先知道的 2 件事

1. 图片和 TTS 通常是**同步返回**，比较适合新手先试
2. 视频和音乐通常是**后台异步任务**，所以不要以为“命令没立刻吐文件就是失败”

### 小白最容易误会的地方

- 把“图片理解模型”当成“图片生成模型”
- 不知道视频生成要去 `tasks` 里看状态
- 看见 provider 很多，就误以为每个都必须配置

---

## 14.1 先记住这张官方媒体能力图谱

OpenClaw 当前的媒体能力不是零散插件，而是一套共享能力层：

| 能力 | 工具 / 命令 | 常见 provider | 说明 |
|------|-------------|---------------|------|
| 图片生成 | `image_generate` / `openclaw infer image generate` | ComfyUI、fal、Google、MiniMax、OpenAI、Vydra | 文生图、参考图编辑 |
| 视频生成 | `video_generate` / `openclaw infer video generate` | Alibaba、BytePlus、ComfyUI、fal、Google、MiniMax、OpenAI、Qwen、Runway、Together、Vydra、xAI | 文生视频、图生视频、视频转视频 |
| 音乐生成 | `music_generate` | ComfyUI、Google、MiniMax | 生成音乐 / 音轨 |
| TTS | `tts` / `openclaw infer tts convert` | ElevenLabs、Microsoft、MiniMax、OpenAI | 把文本转成语音 |
| 媒体理解 | `image describe` / `audio transcribe` / `video describe` | 各类多模态 provider | 读图、读音频、读视频 |

最重要的两点：

1. **图片和 TTS 更偏同步**
2. **视频和音乐是异步后台任务**，会进入 task ledger，完成后再唤醒 agent 把结果发回原会话

---

## 14.2 图片工作流：现在应该怎么做

### 14.2.1 命令行直出图

如果你是第一次试媒体能力，强烈建议从这里开始，因为它反馈最快，也最容易判断到底是提示词问题，还是 provider 没配好。

```bash
openclaw infer image generate   --prompt "一张手写白板风格的 OpenClaw 自动化架构图"   --json
```

适合：

- 教程配图
- 封面图
- 白板图
- 社交媒体海报
- 结构示意图

#### 看到什么算图片能力已经跑通

- 命令能返回 JSON 结果或文件输出信息
- 生成效果不满意时，你知道先改提示词，而不是先怀疑整套系统坏了
- 你已经能分清“命令行直出图”和“会话里让 agent 自动调工具”这两种方式

### 14.2.2 对话里直接让 agent 生成

```text
帮我生成一张白板手写风格的配图，主题是“从 cron 到 Task Flow 的自动化升级路径”。
```

如果 `image_generate` 已可用，agent 会自动调用对应工具。相比旧教程里的历史 Skill 名称，这才是当前默认主线。

### 14.2.3 什么时候要单独配 `imageGenerationModel`

当你满足下面任一情况时，建议手动指定：

- 团队里统一使用某个 provider
- 你想严格控制成本
- 你不希望 OpenClaw 自动推断 provider

```json
{
  "agents": {
    "defaults": {
      "imageGenerationModel": {
        "primary": "openai/gpt-image-1"
      }
    }
  }
}
```

---

## 14.3 视频工作流：理解“异步返回”很关键

### 14.3.1 最短可用示例

视频生成比图片慢很多，所以你第一次测视频时，目标不是“直接出大片”，而是先确认任务能成功入账并最终完成。

```bash
openclaw infer video generate   --prompt "一段 5 秒的电影感镜头：桌面上的 OpenClaw 仪表盘正在更新任务状态"   --json
```

### 14.3.2 当前视频工作流的正确心智模型

视频生成不是“一条命令马上拿到 mp4”。更准确的过程是：

1. OpenClaw 把请求发给 provider
2. provider 返回任务 id
3. 任务进入 background task ledger
4. 完成后 OpenClaw 唤醒原会话，把视频回贴回来

所以你需要学会看：

```bash
openclaw tasks list
openclaw tasks show <task-id>
openclaw tasks audit
```

### 14.3.3 推荐的视频默认模型写法

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "google/veo-3.1-fast-generate-preview"
```

如果你希望带回退链：

```json
{
  "agents": {
    "defaults": {
      "videoGenerationModel": {
        "primary": "google/veo-3.1-fast-generate-preview",
        "fallbacks": [
          "qwen/wan2.6-r2v-flash"
        ]
      }
    }
  }
}
```

### 14.3.4 适合 OpenClaw 做的视频场景

- 产品演示短视频
- 课程片头 / 社交媒体短片
- 图生视频、视频转视频实验
- 自动化营销素材流水线中的“中短视频生成”步骤

---

## 14.4 音乐生成与 TTS：创意产出的最后两块拼图

### 14.4.1 音乐生成

当前推荐优先通过 agent 工具 `music_generate` 使用。根据官方文档，如果你看不到这个工具，优先检查：

- provider API key 是否已配置
- `agents.defaults.musicGenerationModel` 是否已配置

推荐配置示例：

```json
{
  "agents": {
    "defaults": {
      "musicGenerationModel": {
        "primary": "google/lyria-3-clip-preview"
      }
    }
  }
}
```

典型 prompt：

```text
生成一段 20 秒的轻电子 synthpop 背景音乐，节奏明快，不要人声，适合做 AI 产品介绍短视频配乐。
```

### 14.4.2 TTS：脚本里用 `infer`，会话里用 `tts`

```bash
openclaw infer tts convert   --text "欢迎来到今天的 OpenClaw 自动化教程。"   --output ./intro.mp3   --json
```

适合：

- 视频旁白草稿
- 日报语音播报
- 课程试音
- 产品播报提醒

---

## 14.5 ComfyUI：本地 / 可控工作流的官方连接点

旧教程里很多“本地媒体工作流”写法，是靠零散脚本或第三方桥接完成的。现在更推荐：

- OpenClaw 继续做**调度与对话入口**
- `ComfyUI` 负责**本地媒体图形工作流**
- 两者通过官方 provider/plugin 接起来

这套组合非常适合：

- 固定模板的海报 / 封面量产
- 统一风格的视频片头 / 片尾
- 音乐或图像的工作流固化
- 本地私有资产处理

如果你的目标是“生产级可重复工作流”，优先考虑 ComfyUI，而不是继续在旧 Skill 名称上做兼容。

---

## 14.6 四类值得直接照搬的创意工作流

### 工作流 1：教程配图流水线

- `infer web fetch` 抓资料
- 主模型整理要点
- `infer image generate` 生成白板图
- 保存到素材目录

### 工作流 2：短视频脚本 + 旁白草稿

- 主模型出脚本
- `infer tts convert` 生成试音
- `video_generate` 出镜头草案
- 人工二次剪辑

### 工作流 3：品牌一致的封面批量生成

- 统一 prompt 模板
- 统一图片尺寸和构图要求
- 指定 `imageGenerationModel`
- 批量脚本调用 `infer image generate`

### 工作流 4：音乐 / 视频异步生产

- 用 agent 发起 `music_generate` / `video_generate`
- 用 `tasks list` 看进度
- 完成后自动回传渠道

---

## 14.7 本章最容易踩的坑

### 坑 1：继续把历史第三方 Skill 当默认主线

现在不建议把这些当教程默认入口：

- 历史图像 Skill 名称
- 旧视频脚本命令
- 旧 TTS 子命令路径
- 零散中转站配置

### 坑 2：没区分“同步”和“异步”媒体任务

- 图片、TTS：更接近同步
- 视频、音乐：更接近异步后台任务

### 坑 3：没给媒体模型单独设默认值

主模型能聊天，不代表它就是最适合图片 / 视频 / 音乐的模型。请把这些能力拆开配置。

### 坑 4：直接把创意产物交付，不留人工审核

当前最稳的方式仍然是：

- OpenClaw 负责起草、批处理、编排
- 你负责最终审美、品牌和法务风险判断

---

## 14.8 官方参考

- GitHub Releases：https://github.com/openclaw/openclaw/releases
- Media Overview：https://docs.openclaw.ai/tools/media-overview
- Inference CLI：https://docs.openclaw.ai/cli/infer
- Video Generation：https://docs.openclaw.ai/tools/video-generation
- Music Generation：https://docs.openclaw.ai/tools/music-generation
