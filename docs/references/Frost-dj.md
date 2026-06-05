# Claudio AI DJ 个人专属电台开发文档

## 1. 项目简介
Claudio 是一个旨在提供沉浸式“复古电台”体验的私人 AI DJ 项目。该项目通过整合大语言模型（LLM）、语音合成（TTS）、本地日历/天气服务以及在线音乐 API，打造出一个能够读懂用户听歌习惯、结合当下情绪与环境，并像真实电台 DJ 一样进行语音播报的专属陪伴程序。

本系统采用开放架构蓝图（Specification）的形式，允许开发者自由组合本地服务和外部 API 进行搭建。

---

## 2. 界面与交互设计 (UI/UX)
本项目的交互前端主要以 PWA（Progressive Web App）形式运行，视觉与交互高度参考了 Spotify 的 AI DJ (DJ X) 体验。

*   **拟人化视觉反馈：** 顶部设有 `Speaking...` 状态栏配合绿色呼吸指示灯，背景运用随音频动态跳动的可视化波形（Visualizer），强化 AI 正在发声的沉浸感。
*   **卡片式层叠 UI：** 正在播放的音乐实体（如封面、歌名、艺术家）以白色圆角卡片悬浮于深色动态波形之上，形成强烈的视觉对比。
*   **流式播报字幕：** 屏幕中下方以类似歌词滚动的形式，实时呈现 AI 生成的主持词（Transcript）。

---

## 3. 系统架构设计
系统分为前端交互表层、中枢控制层和运行上下文聚合三大模块。

### 3.1 交互表层 (前端 PWA)
*   **运行环境：** 本地 `LOCALHOST:8080` 或部署为 PWA 应用。
*   **核心功能：** 处理 Player（播放器）、Profile（用户画像）、Settings（设置）三视图的渲染。
*   **性能优化：** 利用 Service Worker (sw) 进行缓存，实现流媒体的 10 秒预加载（prefetch 10s），保证音频播放不卡顿。
*   **通信契约：** 前后端通过 6 条主干线通信：
    *   `POST /api/chat`：处理聊天意图
    *   `GET /api/now`：获取当前播放状态
    *   `GET /api/next`：获取下一首预告
    *   `GET /api/taste`：获取品味配置
    *   `GET /api/plan/today`：获取今日日程
    *   `WS /stream`：通过 WebSocket 推送流式聊天与播放状态

### 3.2 中枢控制层 (本地 Node.js Server)
后端承担业务中转与逻辑分发，由以下核心模块构成：
*   **ROUTER.JS (意图分流)：** 识别用户指令。简单指令直连；音乐检索走音乐 API；自然语言交互交给 Claude 大脑处理。
*   **CONTEXT.JS (提示词组装)：** 负责将用户品味、环境信息、历史记录拼接组装为 System Prompt。
*   **CLAUDE.JS (大脑适配器)：** 通过 `spawn` 子进程调用 Claude 模型。该模块负责解析出结构化的控制命令 `{say, play[], reason, segue}`。
*   **SCHEDULER.JS (节律调度)：** 处理定时任务，如 07:00 每日规划、09:00 早间播报以及每小时的情绪检查。
*   **TTS.JS (声音管线)：** 接收外部 TTS 合成的音频，将其缓存为本地的 `.mp3` 文件并暴露访问路径。
*   **STATE.DB (状态与记忆)：** 负责应用状态持久化，存储会话消息、播放历史和用户偏好，确保跨重启记忆不丢失。

### 3.3 运行时聚合 (Context Window)
每次触发 AI 响应时，系统会按固定格式将 6 片核心信息组装为上下文（Context）交给模型：
1.  **系统提示词** (`prompts/dj-persona.md`)：定义 DJ 人设。
2.  **用户语料** (`user/*.md`)：包含听歌习惯、个人品味等文件。
3.  **环境注入**：当前时间、天气、日历安排。
4.  **已检索记忆**：播放历史 (`state.db` -> plays)。
5.  **用户输入/工具结果**：当前的聊天输入或音乐检索结果。
6.  **执行轨迹**：调度器状态或 Webhook 触发源。

---

## 4. 外部依赖与 API 接入
中枢系统通过调用各司其职的外部 API 来实现完整闭环。

### 4.1 核心依赖
*   **LLM 大脑：** Claude Code (利用子进程调用，通过订阅可免 API Key 直接使用)。
*   **语音合成 (TTS)：** Fish Audio（负责将文本转换为富有情感的主播人声）。
*   **日程与天气：** 飞书 API（读取日程）、OpenWeather（获取天气）。
*   **硬件推送：** UPnP（用于将音乐串流推送至家庭/客厅音响）。

### 4.2 音乐源 API 替换指北 (网易云 -> QQ 音乐)
原架构使用 `NeteaseCloudMusicApi`。若需切换至 QQ 音乐生态，推荐使用开源社区的 Node.js 版接口。

**推荐方案库：**
*   **方案 A (推荐)：** `jsososo/QQMusicApi` (基于 Express/Axios，贴合原版逻辑)。
*   **方案 B：** `Rain120/qq-music-api` (基于 Koa2/TypeScript，自带可视化调试界面)。

**替换施工步骤：**
1.  独立部署 QQ 音乐 API 服务至本地端口（如 `localhost:3300`）。
2.  在 `ROUTER.JS` 中将检索逻辑（Search）更改为向 QQ 音乐服务发起 `songmid` 关键字查询。
3.  重写获取播放直链（Song URL）的逻辑。**注意：部分版权歌曲可能需要配置登录用户的 Cookie。**
4.  将歌词获取接口替换为 QQ 音乐的 Lyric API，抓取文本后继续注入到 `CONTEXT.JS` 供 Claude 解析。

