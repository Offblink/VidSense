# Design: 视频理解应用 (Video Understanding App) — v2

## 需求（用户原话核心）
> 塞一个视频（文件 **或 URL**）进去，应用能**调用 API**，**结合它自己的理解**，**返回视频内容**。

## 定位
**不做剪辑、不做播放器 UI。** 核心是「视频 → 结构化内容理解」的**理解引擎**。

## 架构（四层，输入 → 理解 → API → 输出）

```
输入: 本地视频文件 或 视频URL(yt-dlp 下载)
   │
   ├─ [理解层] 音频轴: ffmpeg 抽音频 → faster-whisper → 逐段文本+时间戳+说话人
   └─ [理解层] 视觉轴: cv2 抽帧 → CLIP → scene_cut / content_frame / mmr_keyframes
        │  两轴按时间戳对齐 → Grounding Context(事件卡: 时间轴事件)
        ▼
   [API 层] 把 Grounding Context(转写+事件卡+关键帧) 发给 视觉LLM API(OpenAI兼容,可配)
        ▼
   [输出层] 结构化视频内容理解(中文, 文本/JSON)
```

### 关键点
- **"它自己的理解"** = 本地两轴分析出的 **Grounding Context**（转写 + 事件卡 + 关键帧），作为喂给 LLM 的"事实依据"，避免模型幻觉。
- **"调用 API"** = 一个**可配置的 OpenAI 兼容视觉 LLM**（DeepSeek / Gemini / OpenAI / ollama 本地皆可）。`base_url + api_key + model` 从环境变量/配置读取。
- **"返回视频内容"** = 结构化理解：整体摘要 + 时间轴内容事件（每段：字幕/说话人/关键帧/要点）+ 主题/关键信息。

## 事件卡 / Grounding Schema（接口契约）
```json
{
  "video_id": "<md5或文件名>",
  "duration": 123.45,
  "fps_sampled": 1.0,
  "scenes": [ {"start": 0.0, "end": 4.1}, ... ],
  "keyframes": [ {"id": 12, "t": 3.4, "caption": "说话人亮相", "segment_id": 0}, ... ],
  "segments": [
    {
      "id": 0, "start": 3.2, "end": 7.8,
      "speaker": "A" | null,
      "text": "大家好，今天讲视频理解",
      "words": [ {"start": 3.2, "end": 3.5, "text": "大家"}, ... ],
      "frame_id": 12, "scene_index": 0
    }
  ],
  "embeddings": [ ... ]   // keyframe 的 CLIP 向量(检索用)
}
```

## 一致性保证（本地两轴，与你"两机一致"的要求）
CLIP(`openai/clip-vit-base-patch32`) 与 faster-whisper 均用**冻结权重 + 固定预处理常量**（fps/resize/阈值/批大小），设备自适应但结果与设备无关 → **两机输出一致，仅速度不同**。API 层是外部模型，输出由 API 决定，与本地机器无关。

## 组件
1. `ingest.py` — 输入解析：本地文件 / URL（yt-dlp）；ffmpeg 抽音频 + cv2 抽帧 + 元数据。
2. `frames.py` — CLIP `FrameEncoder` + `scene_cut`/`content_frame_per_segment`/`mmr_keyframes`（算法已验证）。
3. `transcribe.py` — faster-whisper → segments（start/end + words + speaker）。
4. `cards.py` — 合并转写与帧信息 → Grounding Context。
5. `api_client.py` — OpenAI 兼容视觉调用（可配 base_url/key/model）。
6. `understand.py` — CLI：video → context → API → 输出（文本/JSON）。

## 实施大纲（增量）
1. 项目结构 + schema/常量（模型名/fps/阈值/路径）。
2. `ingest.py`（先本地文件）。
3. `frames.py`（接入已验证算法，本机 CPU 可跑）。
4. `transcribe.py`（faster-whisper）。
5. `cards.py`（grounding context）。
6. `api_client.py`（OpenAI 兼容）。
7. `understand.py`（端到端 CLI）。
8. 阶段2：URL(yt-dlp) + 向量检索/HTTP 端点。

## Open Questions（需拍板）
1. **视觉 LLM API**：用哪个 provider/model？key 怎么给（env/配置文件）？—— 硬依赖，没有它无法"调用 API"。
2. **"返回视频内容"粒度**：整体摘要？摘要+时间轴内容？摘要+时间轴+主题/关键信息？（我默认第三档）
3. **输出形态**：CLI 打印为主？还是也要 HTTP API 端点（别人可调用）？

## 备注：ASR 已定
faster-whisper（你已选）：CPU 友好、无 HF token 门槛、句级时间戳可靠，本机可跑。词级强制对齐(whisperx)后续按需升级。
