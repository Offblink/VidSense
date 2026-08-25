# Implementation Plan: 视频理解引擎 (v2)

## Design Reference
`DESIGN.md` (v2)。需求：塞视频(文件/URL) → 本地理解(转写+关键帧) → 调视觉LLM API → 返回结构化内容。

## 已锁决策
- 视觉 LLM：DeepSeek（OpenAI 兼容，可配 base_url/key/model，key 走环境变量）
- 内容粒度：整体摘要 + 时间轴内容 + 主题/关键信息
- ASR：faster-whisper（句级时间戳）
- 设备：两机一致（冻结权重+固定预处理），仅速度不同；本机 CPU 先跑

## 组件地图 (NEW)
```
video-understanding/
  DESIGN.md  PLAN.md  requirements.txt
  vunderstand/
    __init__.py
    config.py        # 集中常量/API 配置        [done]
    schema.py        # 事件卡数据结构            [done]
    ingest.py        # 输入+ffmpeg音频+cv2抽帧+元数据
    frames.py        # CLIP编码 + scene_cut/content_frame/mmr
    transcribe.py    # faster-whisper -> segments
    cards.py         # 合并 -> Grounding Context
    api_client.py    # OpenAI兼容视觉调用
    cli.py           # CLI 入口 (understand 命令)
  output/            # 产物(自动建)
```

## Tasks（顺序）
1. 脚手架：config + schema + requirements + __init__ [done]
2. `ingest.py`：本地文件 → 音频wav + 帧列表 + 时长元数据
3. `frames.py`：FrameEncoder(CLIP) + scene_cut/content_frame/mmr（算法已验证，接入）
4. `transcribe.py`：faster-whisper → segments（支持中文，自动语言）
5. `cards.py`：按时间戳对齐两轴 → EventCard（grounding context）
6. `api_client.py`：DeepSeek 视觉调用（OpenAI 兼容，transcript+事件卡+关键帧图）
7. `cli.py`：端到端 `python -m vunderstand.cli <video>` → 输出结构化内容
8. 阶段2：URL(yt-dlp)、HTTP端点、语义检索

## 执行策略
- 每个任务垂直切片，先跑通再进下一个（incremental）。
- 本地管线(1-5)本机全可测，无需 API key。
- 第6步需要 `DEEPSEEK_API_KEY` 才能端到端验证；无 key 时先验证"上下文构造 + 请求体生成"。

## 全局约束
- 预处理常量只从 config.py 取；不改阈值/fps 的默认值(一致性)。
- 设备自适应：`torch.cuda.is_available()` → cuda/cpu，结果与设备无关。
- 中文输出；代码注释用中文。
- 每个新函数至少一条正常路径验证。
- 模型权重走镜像：`export HF_ENDPOINT=https://hf-mirror.com`。
