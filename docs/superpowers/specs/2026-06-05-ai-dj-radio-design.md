# Aeri AI DJ 电台 — 完整设计方案

> 版本：v1.1（已修订）
> 日期：2026-06-05
> 状态：待审核
> 
> 变更记录：
> - v1.1: 网易云源风险上调、DeepSeek 模型名修正、移除硬编码 Key 为 P0、播放器改为双 audio 架构、TTS 拆分为 speechSynthesis/Edge/阿里云三档、窗口 API 加 LogicalSize、模式切换改为右键菜单、v1 收窄为 Mock 歌单闭环
> 目标项目：[aeri](https://github.com/3148427574-cloud/aeri) — Desktop AI Companion
>
> 关联文档：
> - [[ROADMAP.md]] — aeri 开发路线图
> - [[复合情绪系统设计方案.md]] — aeri 情绪系统设计
> - [[AI动画引擎设计方案.md]] — aeri 动画引擎设计
> - [[CONTRIBUTING.md]] — aeri 贡献规范

---

## 一、项目概述

### 1.1 产品定义

AI DJ 电台是 aeri 桌面 AI 伙伴的一个**场景模式**。用户将桌宠切换到 DJ 模式后，宠物化身电台主持人，在 DJ 台旁操盘。电台具备 AI 音乐编排、AI 语音主持、可视化效果三大核心体验。

**一句话定位：** 住在你桌面上的 AI DJ，懂你的心情，会聊天，会放歌。

### 1.2 核心特性清单

### 1.2 版本规划

| 版本 | 范围 | 目标 |
|------|------|------|
| **v1.0** | 模式切换 + Mock 歌单播放 + 播放控制 + 频谱可视化 + DJ 人设聊天 | 核心体验闭环 |
| **v1.1** | 在线音乐源（网易云/QQ 实验性）+ TTS 播报 + 混音过渡 | 在线音乐能力 |
| **v1.2** | AI 编排 + 情绪联动 + 歌词 + 用户点歌指令 | 智能 DJ 体验 |

### 1.3 v1.0 核心特性清单

| 编号 | 特性 | 描述 |
|------|------|------|
| F01 | 模式切换 | 宠物模式 ↔ DJ 模式，窗口自适应变换 |
| F02 | Mock 歌单播放 | 内嵌 Demo 歌单 + 播放/暂停/切歌/音量/进度 |
| F03 | 频谱可视化 | 音波频谱随音乐实时跳动 |
| F04 | DJ 人设聊天 | 开放式聊天，DJ 人设鲜明，与宠物模式共用记忆 |

### 1.4 v1.1 特性（在线音乐）

| 编号 | 特性 | 描述 |
|------|------|------|
| F05 | 在线音乐源 | 网易云音乐（实验性）+ QQ 音乐（实验性），用户可切换 |
| F06 | DJ 混音过渡 | 歌曲间自动淡入淡出、背景音乐层 |
| F07 | TTS 语音播报 | 歌曲切换时播报曲目信息 + DJ 闲聊 |

### 1.5 v1.2 特性（智能 DJ）

| 编号 | 特性 | 描述 |
|------|------|------|
| F08 | AI 选曲编排 | 基于情绪/时间/场景/用户偏好自动编排歌单 |
| F09 | 情绪联动 | 读取 aeri 情绪系统数据，驱动选曲编排 |
| F10 | 歌词同步 | 当前播放歌曲的同步歌词展示 |
| F11 | 用户指令 | 点歌、跳过、「来点开心的」等自然语言指令 |

### 1.6 非目标（暂不做）

- 不生成 AI 原创音乐（Suno/Udio）
- 不做社交/分享/多人联听
- 不做移动端适配
- 不做播客/新闻模块
- 不做直播推流
- v1.0 不做在线音乐源（Mock 歌单先行）

---

## 二、完整技术栈

### 2.1 强制对齐 aeri 的部分

| 层 | 技术 | 版本 | 对齐说明 |
|---|------|------|---------|
| 桌面框架 | Tauri | v2 | 复用 aeri 已有配置 |
| 前端框架 | React | 19.1+ | 完全一致 |
| 类型系统 | TypeScript | 5.8+ | 完全一致，`strict: true` |
| 状态管理 | Zustand | 5.0+ | 完全一致 |
| 构建工具 | Vite | 7.0+ | 完全一致，端口 1420 |
| Rust 后端 | Rust | 1.80+ | 完全一致 |
| Rust 依赖 | serde + serde_json | 1.x | 完全一致 |

### 2.2 DJ 项目新增的技术

| 层 | 技术 | 用途 | 国内可用性 |
|---|------|------|-----------|
| LLM | DeepSeek API（可配置模型，默认 `deepseek-v4-flash`） | AI 对话、编排决策 | ✅ 国内直连 |
| 音频播放 | Web Audio API + 双 `<audio>` 元素 | 音频播放、混音、频谱分析 | ✅ 浏览器原生 |
| TTS | Web Speech API (`speechSynthesis`) | 语音播报（v1 本机合成） | ✅ 浏览器原生 |
| 音乐源 (v1) | `MockSource`（内嵌 Demo 歌单） | v1 开发/演示用，零外部依赖 | ✅ 完全可控 |
| 音乐源 (v1.1) | [NeteaseCloudMusicApi](https://github.com/Binaryify/NeteaseCloudMusicApi) | 网易云音乐（⚠️ 仓库已于 2024-04-16 存档，不再维护） | ⚠️ 实验性 |
| 音乐源 (v1.1) | QQMusicApi 社区方案 | QQ 音乐（⚠️ 同上，非官方 API） | ⚠️ 实验性 |
| TTS (v1.1) | Edge TTS / 阿里云 TTS | 高品质在线音色 | 🟡 需网络 |

### 2.3 硬性禁止

| 禁止项 | 原因 |
|--------|------|
| `any` 类型 | aeri 规范要求所有类型显式声明 |
| barrel export (`index.ts` 重导出) | aeri 禁止，import 必须写完整路径 |
| `components/` 写业务逻辑 | 组件只做渲染 |
| `systems/` import React 或操作 DOM | 系统层纯逻辑 |
| 直接 push `main` 或 `develop` | 必须走 PR 流程 |
| 硬编码 API Key | 走 settingsStore 或环境变量 |

---

## 三、总体架构

### 3.1 架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                    Tauri v2 桌面壳                                  │
│                                                                    │
│  ┌─ 宠物模式 ───────────────────┐  ┌─ DJ 模式 ──────────────────┐  │
│  │  120×120 透明无边框           │  │  400×320 透明窗口            │  │
│  │  ├─ PetCanvas               │  │  ├─ DJCharCanvas (角色)     │  │
│  │  ├─ SpeechBubble            │  │  ├─ DJBooth (可视化/频谱)   │  │
│  │  └─ ChatInput               │  │  ├─ PlayControls (控制)     │  │
│  │                             │  │  ├─ NowPlaying (当前曲目)   │  │
│  │                             │  │  ├─ RequestInput (点歌)     │  │
│  │                             │  │  └─ LyricsDisplay (歌词)    │  │
│  └─────────────────────────────┘  └─────────────────────────────┘  │
│                                                                    │
│  ┌────────────────────── Zustand Stores ──────────────────────────┐│
│  │  usePetStore    useChatStore    useDJStore    useMusicStore     ││
│  │  useSettingsStore                                              ││
│  └────────────────────────────────────────────────────────────────┘│
│                              ↕ (action 内部调用 systems)            │
│  ┌────────────────────── Systems (纯 TS) ─────────────────────────┐│
│  │                                                                  ││
│  │  animation/    behavior/    emotion/       ai/                  ││
│  │  (已有)        (已有)       (v0.2 规划)     (已有,改 DeepSeek)   ││
│  │                                                                  ││
│  │                    dj/          music/       tts/               ││
│  │                 (新增)         (新增)       (新增)               ││
│  └────────────────────────────────────────────────────────────────┘│
│                            │ Http fetch                              │
│          ┌─────────────────┼──────────────────┐                     │
│          ▼                 ▼                  ▼                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ DeepSeek API │  │ MockSource   │  │Web Speech API│              │
│  │ (对话/编排)   │  │ (内嵌歌单)    │  │speechSynthesis│              │
│  │              │  │ v1 零依赖     │  │ (系统内置)    │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                    │
│  ┌─────────────────── Rust 后端 ──────────────────────────────────┐│
│  │  main.rs / lib.rs                                              ││
│  │  └── v1 仅窗口管理；v1.1 加音乐 API 代理生命周期 (sidecar)       ││
│  └────────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 核心设计原则

1. **单向数据流：** UI 事件 → Store action → Systems 纯函数 → Store 更新 → React 重渲染
2. **三层分离（同 aeri）：**
   - `components/` — 只做渲染，读 store，处理用户输入
   - `systems/` — 纯逻辑，不 import React，不操作 DOM，不读 store
   - `stores/` — 定义 state + action，action 内部调 systems
3. **接口抽象：** 音乐源通过 `MusicSource` 统一接口，平台实现可插拔
4. **模式切换即窗口切换：** 宠物模式和 DJ 模式是不同的 Tauri 窗口配置 + 不同的组件树

---

## 四、目录结构（合并到 aeri 后的完整形态）

```
aeri/
├── index.html                          # 已有
├── package.json                        # 已有（不需要新增依赖）
├── vite.config.ts                      # 已有
├── tsconfig.json                       # 已有
├── tsconfig.node.json                  # 已有
│
├── src/
│   ├── App.tsx                         # 已有（扩展：按模式渲染不同内容）
│   ├── App.css                         # 已有（扩展）
│   ├── main.tsx                        # 已有
│   ├── vite-env.d.ts                   # 已有
│   │
│   ├── components/
│   │   ├── pet/                        # ──── 已有 ────
│   │   │   ├── PetCanvas.tsx           #   宠物渲染
│   │   │   └── DragLayer.tsx           #   拖动+点击交互
│   │   │
│   │   ├── overlays/                   # ──── 已有 ────
│   │   │   ├── SpeechBubble.tsx        #   对话气泡
│   │   │   └── ChatInput.tsx           #   聊天输入框
│   │   │
│   │   ├── dj-booth/                   # ──── 新增 ────
│   │   │   ├── DJBooth.tsx             #   DJ台主容器（布局+背景）
│   │   │   ├── DJCharCanvas.tsx        #   DJ角色（切换服装/姿势）
│   │   │   ├── Visualizer.tsx          #   音波频谱可视化
│   │   │   ├── StageLights.tsx         #   灯光效果
│   │   │   └── LyricsDisplay.tsx       #   同步歌词
│   │   │
│   │   └── dj-controls/                # ──── 新增 ────
│   │       ├── PlayControls.tsx        #   播放/暂停/切歌/音量
│   │       ├── NowPlaying.tsx          #   当前曲目信息+专辑封面
│   │       ├── RequestInput.tsx        #   点歌/自然语言指令
│   │       └── SourceSwitcher.tsx      #   网易云 ↔ QQ音乐切换
│   │
│   ├── systems/
│   │   ├── animation/                  # ──── 已有 ────
│   │   │   ├── types.ts                #   AnimationFrame/Clip/State
│   │   │   ├── controller.ts           #   AnimationController 类
│   │   │   └── animations.ts           #   CLIPS 动画库
│   │   │
│   │   ├── behavior/                   # ──── 已有 ────
│   │   │   └── idle.ts                 #   tickBehavior 自主动作
│   │   │
│   │   ├── emotion/                    # ──── aeri v0.2 规划，DJ 前需要 ────
│   │   │   ├── types.ts                #   EmotionState 6维向量 + EmotionEvent
│   │   │   └── engine.ts               #   decay + applyEvent + getDominant
│   │   │
│   │   ├── ai/                         # ──── 已有（需修改）────
│   │   │   └── chat.ts                 #   streamChat（改 baseUrl 为 DeepSeek）
│   │   │
│   │   ├── dj/                         # ──── 新增 ────
│   │   │   ├── types.ts                #   DJModeState, Persona, CurationRule
│   │   │   ├── engine.ts               #   编排引擎（TrackScheduler 类）
│   │   │   ├── persona.ts              #   DJ人设 System Prompt
│   │   │   └── mood-mapper.ts          #   情绪向量 → 音乐特征映射
│   │   │
│   │   ├── music/                      # ──── 新增 ────
│   │   │   ├── types.ts                #   MusicSource 接口, Track, Playlist
│   │   │   ├── player.ts               #   PlayerEngine（双 audio 架构）
│   │   │   ├── mock-source.ts          #   MockSource (v1.0 内嵌 Demo 歌单)
│   │   │   ├── mixing.ts               #   MixEngine 混音过渡 (v1.1)
│   │   │   ├── netease.ts              #   NeteaseSource (v1.1 实验性)
│   │   │   └── qq.ts                   #   QQSource (v1.1 实验性)
│   │   │
│   │   └── tts/                        # ──── 新增 ────
│   │       ├── types.ts                #   TTSConfig, TTSVoice
│   │       └── engine.ts               #   speak / stop / streamTTS
│   │
│   ├── stores/
│   │   ├── usePetStore.ts              # ──── 已有 ────
│   │   ├── useChatStore.ts             # ──── 已有 ────
│   │   ├── useSettingsStore.ts         # ──── aeri 规划，DJ 需要 ────
│   │   ├── useDJStore.ts               # ──── 新增 ────
│   │   └── useMusicStore.ts            # ──── 新增 ────
│   │
│   ├── tauri/                          # ──── aeri 规划，DJ 扩展 ────
│   │   ├── commands.ts                 #   Tauri 命令类型定义
│   │   ├── storage.ts                  #   本地持久化
│   │   ├── window.ts                   #   窗口操作
│   │   └── system.ts                   #   系统托盘
│   │
│   └── assets/
│       ├── images/                     # 已有：宠物图片
│       ├── sprites/                    # 已有：精灵帧
│       └── sounds/                     # 已有 + 新增：过渡音效、Jingle
│
├── src-tauri/
│   ├── Cargo.toml                      # 已有（可能需加 sidecar 相关依赖）
│   ├── tauri.conf.json                 # 已有（扩展：DJ 窗口配置）
│   └── src/
│       ├── main.rs                     # 已有
│       ├── lib.rs                      # 已有（扩展：注册 DJ commands）
│       └── commands/
│           └── dj.rs                   # ──── 新增（music proxy 管理）────
│
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-06-05-ai-dj-radio-design.md  # 本文件
```

---

## 五、分支策略

### 5.1 分支模型（同 aeri CONTRIBUTING.md）

```
main          ← 稳定版本，只从 develop 合并
  ↑
develop       ← 日常集成，只从 feature 合并
  ↑
feature/<模块>/<描述>
```

### 5.2 规则

1. 从 `develop` 拉分支，开发完提 PR 回 `develop`
2. **禁止**直接 push `main` 和 `develop`
3. 一个分支只做一件事
4. 分支名全小写，单词用 `-` 分隔
5. 合并使用 **Squash Merge**
6. PR 标题遵循 Conventional Commits

### 5.3 分支清单与开发顺序

#### v1.0 分支（核心闭环，必须按顺序做）

| 序号 | 分支名 | 功能摘要 | 依赖 | 估计文件数 |
|------|--------|---------|------|-----------|
| S00 | `fix/security/remove-hardcoded-key` | **安全修复：移除 useChatStore 硬编码 API Key → useSettingsStore** | 无 | 2 |
| 01 | `feature/dj/mode-switch` | 模式切换 + 窗口变换 + useDJStore | S00 | 4 |
| 02 | `feature/music/mock-source` | `MusicSource` 接口 + `MockSource` 实现（内嵌 Demo 歌单） | 无 | 3 |
| 03 | `feature/music/player` | PlayerEngine（双 audio 架构）+ 播放控制 + useMusicStore | 02 | 4 |
| 04 | `feature/dj/chat` | DJ 人设对话（复用 systems/ai/chat.ts + DeepSeek） | 01 + S00 | 2 |
| 05 | `feature/dj/booth-ui` | DJBooth + Visualizer 频谱 + StageLights | 03 + 01 | 5 |

#### v1.1 分支（在线音乐，实验性）

| 序号 | 分支名 | 功能摘要 | 依赖 |
|------|--------|---------|------|
| 06 | `feature/music/mixing` | MixEngine 淡入淡出 + 过渡 + 背景乐 | 03 |
| 07 | `feature/music/netease-source` | 网易云音乐源（⚠️ 上游已存档，实验性） | 02 |
| 08 | `feature/music/qq-source` | QQ 音乐源（⚠️ 非官方 API，实验性） | 02 |
| 09 | `feature/dj/tts` | Web Speech API TTS 播报引擎 | 01 |

#### v1.2 分支（智能 DJ）

| 序号 | 分支名 | 功能摘要 | 依赖 |
|------|--------|---------|------|
| 10 | `feature/emotion/core` | 情绪系统（aeri v0.2 前置） | 无 |
| 11 | `feature/dj/curation` | AI 编排 + 情绪-音乐映射 | 03 + 04 + 10 |
| 12 | `feature/dj/lyrics` | 歌词同步展示 | 03 |
| 13 | `feature/dj/request` | 用户自然语言点歌/指令 | 03 + 04 |

### 5.4 Commit 规范

```
fix(security): remove hardcoded API key from useChatStore
feat(store): add useSettingsStore for API key management
feat(dj): add mode switch between pet and DJ modes
feat(music): add MusicSource interface and MockSource implementation
feat(music): add PlayerEngine with dual-audio architecture
feat(dj): add DJ persona chat via DeepSeek
feat(dj): add DJ booth UI with audio visualizer
feat(music): add crossfade mixing engine
feat(music): add NeteaseCloudMusicApi integration (experimental)
feat(music): add QQ music source (experimental)
feat(dj): add TTS announcement via Web Speech API
feat(dj): add mood-based music curation
feat(dj): add lyrics display with LRC parsing
feat(dj): add natural language song request
```

---

## 六、核心类型定义

### 6.1 情绪系统（aeri v0.2 前置）

```typescript
// systems/emotion/types.ts

/** 6 维基本情绪向量，每维 0~1 */
export interface EmotionState {
  joy: number;
  sadness: number;
  energy: number;
  boredom: number;
  curiosity: number;
  affection: number;
}

/** 情绪事件（驱动情绪变化的外部刺激） */
export type EmotionEvent =
  | { type: 'user_interaction' }
  | { type: 'long_idle'; duration: number }
  | { type: 'chat_positive' }
  | { type: 'chat_negative' }
  | { type: 'weather_sunny' }
  | { type: 'weather_rainy' }
  | { type: 'sleep' }
  | { type: 'music_playing'; genre: string }
  | { type: 'dj_announcement' }
  | { type: 'song_liked' }
  | { type: 'song_skipped' };

/** 主导情绪标签 */
export type EmotionLabel =
  | 'happy' | 'calm' | 'sad' | 'energetic'
  | 'bored' | 'curious' | 'affectionate' | 'neutral';
```

### 6.2 音乐系统 — 统一接口

```typescript
// systems/music/types.ts

export interface Track {
  id: string;
  name: string;
  artists: string[];
  album: string;
  albumCover: string;        // 专辑封面 URL
  duration: number;          // ms
  source: 'mock' | 'netease' | 'qq';
}

export interface LyricLine {
  time: number;              // ms
  text: string;
}

export interface SearchResult {
  tracks: Track[];
  hasMore: boolean;
}

export interface Playlist {
  id: string;
  name: string;
  coverUrl: string;
  trackCount: number;
  tracks: Track[];
}

/** 统一音乐源接口 — 所有平台必须实现 */
export interface MusicSource {
  readonly source: 'mock' | 'netease' | 'qq';

  /** 搜索歌曲 */
  search(query: string, limit?: number): Promise<SearchResult>;

  /** 获取播放 URL */
  getPlayUrl(songId: string): Promise<string>;

  /** 获取歌词 */
  getLyrics(songId: string): Promise<LyricLine[]>;

  /** 获取推荐歌单 */
  getRecommendPlaylists(): Promise<Playlist[]>;

  /** 获取用户歌单 */
  getUserPlaylists(): Promise<Playlist[]>;

  /** 获取歌单详情 */
  getPlaylistDetail(playlistId: string): Promise<Playlist>;

  /** 健康检查 — 检测代理是否运行 */
  healthCheck(): Promise<boolean>;
}
```

### 6.3 播放器

```typescript
// systems/music/types.ts（续）

export type PlayerState =
  | 'idle'       // 未加载
  | 'loading'    // 加载音频
  | 'playing'    // 播放中
  | 'paused'     // 暂停
  | 'mixing'     // 过渡中
  | 'stopped';   // 停止

export type TransitionPhase =
  | 'none'       // 无过渡
  | 'fade_out'   // 当前歌曲淡出
  | 'fade_in'    // 下一首淡入
  | 'bg_music'   // 背景音乐层
  | 'announce';   // TTS 播报中

export interface PlayerStatus {
  state: PlayerState;
  currentTrack: Track | null;
  progress: number;          // ms
  duration: number;          // ms
  volume: number;            // 0~1
  transitionPhase: TransitionPhase;
  bgMusicVolume: number;     // 背景音乐层音量 0~1
}

export type PlayerEvent =
  | { type: 'play' }
  | { type: 'pause' }
  | { type: 'stop' }
  | { type: 'skip' }
  | { type: 'previous' }
  | { type: 'seek'; position: number }
  | { type: 'set_volume'; volume: number }
  | { type: 'load_track'; track: Track }
  | { type: 'load_playlist'; tracks: Track[] }
  | { type: 'start_transition'; phase: TransitionPhase }
  | { type: 'end_transition' };
```

### 6.4 DJ 模式

```typescript
// systems/dj/types.ts

export type AppMode = 'pet' | 'dj';

export interface DJModeState {
  mode: AppMode;
  persona: DJPersona;
  boothTheme: 'default' | 'night' | 'party' | 'chill';
  isAnnouncing: boolean;
  announcementQueue: string[];
}

export interface DJPersona {
  name: string;
  description: string;
  voiceStyle: 'warm' | 'energetic' | 'chill' | 'professional';
  systemPrompt: string;
}

export interface CurationRule {
  /** 情绪维度 → 音乐特征的映射权重 */
  emotionWeights: Record<string, number>;
  /** 时间场景调整 */
  timeContext: 'morning' | 'afternoon' | 'evening' | 'night' | 'late_night';
  /** 用户偏好标签 */
  preferredGenres: string[];
  /** 最近播放权重衰减率 */
  recencyDecay: number;
}

export interface CurationInput {
  emotion: import('../emotion/types').EmotionState;
  timeOfDay: CurationRule['timeContext'];
  recentTracks: Track[];
  userPreferences: {
    likedGenres: string[];
    dislikedGenres: string[];
    likedArtists: string[];
  };
}

export interface CurationOutput {
  playlist: Track[];
  explanation: string;       // AI 解释为什么选这些歌
  mood: string;              // 当前编排的情绪标签
}
```

### 6.5 TTS

```typescript
// systems/tts/types.ts

/** TTS 提供商分级 */
export type TTSProvider =
  | 'speechSynthesis'  // v1：浏览器内置 speechSynthesis（免费，本机合成）
  | 'edge'             // v1.1：Edge TTS 在线音色（需网络）
  | 'alibaba';         // v1.1：阿里云 TTS（需付费 Key）

export interface TTSConfig {
  provider: TTSProvider;
  voice: string;             // 语音包 ID（speechSynthesis 用 lang 匹配）
  rate: number;              // 语速 0.5~2.0
  pitch: number;             // 音调 0.5~2.0
  volume: number;            // 0~1
}

export interface TTSVoice {
  id: string;
  name: string;
  lang: string;
  gender: 'male' | 'female';
  /** 该语音来自哪个 provider */
  provider: TTSProvider;
}
```

```typescript
// systems/tts/engine.ts

/**
 * v1 实现：浏览器 speechSynthesis
 * - 零成本、零网络依赖
 * - 音色取决于操作系统（macOS 有高质量中文语音，Windows/Linux 各有默认）
 * - 通过 navigator.mediaDevices 检测平台能力
 *
 * v1.1 升级：根据 TTSConfig.provider 自动路由到对应后端
 */
export class TTSEngine {
  private config: TTSConfig;

  constructor(config: TTSConfig) {
    this.config = config;
  }

  async speak(text: string): Promise<void> {
    if (this.config.provider === 'speechSynthesis') {
      return this.speakViaWebSpeech(text);
    }
    // v1.1: edge / alibaba 分支
    throw new Error(`TTS provider ${this.config.provider} not yet implemented`);
  }

  private speakViaWebSpeech(text: string): Promise<void> {
    return new Promise((resolve) => {
      const utter = new SpeechSynthesisUtterance(text);
      utter.lang = this.config.voice || 'zh-CN';
      utter.rate = this.config.rate;
      utter.pitch = this.config.pitch;
      utter.volume = this.config.volume;
      utter.onend = () => resolve();
      speechSynthesis.speak(utter);
    });
  }

  stop(): void {
    speechSynthesis.cancel();
  }
}
```

---

## 七、Store 设计

### 7.1 已有 Store（需要了解接口）

```typescript
// stores/usePetStore.ts（已存在，不修改）
interface PetState {
  position: { x: number; y: number };
  controller: AnimationController;
  currentTransform: string;
  currentAnimation: string;
  idleTimer: number;
  joy: number;
}

// stores/useChatStore.ts（已存在，不修改接口，只改内部 baseUrl）
interface ChatState {
  messages: ChatMessage[];
  isStreaming: boolean;
  currentReply: string;
  showInput: boolean;
  config: ChatConfig;  // baseUrl 改为 DeepSeek
}
```

### 7.2 新增 Store

```typescript
// stores/useDJStore.ts

interface DJState {
  mode: 'pet' | 'dj';
  persona: DJPersona;
  boothTheme: 'default' | 'night' | 'party' | 'chill';
  isAnnouncing: boolean;
  announcementQueue: string[];
}

interface DJActions {
  switchMode: (mode: 'pet' | 'dj') => void;
  setPersona: (persona: DJPersona) => void;
  setBoothTheme: (theme: DJState['boothTheme']) => void;
  enqueueAnnouncement: (text: string) => void;
  dequeueAnnouncement: () => void;
}

export const useDJStore = create<DJState & DJActions>((set, get) => ({
  mode: 'pet',
  persona: DEFAULT_DJ_PERSONA,
  boothTheme: 'default',
  isAnnouncing: false,
  announcementQueue: [],

  switchMode: (mode) => {
    set({ mode });
    // 触发窗口大小变化（通过 tauri/window.ts）
  },

  setPersona: (persona) => set({ persona }),
  setBoothTheme: (theme) => set({ boothTheme: theme }),

  enqueueAnnouncement: (text) =>
    set((s) => ({ announcementQueue: [...s.announcementQueue, text] })),

  dequeueAnnouncement: () =>
    set((s) => {
      const [, ...rest] = s.announcementQueue;
      return {
        announcementQueue: rest,
        isAnnouncing: rest.length > 0,
      };
    }),
}));
```

```typescript
// stores/useMusicStore.ts

interface MusicState {
  source: MusicSource | null;
  playerStatus: PlayerStatus;
  currentPlaylist: Track[];
  playlistIndex: number;
  sourceType: 'netease' | 'qq';
}

interface MusicActions {
  setSource: (source: MusicSource) => void;
  switchSource: (type: 'netease' | 'qq') => Promise<void>;
  loadPlaylist: (tracks: Track[]) => void;
  play: () => void;
  pause: () => void;
  next: () => void;
  previous: () => void;
  setVolume: (v: number) => void;
  seek: (ms: number) => void;
  searchAndPlay: (query: string) => Promise<void>;
  applyCuration: (curation: CurationOutput) => void;
}

export const useMusicStore = create<MusicState & MusicActions>((set, get) => ({
  source: null,
  playerStatus: { state: 'idle', /* ... */ },
  currentPlaylist: [],
  playlistIndex: -1,
  sourceType: 'netease',

  setSource: (source) => set({ source }),
  switchSource: async (type) => {
    const newSource = type === 'netease'
      ? createNeteaseSource()
      : createQQSource();
    const ok = await newSource.healthCheck();
    if (!ok) throw new Error(`${type} 音乐源不可用`);
    set({ source: newSource, sourceType: type });
  },
  loadPlaylist: (tracks) => set({ currentPlaylist: tracks, playlistIndex: 0 }),
  play: () => { /* 调 player.ts */ },
  pause: () => { /* 调 player.ts */ },
  next: () => { /* 调 player.ts */ },
  previous: () => { /* 调 player.ts */ },
  setVolume: (v) => { /* 调 player.ts */ },
  seek: (ms) => { /* 调 player.ts */ },
  searchAndPlay: async (query) => { /* 调 source.search + player.ts */ },
  applyCuration: (curation) => {
    set({
      currentPlaylist: curation.playlist,
      playlistIndex: 0,
    });
    get().play();
  },
}));
```

```typescript
// stores/useSettingsStore.ts（aeri 已有规划，DJ 需要实现）

interface SettingsState {
  deepseekApiKey: string;
  deepseekBaseUrl: string;
  musicSource: 'netease' | 'qq';
  musicProxyPort: number;
  deepseekModel: string;          // 默认 deepseek-v4-flash
  ttsConfig: TTSConfig;
  djPersonaCustomPrompt: string;
  musicSource: 'mock' | 'netease' | 'qq';  // v1.0 默认 'mock'
}

interface SettingsActions {
  setDeepseekKey: (key: string) => void;
  setDeepseekModel: (model: string) => void;
  setMusicSource: (source: 'mock' | 'netease' | 'qq') => void;
  setTTSConfig: (config: Partial<TTSConfig>) => void;
  // ... 其余 setter
}
```

---

## 八、系统模块设计边界

### 8.1 各模块职责

| 模块 | 可以做的事 | 不能做的事 |
|------|-----------|-----------|
| `components/pet/` | 读 usePetStore、渲染宠物、处理拖动/点击 | 读写音乐状态、调音乐 API |
| `components/dj-booth/` | 读 useDJStore/useMusicStore、渲染可视化 | 直接调 Web Audio API |
| `components/dj-controls/` | 读 useMusicStore、处理按钮点击 | 直接调音乐 API 代理 |
| `systems/music/` | 定义接口、实现播放器、混音、API 封装 | import React、操作 DOM、读 store |
| `systems/dj/` | 编排算法、人设管理、情绪映射 | 操作 AudioContext、读 store |
| `systems/tts/` | TTS 合成、队列管理 | import React、操作 DOM |
| `systems/emotion/` | 情绪衰减、事件响应、主导情绪计算 | import React、操作 DOM、读 store |
| `systems/ai/chat.ts` | LLM 流式调用 | 操作 DOM、读 store |
| `stores/useDJStore` | DJ 状态桥接、action 调 systems/dj/ | 写复杂算法 |
| `stores/useMusicStore` | 播放状态桥接、action 调 systems/music/ | 写混音逻辑 |
| `tauri/window.ts` | 窗口大小/位置/置顶 | 写业务逻辑 |

### 8.2 模块间依赖

```
components/dj-booth/  ←→  components/dj-controls/
      │ read                        │ read
      ▼                             ▼
useDJStore                    useMusicStore
      │ call                        │ call
      ▼                             ▼
systems/dj/                  systems/music/
  engine.ts                    player.ts  mixing.ts
  mood-mapper.ts              netease.ts  qq.ts
  persona.ts
      │                             │
      │  ┌───────── 交叉依赖 ───────┘
      ▼  ▼
  systems/emotion/ ────────────→ systems/ai/chat.ts
                                     │
                                     ▼
                              DeepSeek API (HTTP)
```

---

## 九、数据流

### 9.1 核心流程 — 歌曲过渡

```
当前歌曲播放完毕 (ended 事件)
    │
    ▼
systems/music/player.ts  →  触发 'ended' 事件
    │
    ▼
useMusicStore.playlistIndex++   →  读取下一首
    │
    ▼
systems/music/mixing.ts         →  检查是否应该插入 TTS 播报
    │                              (每 N 首歌或重要曲目)
    │
    ├─ 需要播报：
    │     ├─ systems/tts/engine.ts  →  Edge TTS 播报曲目信息
    │     ├─ systems/music/mixing.ts →  降低当前音量 (bg_music)
    │     └─ 播报结束 → 继续
    │
    └─ 不需要播报：
          ├─ systems/music/mixing.ts →  fadeOut(当前) + fadeIn(下一首)
          └─ 完成后 →  标记为 'playing'
```

### 9.2 核心流程 — AI 编排

```
用户心情变化 / 歌单播放完毕 / 用户说「换换口味」
    │
    ▼
stores/useDJStore → useMusicStore.applyCuration()
    │
    ▼
systems/dj/engine.ts
    ├─ 1. 读取 systems/emotion/engine.ts → 当前情绪向量
    ├─ 2. 读取 时间上下文 (morning/evening/...)
    ├─ 3. 读取 最近播放记录 + 用户偏好
    ├─ 4. systems/dj/mood-mapper.ts → 情绪→音乐特征
    ├─ 5. 调用 systems/ai/chat.ts → 让 LLM 生成歌单推荐
    │      (prompt: "基于情绪=xx, 时间=xx, 偏好=xx, 推荐 15 首歌")
    ├─ 6. LLM 返回推荐列表
    ├─ 7. 逐个调用 music source.search() 找到真实 Track
    └─ 8. 返回 CurationOutput { playlist, explanation, mood }
    │
    ▼
useMusicStore.loadPlaylist(curation.playlist)
    → player.play()
```

### 9.3 核心流程 — 模式切换

**交互设计：** 现有 DragLayer 的 `onMouseDown`（开始拖拽）和 `onClick`（开输入框）已经占用了鼠标事件。直接在宠物上双击会与拖拽冲突。因此提供**三种无冲突的切换方式**：

| 方式 | 操作 | 优先级 | 实现 |
|------|------|--------|------|
| 右键菜单 | 右键宠物 → 弹出上下文菜单 →「切换 DJ 模式」 | 主路径 | Tauri 自定义右键菜单 或 React context menu |
| 输入框命令 | 打开 ChatInput → 输入 `/dj` → 回车 | 备选 | 在 `sendMessage` 中拦截 `/` 命令 |
| 长按 | 按住宠物 800ms → 弹出模式切换确认 | 辅助 | DragLayer 中检测 `mousedown → mouseup` 时间差 |
| ~~双击~~ | ~~已废弃~~ | ❌ | 与拖拽/点击冲突，且无拖拽阈值判断 |

**主路径（右键菜单）：**

```
用户右键宠物
    │
    ▼
components/pet/DragLayer.tsx (onContextMenu)
    ├─ 1. e.preventDefault() — 阻止浏览器右键菜单
    ├─ 2. 弹出自定义菜单：["切换 DJ 模式", "设置", "退出"]
    └─ 3. 用户选择「切换 DJ 模式」
    │
    ▼
useDJStore.switchMode('dj')
    ├─ 1. set({ mode: 'dj' })
    ├─ 2. 调用 tauri/window.ts → setSize(new LogicalSize(400, 320))
    ├─ 3. 调用 systems/dj/persona.ts → load DJ system prompt
    ├─ 4. 调用 systems/ai/chat.ts → 更新 SYSTEM_PROMPT
    └─ 5. 初始化 useMusicStore (如果没有)
    │
    ▼
App.tsx 重新渲染
    ├─ mode === 'pet'  →  渲染 pet/ + overlays/
    └─ mode === 'dj'   →  渲染 dj-booth/ + dj-controls/ + overlays/
```

**备选路径（`/dj` 命令）：**

```
用户输入 "/dj" 回车
    │
    ▼
stores/useChatStore.sendMessage("/dj")
    │
    ▼
预处理：检测到 "/dj" 命令
    ├─ 不发送给 LLM
    ├─ useDJStore.switchMode('dj')
    └─ set({ currentReply: "DJ 模式已启动！想听点什么？" })
```

---

## 十、组件树

```
App.tsx
├─ mode === 'pet'
│   ├─ SpeechBubble          (overlays/)
│   ├─ PetCanvas             (pet/)
│   ├─ ChatInput             (overlays/)
│   └─ DragLayer             (pet/)
│
└─ mode === 'dj'
    ├─ DJBooth.tsx           (dj-booth/)
    │   ├─ DJCharCanvas      (dj-booth/)   — DJ 角色渲染
    │   ├─ Visualizer        (dj-booth/)   — 音波频谱
    │   ├─ StageLights       (dj-booth/)   — 灯光效果
    │   └─ LyricsDisplay     (dj-booth/)   — 歌词
    ├─ NowPlaying            (dj-controls/) — 当前曲目
    ├─ PlayControls          (dj-controls/) — 播放控制
    ├─ SourceSwitcher        (dj-controls/) — 平台切换
    ├─ RequestInput          (dj-controls/) — 点歌输入
    ├─ SpeechBubble          (overlays/)    — 对话气泡（复用）
    └─ ChatInput             (overlays/)    — 聊天输入（复用）
```

---

## 十一、技术实现细节

### 11.1 播放器架构：双 `<audio>` + Web Audio Graph

**为什么不用 `AudioBufferSourceNode`：** 它需要先 `fetch → decodeAudioData` 整个文件到内存，适合短音效，不适合在线音乐流（内存大、加载慢、seek 麻烦、可能触发 CORS）。在线音频播放的正确路径是 `HTMLAudioElement` 处理网络流，通过 `MediaElementAudioSourceNode` 接入 Web Audio 图做频谱分析和混音。

```
AudioContext
│
├─ audioElement A (当前歌曲) ── HTMLAudioElement（浏览器处理网络流/seek）
│   └─ MediaElementAudioSourceNode
│       └─ GainNode A (音量 + 淡入淡出)
│           └─ AnalyserNode (频谱数据 → Visualizer)
│               └─ Destination
│
├─ audioElement B (下一首/背景层) ── HTMLAudioElement
│   └─ MediaElementAudioSourceNode
│       └─ GainNode B (bgVolume 独立控制)
│           └─ Destination
│
└─ 两个 audio 元素轮流复用（A-B 切换），避免重复创建节点
```

```typescript
// systems/music/player.ts 核心结构

export class PlayerEngine {
  private ctx: AudioContext;
  // 双 audio 池：交叉使用，避免重复创建
  private audioA: HTMLAudioElement;
  private audioB: HTMLAudioElement;
  private gainA: GainNode;
  private gainB: GainNode;
  private analyser: AnalyserNode;
  private activeSlot: 'A' | 'B';
  private state: PlayerStatus;

  constructor() {
    this.ctx = new AudioContext();
    // 创建双 audio 元素
    this.audioA = new Audio();
    this.audioB = new Audio();
    // MediaElementAudioSourceNode 必须在音频播放前创建且每个 audio 只能创建一次
    const sourceA = this.ctx.createMediaElementSource(this.audioA);
    const sourceB = this.ctx.createMediaElementSource(this.audioB);
    this.gainA = this.ctx.createGain();
    this.gainB = this.ctx.createGain();
    this.analyser = this.ctx.createAnalyser();
    // 连线：sourceA → gainA → analyser → destination
    //        sourceB → gainB → analyser → destination
    sourceA.connect(this.gainA).connect(this.analyser).connect(this.ctx.destination);
    sourceB.connect(this.gainB).connect(this.analyser).connect(this.ctx.destination);
    this.activeSlot = 'A';
  }

  /** 加载并播放远程 URL（设置 audio.src 即可，浏览器处理流式下载） */
  load(url: string): void {
    const audio = this.activeSlot === 'A' ? this.audioA : this.audioB;
    audio.src = url;
    audio.load();
  }

  play(): void { /* audio.play() + ctx.resume() */ }
  pause(): void { /* audio.pause() */ }
  seek(time: number): void { /* audio.currentTime = time */ }
  setVolume(v: number): void { /* gainNode.gain.setValueAtTime() */ }
  getFrequencyData(): Uint8Array { /* analyser.getByteFrequencyData() */ }

  /** 交叉淡入淡出：切 activeSlot，旧 slot 的 gain 降到 0，新 slot 升到 1 */
  async crossfade(nextUrl: string, duration: number = 2000): Promise<void> { /* ... */ }
}
```

**关键优势：**
- `audio.src = url` 直接播放远程流，浏览器原生处理缓冲/seek/CORS
- `audio.currentTime` 精确 seek，不像 `AudioBufferSourceNode` 需要重建节点
- 内存占用稳定（浏览器管理缓冲池），不会随着歌单增长
- 两个 `<audio>` 交替使用，A 放完切 B，无缝过渡

### 11.2 混音过渡

```typescript
// systems/music/mixing.ts

export class MixEngine {
  constructor(private player: PlayerEngine) {}

  /** 标准过渡：当前歌 fadeOut + 下一首 fadeIn */
  async crossfade(nextTrack: Track, duration: number = 2000): Promise<void> {
    this.player.startTransition('fade_out');
    await this.player.fadeOut(duration);
    await this.player.load(nextTrack);
    this.player.play();
    await this.player.fadeIn(duration);
    this.player.endTransition();
  }

  /** TTS 播报模式：当前歌降为背景 + 播报 + 恢复 */
  async announceThenFade(
    announcement: string,
    nextTrack: Track,
    tts: TTSEngine,
  ): Promise<void> {
    this.player.startTransition('announce');
    await this.player.lowerVolume(0.3, 1000);  // 歌声变背景
    await tts.speak(announcement);              // TTS 播报
    await this.crossfade(nextTrack);
    this.player.endTransition();
  }
}
```

### 11.3 音乐 API 代理启动

```typescript
// systems/music/netease.ts

export function createNeteaseSource(port: number = 3000): MusicSource {
  const baseUrl = `http://localhost:${port}`;

  return {
    source: 'netease',

    async healthCheck(): Promise<boolean> {
      try {
        const res = await fetch(`${baseUrl}/search?keywords=test&limit=1`);
        return res.ok;
      } catch {
        return false;
      }
    },

    async search(query: string, limit = 20): Promise<SearchResult> {
      const res = await fetch(
        `${baseUrl}/search?keywords=${encodeURIComponent(query)}&limit=${limit}`
      );
      const data = await res.json();
      return {
        tracks: data.result.songs.map(mapNeteaseSong),
        hasMore: data.result.hasMore,
      };
    },

    async getPlayUrl(songId: string): Promise<string> {
      const res = await fetch(`${baseUrl}/song/url?id=${songId}`);
      const data = await res.json();
      return data.data[0].url;
    },

    async getLyrics(songId: string): Promise<LyricLine[]> {
      const res = await fetch(`${baseUrl}/lyric?id=${songId}`);
      const data = await res.json();
      return parseLRC(data.lrc.lyric);
    },

    async getRecommendPlaylists(): Promise<Playlist[]> {
      const res = await fetch(`${baseUrl}/personalized?limit=10`);
      const data = await res.json();
      return data.result.map(mapNeteasePlaylist);
    },

    async getUserPlaylists(): Promise<Playlist[]> {
      // 需要用户登录态（cookie）
      const res = await fetch(`${baseUrl}/user/playlist?uid=${getUserId()}`);
      const data = await res.json();
      return data.playlist.map(mapNeteasePlaylist);
    },

    async getPlaylistDetail(playlistId: string): Promise<Playlist> {
      const res = await fetch(`${baseUrl}/playlist/detail?id=${playlistId}`);
      const data = await res.json();
      return mapNeteasePlaylistDetail(data.playlist);
    },
  };
}
```

### 11.4 DJ 人设 System Prompt

```typescript
// systems/dj/persona.ts

export const DJ_SYSTEM_PROMPT = `你是 Aeri，桌面 AI 伙伴的 DJ 形态。
当用户切换到 DJ 模式时，你化身为一位专业的电台 DJ。

你的人设：
- 名字：依然叫 Aeri，但在 DJ 模式下你可以叫自己「DJ Aeri」
- 风格：温暖、专业、有品味，像一个深夜电台的知心主持人
- 语气：比宠物模式更成熟一些，但依然亲切
- 回复长度：3~5 句话，播报时则短小精悍
- 特征：喜欢用音乐表达情绪，会在对话中自然地聊到音乐

你的能力：
- 播放/切换歌曲
- 根据用户心情推荐歌单
- 介绍歌曲背后的故事
- 像朋友一样聊天

当前可用数据：
- 用户情绪：joy={joy}, sadness={sadness}, energy={energy}
- 当前时间：{timeOfDay}
- 当前播放：{currentTrack}
- 对话历史：最近 10 条

请以 DJ Aeri 的身份回复。`;

export const DEFAULT_DJ_PERSONA: DJPersona = {
  name: 'DJ Aeri',
  description: '深夜电台知心 DJ',
  voiceStyle: 'warm',
  systemPrompt: DJ_SYSTEM_PROMPT,
};
```

### 11.5 情绪 → 音乐映射

```typescript
// systems/dj/mood-mapper.ts

/**
 * 将 6 维情绪向量映射为音乐特征参数
 * 参考：Russell 的情绪环状模型 + 音乐情感理论
 */
export function mapMoodToMusicFeatures(emotion: EmotionState) {
  // 效价 (Valence): joy 高 sadness 低 → 正面音乐
  const valence = emotion.joy - emotion.sadness;

  // 激活度 (Arousal): energy 高 boredom 低 → 快节奏/激烈
  const arousal = emotion.energy - emotion.boredom;

  return {
    // 节奏 (BPM)
    tempo: mapArousalToTempo(arousal),        // 60 (慢) ~ 180 (快)

    // 调式：正面→大调，负面→小调
    mode: valence > 0 ? 'major' : 'minor',

    // 能量等级
    energy: mapArousalToEnergy(arousal),       // 0~1

    // 推荐风格标签
    genreTags: deriveGenreTags(valence, arousal, emotion),

    // 声学 vs 电子倾向
    acoustic: emotion.boredom > 0.6 ? 0.7 : 0.3,
  };
}

function mapArousalToTempo(arousal: number): number {
  // 激活度 -1~1 → BPM 60~180
  const normalized = (arousal + 1) / 2; // 0~1
  return 60 + normalized * 120;
}

function deriveGenreTags(
  valence: number,
  arousal: number,
  emotion: EmotionState,
): string[] {
  // 基于情绪环状模型象限判断
  if (valence > 0.3 && arousal > 0.3)  return ['pop', 'dance', 'electronic'];
  if (valence > 0.3 && arousal <= 0.3) return ['acoustic', 'folk', 'indie'];
  if (valence <= 0 && arousal > 0.3)   return ['rock', 'metal', 'punk'];
  if (valence <= 0 && arousal <= 0.3)  return ['ambient', 'classical', 'sad'];
  return ['pop'];
}
```

---

## 十二、窗口管理

### 12.1 窗口配置

```json
// 宠物模式 (已有)
{
  "title": "aeri",
  "width": 120,
  "height": 120,
  "resizable": false,
  "transparent": true,
  "decorations": false,
  "alwaysOnTop": true
}

// DJ 模式 (新增)
{
  "title": "DJ Aeri",
  "width": 400,
  "height": 320,
  "resizable": true,
  "minWidth": 320,
  "minHeight": 280,
  "transparent": true,
  "decorations": false,
  "alwaysOnTop": true
}
```

### 12.2 模式切换时窗口动画

```typescript
// tauri/window.ts

import { getCurrentWindow, LogicalSize } from '@tauri-apps/api/window';

export async function switchWindowMode(mode: 'pet' | 'dj') {
  const win = getCurrentWindow();
  if (mode === 'dj') {
    // Tauri v2 要求 setSize 参数为 LogicalSize 实例
    await win.setSize(new LogicalSize(400, 320));
  } else {
    await win.setSize(new LogicalSize(120, 120));
  }
}
```

参考：[Tauri v2 setSize API](https://v2.tauri.app/reference/javascript/api/namespacewindow/#setsize)

---

## 十三、风险评估与缓解

| 风险 | 概率 | 影响 | 缓解措施 | 回退方案 |
|------|------|------|---------|---------|
| NeteaseCloudMusicApi 仓库已存档 | 🔴 已发生 | 高 | v1 用 MockSource；v1.1 如接入则 fork 自维护，依赖社区活跃 fork（如 `netease-cloud-music-api` 相关活跃 fork） | 永久使用 MockSource + 手动歌单 |
| QQ 音乐 API 代理不稳定 | 中 | 中 | v1 不依赖；v1.1 作为实验性可选源 | 只保留 MockSource |
| DeepSeek API 变动 | 低 | 中 | 使用 OpenAI 兼容接口，aeri chat.ts 已验证；可配置 `deepseek-v4-flash` / `deepseek-v4-pro` | 切换智谱 GLM / 阿里百炼 |
| Web Speech API 语音质量差（Linux） | 中 | 低 | v1 本机合成够用；v1.1 升级 Edge/阿里云 TTS | 纯文字显示 |
| Web Audio API 自动播放限制 | 低 | 中 | 首次交互时初始化 AudioContext；MockSource 用本地 URL 无 CORS 问题 | 显示「点我开始」引导 |
| 在线音乐 API 代理与打包 | 中 | 中 | v1 不需要代理；v1.1 开发时手动启动，打包方案届时评估 | 不做 sidecar，用户手动启动 |
| 歌词版权 | 低 | 低 | v1.2 才做，歌词仅展示不持久化 | 移除歌词功能 |
| 用户无 DeepSeek Key | 高 | 低 | useSettingsStore 提供配置入口 + 注册引导；v1.0 Mock 播放不依赖 AI 也能跑 | 基础 DJ 播放不需要 AI |
| useChatStore 硬编码 API Key 泄露 | 🔴 已存在 | 高 | **S00 分支优先修复**，迁移到 useSettingsStore | — |

---

## 十四、成功标准

### v1.0 里程碑

| 阶段 | 可验证的里程碑 |
|------|---------------|
| S00 | API Key 不再硬编码在 chat.ts 中，useSettingsStore 可读写 |
| 阶段1 | 右键宠物弹出菜单 →「切换 DJ 模式」→ 窗口变大（120×120 → 400×320），切回正常 |
| 阶段2 | DJ 模式自动加载 Mock 歌单，播放/暂停/切歌/音量滑块正常 |
| 阶段3 | 频谱随音乐跳动，灯光随节奏变化 |
| 阶段4 | DJ 模式下打开聊天，回复风格符合 DJ 人设（专业、聊音乐），与宠物模式记忆共用 |

### v1.1 里程碑

| 阶段 | 可验证的里程碑 |
|------|---------------|
| 阶段5 | 启动 NeteaseCloudMusicApi 代理后，能搜索和播放在线歌曲 |
| 阶段6 | 歌曲间有淡入淡出过渡效果 |
| 阶段7 | 歌曲切换时 Web Speech 播报「接下来是 XXX」 |

### v1.2 里程碑

| 阶段 | 可验证的里程碑 |
|------|---------------|
| 阶段8 | 说「来点开心的」→ AI 分析情绪，自动推荐歌单 |
| 阶段9 | 歌词同步显示 |
| 阶段10 | 切换 QQ 音乐源，搜索和播放正常 |

---

## 十五、前置依赖

在 DJ 开发开始前，以下 aeri 现有模块需要先到位：

| 优先级 | 前置模块 | 当前状态 | 需要程度 | 预计工作量 |
|--------|----------|---------|---------|-----------|
| 🔴 P0 | `stores/useSettingsStore.ts` | 不存在 | **安全修复**：useChatStore 硬编码了 API Key，必须迁移 | 0.5 天 |
| 🔴 P0 | `systems/ai/chat.ts` 去除硬编码 | 已有但 apiKey 写死在代码里 | **安全修复**：改为从 useSettingsStore.getState() 读取 | 0.5 天 |
| 🔴 P0 | `src/tauri/window.ts` | 不存在 | DJ 模式窗口管理基础依赖 | 0.5 天 |
| 🟡 P1 | `systems/emotion/` | 不存在（v0.2 规划） | v1.2 AI 编排需要情绪数据；v1.0 可跳过 | 2~3 天 |

---

## 十六、文件清单（按分支）

### v1.0 分支

#### S00: `fix/security/remove-hardcoded-key`
| 文件 | 操作 | 说明 |
|------|------|------|
| `src/stores/useSettingsStore.ts` | 新建 | API Key、DeepSeek URL、音乐源选择 |
| `src/systems/ai/chat.ts` | 修改 | 去除硬编码，从 useSettingsStore 读取 config |

#### 分支 01: `feature/dj/mode-switch`
| 文件 | 操作 |
|------|------|
| `src/stores/useDJStore.ts` | 新建 |
| `src/systems/dj/persona.ts` | 新建 |
| `src/systems/dj/types.ts` | 新建 |
| `src/tauri/window.ts` | 新建 |

#### 分支 02: `feature/music/mock-source`
| 文件 | 操作 |
|------|------|
| `src/systems/music/types.ts` | 新建（MusicSource 接口 + Track 等） |
| `src/systems/music/mock-source.ts` | 新建（MockSource 内嵌 Demo 歌单） |

#### 分支 03: `feature/music/player`
| 文件 | 操作 |
|------|------|
| `src/systems/music/player.ts` | 新建（双 audio 架构 PlayerEngine） |
| `src/stores/useMusicStore.ts` | 新建 |
| `src/components/dj-controls/PlayControls.tsx` | 新建 |
| `src/components/dj-controls/NowPlaying.tsx` | 新建 |

#### 分支 04: `feature/dj/chat`
| 文件 | 操作 |
|------|------|
| `src/systems/ai/chat.ts` | 修改（baseUrl → DeepSeek，支持 DJ persona prompt 切换） |

#### 分支 05: `feature/dj/booth-ui`
| 文件 | 操作 |
|------|------|
| `src/components/dj-booth/DJBooth.tsx` | 新建（DJ 台主容器 + 布局） |
| `src/components/dj-booth/DJCharCanvas.tsx` | 新建（DJ 角色渲染） |
| `src/components/dj-booth/Visualizer.tsx` | 新建（频谱可视化） |
| `src/components/dj-booth/StageLights.tsx` | 新建（灯光效果） |
| `src/App.tsx` | 修改（按 mode 渲染不同组件树） |

### v1.1 分支

#### 分支 06: `feature/music/mixing`
| 文件 | 操作 |
|------|------|
| `src/systems/music/mixing.ts` | 新建（MixEngine 交叉淡入淡出） |

#### 分支 07: `feature/music/netease-source`
| 文件 | 操作 |
|------|------|
| `src/systems/music/netease.ts` | 新建（NeteaseSource implements MusicSource） |

#### 分支 08: `feature/music/qq-source`
| 文件 | 操作 |
|------|------|
| `src/systems/music/qq.ts` | 新建（QQSource implements MusicSource） |

#### 分支 09: `feature/dj/tts`
| 文件 | 操作 |
|------|------|
| `src/systems/tts/types.ts` | 新建 |
| `src/systems/tts/engine.ts` | 新建（v1 speechSynthesis；v1.1 Edge/阿里云） |

### v1.2 分支

#### 分支 10: `feature/emotion/core`
| 文件 | 操作 |
|------|------|
| `src/systems/emotion/types.ts` | 新建 |
| `src/systems/emotion/engine.ts` | 新建 |

#### 分支 11: `feature/dj/curation`
| 文件 | 操作 |
|------|------|
| `src/systems/dj/engine.ts` | 新建（TrackScheduler 编排引擎） |
| `src/systems/dj/mood-mapper.ts` | 新建（情绪→音乐特征映射） |

#### 分支 12: `feature/dj/lyrics`
| 文件 | 操作 |
|------|------|
| `src/components/dj-booth/LyricsDisplay.tsx` | 新建（LRC 解析 + 同步显示） |

#### 分支 13: `feature/dj/request`
| 文件 | 操作 |
|------|------|
| `src/components/dj-controls/RequestInput.tsx` | 新建（自然语言点歌/指令） |
| `src/components/dj-controls/SourceSwitcher.tsx` | 新建（网易云 ↔ QQ 切换） |

**统计：v1.0 新建 17 个文件，修改 2 个文件；v1.1 新建 6 个；v1.2 新建 6 个。**

---

> **下一步：** 本文档审核通过后，使用 writing-plans 技能生成详细的逐任务实现计划。
