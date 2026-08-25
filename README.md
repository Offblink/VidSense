# 视频理解应用 (Video Understanding App)

把一段视频变成**可理解的内容**：输入一个视频文件或 B站链接，应用先用本地管线「听懂 + 看懂」建立事实依据（语音转写 + CLIP 关键帧），再调用视觉大模型，**返回这段视频的结构化内容理解**（整体摘要 / 时间轴 / 主题与关键信息）。

> 这不是剪辑软件，也不是播放器。它是一个「视频 → 内容理解」引擎。

---

## ✨ 功能

- **输入**：本地视频文件（mp4/mov/...）或 **B站视频 URL**（`bilibili.com/...`、`b23.tv/...`）。
- **本地理解（事实依据）**：
  - 音频轴：ffmpeg 抽音频 → **faster-whisper** 转写，带**词级时间戳**；
  - 视觉轴：cv2 按固定频率抽帧 → **CLIP** 帧编码 → 场景切分 + MMR 多样关键帧；
  - 两轴按时间戳对齐成**事件卡**（每段：文本 + 说话人 + 关键帧 + 场景）。
- **调用视觉大模型**：把「事件卡（字幕时间轴 + 关键帧图）」作为依据发给 **DeepSeek**（OpenAI 兼容，可换成任意兼容端点），返回结构化理解。
- **输出**：结构化中文内容（摘要 + 时间轴 + 主题/观点/关键信息），同时保存事件卡 JSON。

---

## 🏗️ 架构

```
输入: 本地文件 或 B站URL
   │
   ├─ [音频轴] ffmpeg 抽音频 → faster-whisper → 逐段文本 + 词级时间戳 + 说话人
   └─ [视觉轴] cv2 抽帧 → CLIP → scene_cut / MMR 关键帧
        │  (两轴按时间戳对齐)
        ▼
   Grounding Context (事件卡 / Event Card)
        │  字幕时间轴 + 关键帧图
        ▼
   [调用视觉LLM API]  DeepSeek (OpenAI 兼容, 可配 base_url/key/model)
        ▼
   结构化视频内容理解 (摘要 / 时间轴 / 主题关键信息)
```

### 模块（`vunderstand/`）

| 模块 | 作用 |
|---|---|
| `config.py` | 集中配置（模型/fps/阈值/API），支持 `.env`，保证跨机一致 |
| `schema.py` | 事件卡数据结构（EventCard / Segment / Keyframe / Scene / Word） |
| `ingest.py` | ffmpeg 抽音频 + cv2 高效抽帧 + 时长元数据 |
| `transcribe.py` | faster-whisper → 带词级时间戳的分段 |
| `frames.py` | CLIP 帧编码 + `scene_cut` / `content_frame` / `mmr_keyframes` |
| `cards.py` | 两轴对齐 → 事件卡（每段配内容帧 + 场景 + 全局 MMR 关键帧） |
| `source.py` | B站 URL 解析与下载（渐进式 mp4，无则 DASH 分开拉视频/音频再 ffmpeg 合成） |
| `api_client.py` | OpenAI 兼容视觉调用（文字时间轴 + 关键帧图 → 结构化理解） |
| `understand.py` | 端到端编排（build 管线 / understand 全流程） |
| `cli.py` | 命令行入口 |

---

## 📦 安装

```bash
cd <项目目录>
pip install torch transformers faster-whisper opencv-python numpy requests
```

- **模型权重**首次运行自动下载（whisper `small` + CLIP `ViT-B/32`）。若网络受限，先设镜像：
  ```bash
  export HF_ENDPOINT=https://hf-mirror.com
  ```
- 需要 `ffmpeg` 在 `PATH`（抽音频/帧、B站 DASH 合成）。

---

## 🚀 使用

```bash
# 本地文件
python -m vunderstand.cli 视频.mp4

# B站 URL（自动下载）
python -m vunderstand.cli "https://www.bilibili.com/video/BV1YE8b6mEmc/"

# 只跑本地管线（不调 LLM）—— 无需 API key
python -m vunderstand.cli 视频.mp4 --no-api

# 不发送关键帧图（更省 token / 更快）
python -m vunderstand.cli 视频.mp4 --no-images

# 输出结构化 JSON
python -m vunderstand.cli 视频.mp4 --json
```

**示例输出**（`BV1YE8b6mEmc`，小女孩偷开爸爸车的搞笑视频）：