### 4.3 音乐资源获取详解 (封面图 / 播放流 / 歌词)

**核心原则：封面、播放流、歌词从同一个 API 一次获取，不需要额外接入图源。**

#### 4.3.1 QQ 音乐 API 返回数据结构

以 `jsososo/QQMusicApi` 为例，搜索接口 (`/search?key=歌名`) 返回的每首歌曲结构：

```json
{
  "songmid": "0039MnYb0qxYhV",
  "songname": "晴天",
  "singer": ["周杰伦"],
  "albumname": "叶惠美",
  "albummid": "0024bjiL2aocxT",
  "interval": 269,
  "albumcover": "https://y.gtimg.cn/music/photo_new/T002R500x500M0000024bjiL2aocxT.jpg",
  "playurl": "https://...",
  "lyric": "https://..."
}
```

**关键字段映射：**

| 字段 | 用途 | 前端对接 |
|---|---|---|
| `albumcover` | 500×500 专辑封面图 | 渲染到 Frost UI 的 `.album-cover` 组件 |
| `playurl` | 音频播放直链 | 推给前端 `<audio>` 或 Web Audio API |
| `lyric` | 歌词文本 (需二次请求) | 注入 `CONTEXT.JS` 供 Claude 解析播报 |
| `songname` | 歌曲名 | 显示在卡片左上角 `.now-playing` |
| `singer[]` | 歌手名 | 显示在 `.now-playing` 副标题 |

**封面图 URL 拼接规则：**
QQ 音乐封面图支持尺寸参数，`albummid` 可拼接出不同分辨率：
```
https://y.gtimg.cn/music/photo_new/T002R300x300M000{albummid}.jpg   // 300×300
https://y.gtimg.cn/music/photo_new/T002R500x500M000{albummid}.jpg   // 500×500 (推荐)
https://y.gtimg.cn/music/photo_new/T002R800x800M000{albummid}.jpg   // 800×800
```

#### 4.3.2 后端集成流程

```
用户请求 "放一首周杰伦的晴天"
  │
  ▼
ROUTER.JS → 识别为音乐检索意图
  │
  ▼
向 QQ 音乐 API 发起搜索: GET localhost:3300/search?key=晴天 周杰伦
  │
  ▼
取第一条结果的 { albumcover, playurl, songname, singer, lyric }
  │
  ├── albumcover, songname, singer → 推前端 (WebSocket) → 更新 UI
  ├── playurl → 推前端 → <audio> 播放
  └── lyric → CONTEXT.JS → 注入 Claude 上下文
```

#### 4.3.3 版权歌曲 Cookie 配置

部分版权歌曲的 `playurl` 需要登录态。以 QQ 音乐为例：
1. 在 QQ 音乐网页版登录 (https://y.qq.com)
2. F12 → Application → Cookies → 复制 `uin` 和 `qm_keyst` 等关键 Cookie
3. 配置到 QQ 音乐 API 服务的 `.env` 或请求头中

```bash
# QQMusicApi 的配置示例
QQ_UID=123456789
QQ_COOKIE=uin=o0123456789; qm_keyst=xxxxx;
```

#### 4.3.4 网易云音乐 API (备选)

`NeteaseCloudMusicApi` 返回结构类似：

```json
{
  "id": 186016,
  "name": "晴天",
  "ar": [{"name": "周杰伦"}],
  "al": {
    "name": "叶惠美",
    "picUrl": "https://p2.music.126.net/...jpg"
  },
  "url": "https://..."
}
```

- 封面字段路径：`song.al.picUrl`（标准 300×300，可追加 `?param=500y500` 调分辨率）
- 播放直链：`song.url`
- 歌词需要单独请求 `/lyric?id={songId}`

#### 4.3.5 欧美曲库扩展：Spotify Web API (可选)

华语曲库 QQ 音乐 + 网易云已足够。如需欧美内容，可加一层 **Spotify Web API**（免费注册开发者账号即可）：

| 项目 | 说明 |
|---|---|
| 注册 | https://developer.spotify.com → Create App |
| 认证 | Client Credentials Flow (无需用户登录) |
| 搜索端点 | `GET https://api.spotify.com/v1/search?q=track:晴天&type=track` |
| 封面图 | `track.album.images[1].url` (300×300) |
| 预览音频 | `track.preview_url` (30 秒预览) |
| 限制 | 完整播放需 Spotify Premium SDK，预览模式仅 30 秒 |

**集成建议**：在 `ROUTER.JS` 中按优先级 fallback — QQ 音乐 → 网易云 → Spotify，确保曲库覆盖最大化。

---

## 5. 核心执行流 (Execution Flow)

1.  **触发阶段：** 用户发起对话，或定时器（Scheduler）达到预设时间。
2.  **上下文准备：** Node.js 收集当前天气、日程、上一首播放的歌曲信息及用户本地画像，交由 `CONTEXT.JS` 组装。
3.  **大模型推理：** 将组装好的上下文传递给 `CLAUDE.JS`。Claude 进行推理，决定接下来要播放什么风格的歌曲，以及要说些什么。
4.  **结构化输出：** 模型返回包含 `say` (播报文本) 和 `play[]` (歌曲列表) 的 JSON 数据。
5.  **并发执行：**
    *   TTS 管线将 `say` 的内容转换为 MP3 格式。
    *   MUSIC 管线向 QQ 音乐/网易云 API 发起请求，获取 `play[]` 对应的歌曲 URL。
6.  **前端呈现：** 后端将音频资源和文本通过 WebSocket 推送至前端，PWA 更新卡片 UI 并开始播放声音和滚屏字幕。