```
1. 整体摘要
一段家庭搞笑视频：一个小女孩趁大人不注意，偷偷爬上汽车驾驶座假装开车。大人发现后
生气质问，女孩却天真回应"我不会开车"。结尾女孩惊觉正被镜头拍下，反问"你在录像吗？"。

2. 时间轴内容
- [0-4.27s]：小女孩爬进驾驶座，好奇地摆弄车内物件。
- [4.27-5.87s]：大人(女性)震惊质问："天哪，你在我的车里干什么？"
- [6.09-7.17s]：小女孩："我不会开车。"
...
3. 主题/观点/关键信息
- 童言无忌与大人抓狂的反差萌...
- 打破第四面墙的喜剧反转...
```

事件卡 JSON（含词级时间戳）会保存到 `output/json/<video>_eventcard.json`，下载/中间产物在 `output/`。

---

## ⚙️ 配置

### 必须：视觉模型 API key

```bash
# 方式一：环境变量
export DEEPSEEK_API_KEY=sk-...

# 方式二：.env 文件（已 gitignore）
# 复制 .env.example 为 .env 并填入
```

### 可调参数（环境变量 / .env）

| 变量 | 默认 | 说明 |
|---|---|---|
| `DEEPSEEK_API_KEY` / `VU_API_KEY` | — | 视觉 LLM key（必填，仅调 API 时需要） |
| `VU_API_BASE_URL` | `https://api.deepseek.com` | OpenAI 兼容端点 |
| `VU_API_MODEL` | `deepseek-v4-flash-vision-exp` | 模型名 |
| `VU_API_MAX_KEYFRAMES` | `12` | 发给模型的关键帧上限 |
| `VU_API_USE_IMAGES` | `1` | 是否发送关键帧图 |
| `VU_FPS` | `1.0` | 抽帧频率（帧/秒） |
| `VU_CLIP_MODEL` | `openai/clip-vit-base-patch32` | CLIP 模型 |
| `VU_ASR_MODEL` | `small` | faster-whisper 模型尺寸 |
| `VU_ASR_LANG` | 自动 | 强制语言（如 `zh`） |
| `VU_ASR_COMPUTE` | `int8` | 计算精度（固定以保两机一致） |
| `VU_CUT` | `0.85` | 场景切分的帧间相似度阈值 |

---

## 🖥️ 跨机一致性（重要）

CLIP 与 faster-whisper 均为**冻结预训练权重**，配合**固定预处理常量**（抽帧 fps、resize、`compute_type=int8`、阈值）与**设备自适应**（有 cuda 用 GPU，否则 CPU）：

> **两台机器输出一致，仅速度不同。**

- CPU/GPU 的浮点微差（~1e-6）远小于判定阈值余量，不会翻转场景切分/关键帧选择。
- 因此：在本机（无独显）验证过的结果，在独显那台用**同一份代码、同一条命令**会得到**相同内容**，只是更快。
- ⚠️ 前提：独显那台也需装依赖、拉权重；若装的是 CPU 版 torch，会静默用 CPU（结果仍对，只是没用上 GPU）。

---

## 🎯 为什么这样设计（防幻觉）

直接让模型"看视频"容易脑补。本项目把**事实依据**前置：

1. **转写 + 时间戳**：台词是硬事实，faster-whisper 给出**词级时间戳**，说话人可区分。
2. **关键帧**：CLIP 从真实像素里抽出**语义多样**的帧（MMR），本地场景切分还原镜头结构。
3. **视觉模型只在依据上描述**：把「字幕时间轴 + 关键帧图」作为输入，让大模型**在给定像素与转写的基础上**总结，而非凭空发挥。

**已用交换实验验证模型确实读像素**：保持同一字幕、只把关键帧换成另一支完全不同的视频（彩色测试卡），模型随即改口描述"测试卡 / 画面与音频不匹配"——证明画面内容来自真实像素，而非从台词脑补。

---

## ⚠️ 限制与后续

- 未做：HTTP API 端点、语义检索 / 问答、说话人分离（pyannote）。
- B站无字幕源转写仍走本地 faster-whisper（不依赖百度 ASR key）。
- 长视频抽帧按 `VU_FPS`，发送给模型的关键帧默认 ≤ 12 张以控制 token。

---

## 📄 License

（未指定）
