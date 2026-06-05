# Aeri AI DJ 电台 — 完整设计方案

> 版本：v1.0
> 日期：2026-06-05
> 状态：待审核
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

| 编号 | 特性 | 描述 |
|------|------|------|
| F01 | 模式切换 | 宠物模式 ↔ DJ 模式，窗口自适应变换 |
| F02 | 音乐播放引擎 | 播放/暂停/切歌/音量/进度控制 |
| F03 | 在线音乐源 | 网易云音乐 + QQ 音乐，用户可切换 |
| F04 | DJ 混音过渡 | 歌曲间自动淡入淡出、背景音乐层 |
| F05 | AI 选曲编排 | 基于情绪/时间/场景/用户偏好自动编排歌单 |
| F06 | TTS 语音播报 | 歌曲切换时播报曲目信息 + DJ 闲聊 |
| F07 | DJ 可视化 | DJ 台动画、音波频谱、灯光效果 |
| F08 | 歌词同步 | 当前播放歌曲的同步歌词展示 |
| F09 | 用户指令 | 点歌、跳过、「来点开心的」等自然语言指令 |
| F10 | DJ 人格 | 开放式聊天，DJ 人设鲜明，与宠物模式共用记忆 |
| F11 | 情绪联动 | 读取 aeri 情绪系统数据，驱动选曲编排 |
| F12 | 平台切换 | 网易云 / QQ 音乐无缝切换 |

### 1.3 非目标（v1 不做）

- 不生成 AI 原创音乐（Suno/Udio）
- 不做社交/分享/多人联听
- 不做移动端适配
- 不做播客/新闻模块
- 不做直播推流

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
| LLM | DeepSeek API (deepseek-chat) | AI 对话、编排决策 | ✅ 国内直连 |
| 音频播放 | Web Audio API (AudioContext) | 音频播放、混音、频谱分析 | ✅ 浏览器原生 |
| TTS | Edge TTS (Web Speech API / ms-tts) | 语音播报 | ✅ 系统自带 |
| 音乐 API 代理 | [NeteaseCloudMusicApi](https://github.com/Binaryify/NeteaseCloudMusicApi) | 网易云音乐 | ✅ 7年维护 30k+ star |
| 音乐 API 代理 | python-qqmusic-api / QQMusicApi | QQ 音乐 | ✅ 备选方案 |

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
│  │ DeepSeek API │  │ 音乐 API 代理 │  │  Edge TTS    │              │
│  │ (对话/编排)   │  │ localhost:    │  │ (系统内置)    │              │
│  │              │  │  3000/3300    │  │              │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                    │
│  ┌─────────────────── Rust 后端 ──────────────────────────────────┐│
│  │  main.rs / lib.rs / commands/dj.rs                            ││
│  │  └── 音乐 API 代理进程生命周期管理 (sidecar)                     ││
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
│   │   │   ├── player.ts               #   PlayerEngine 播放器状态机
│   │   │   ├── mixing.ts               #   MixEngine 混音过渡
│   │   │   ├── netease.ts              #   NeteaseSource implements MusicSource
│   │   │   └── qq.ts                   #   QQSource implements MusicSource
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

| 序号 | 分支名 | 功能摘要 | 依赖 | 估计文件数 |
|------|--------|---------|------|-----------|
| 00 | `feature/emotion/core` | 情绪系统（aeri v0.2 前置） | 无 | 2 |
| 01 | `feature/dj/mode-switch` | 模式切换 + 窗口变换 + useDJStore | 无 | 4 |
| 02 | `feature/music/source` | `MusicSource` 接口 + NeteaseCloudMusicApi 代理封装 | 无 | 3 |
| 03 | `feature/music/player` | PlayerEngine + 播放控制 + useMusicStore | #02 | 4 |
| 04 | `feature/dj/chat` | DJ 人设对话（复用 systems/ai/chat.ts） | #01 | 2 |
| 05 | `feature/music/mixing` | MixEngine 淡入淡出 + 过渡 + 背景乐 | #03 | 2 |
| 06 | `feature/dj/tts` | Edge TTS 播报引擎 | #01 | 3 |
| 07 | `feature/dj/curation` | AI 编排 + 情绪-音乐映射 | #03 + #04 + #00 | 3 |
| 08 | `feature/dj/booth-ui` | DJBooth + Visualizer + StageLights | #03 + #01 | 5 |
| 09 | `feature/music/qq-source` | QQ 音乐源实现 | #02 | 1 |
| 10 | `feature/dj/lyrics` | 歌词同步展示 | #03 | 2 |
| 11 | `feature/dj/request` | 用户自然语言点歌/指令 | #03 + #04 | 1 |

### 5.4 Commit 规范

```
feat(dj): add mode switch between pet and DJ modes
feat(music): add NeteaseCloudMusicApi proxy integration
feat(music): add PlayerEngine with play/pause/skip
feat(dj): add DJ persona chat via DeepSeek
feat(music): add crossfade mixing engine
feat(dj): add Edge TTS announcement engine
feat(dj): add mood-based music curation
feat(dj): add audio visualizer and stage lights
feat(music): add QQ music source implementation
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
  source: 'netease' | 'qq';
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
  readonly source: 'netease' | 'qq';

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

export interface TTSConfig {
  provider: 'edge' | 'alibaba';
  voice: string;             // 语音包 ID
  rate: number;              // 语速 0.5~2.0
  pitch: number;             // 音调 0.5~2.0
  volume: number;            // 0~1
}

export interface TTSVoice {
  id: string;
  name: string;
  lang: string;
  gender: 'male' | 'female';
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
  ttsConfig: TTSConfig;
  djPersonaCustomPrompt: string;
}

interface SettingsActions {
  setDeepseekKey: (key: string) => void;
  setMusicSource: (source: 'netease' | 'qq') => void;
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

```
用户双击宠物 / 点击「DJ 模式」按钮
    │
    ▼
components/pet/DragLayer.tsx (onDoubleClick)
    │
    ▼
useDJStore.switchMode('dj')
    ├─ 1. set({ mode: 'dj' })
    ├─ 2. 调用 tauri/window.ts → resize(400, 320)
    ├─ 3. 调用 systems/dj/persona.ts → load DJ system prompt
    ├─ 4. 调用 systems/ai/chat.ts → 更新 SYSTEM_PROMPT
    └─ 5. 初始化 useMusicStore (如果没有)
    │
    ▼
App.tsx 重新渲染
    ├─ mode === 'pet'  →  渲染 pet/ + overlays/
    └─ mode === 'dj'   →  渲染 dj-booth/ + dj-controls/ + overlays/
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

### 11.1 Web Audio API 播放器

```
AudioContext
├─ SourceNode (当前歌曲)
│   └─ GainNode (音量 + 淡入淡出)
│       └─ AnalyserNode (频谱数据)
│           └─ Destination
│
├─ SourceNode (背景音乐层)
│   └─ GainNode (bgVolume 独立控制)
│       └─ Destination
│
└─ AnalyserNode → frequency data → Visualizer 组件
```

```typescript
// systems/music/player.ts 核心结构

export class PlayerEngine {
  private ctx: AudioContext;
  private currentSource: AudioBufferSourceNode | null;
  private currentGain: GainNode;
  private bgGain: GainNode;
  private analyser: AnalyserNode;
  private state: PlayerStatus;

  constructor() {
    this.ctx = new AudioContext();
    // 构建音频节点链
  }

  async load(url: string): Promise<void> { /* fetch → decode → buffer */ }
  play(): void { /* sourceNode.start() */ }
  pause(): void { /* ctx.suspend() */ }
  setVolume(v: number): void { /* gainNode.gain.rampToValueAtTime() */ }
  getFrequencyData(): Uint8Array { /* analyser.getByteFrequencyData() */ }
}
```

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

import { getCurrentWindow } from '@tauri-apps/api/window';

export async function switchWindowMode(mode: 'pet' | 'dj') {
  const win = getCurrentWindow();
  if (mode === 'dj') {
    await win.setSize({ width: 400, height: 320 });
  } else {
    await win.setSize({ width: 120, height: 120 });
  }
}
```

---

## 十三、风险评估与缓解

| 风险 | 概率 | 影响 | 缓解措施 | 回退方案 |
|------|------|------|---------|---------|
| NeteaseCloudMusicApi 暂停维护 | 低 | 高 | Fork 到项目下自维护 | 切换到 QQ 音乐源 |
| QQ 音乐 API 代理不稳定 | 中 | 中 | 先上网易云，QQ 后续跟进 | 只保留网易云 |
| DeepSeek API 变动 | 低 | 中 | 使用 OpenAI 兼容接口，aeri chat.ts 已验证 | 切换智谱 GLM / 阿里百炼 |
| Edge TTS 不可用（Linux） | 低 | 低 | 检测平台，Linux 回退到阿里云 TTS | 纯文字显示 |
| Web Audio API 自动播放限制 | 低 | 中 | 首次交互时初始化 AudioContext | 显示「点我开始」引导 |
| 音乐 API 代理与 aeri sidecar 打包 | 中 | 中 | 先开发时手动启动代理，打包后续处理 | 不做 sidecar，用户手动启动 |
| 歌词版权 | 低 | 低 | 歌词仅展示，不持久化 | 移除歌词功能 |
| 用户无 DeepSeek Key | 高 | 低 | 提供注册引导链接 | 基础播放模式不需要 AI |

---

## 十四、成功标准

| 阶段 | 可验证的里程碑 |
|------|---------------|
| 阶段1 | 双击宠物切换 DJ 模式，窗口变大，能看到 DJ 台 UI，切回宠物模式正常 |
| 阶段2 | 能在 DJ 模式下播放网易云音乐，切歌、调音量正常 |
| 阶段3 | 歌曲间有淡入淡出过渡效果，不突兀 |
| 阶段4 | DJ 模式下和宠物对话，回复风格符合 DJ 人设（专业、聊音乐） |
| 阶段5 | TTS 在歌曲间播报「接下来是 XXX」，效果自然 |
| 阶段6 | 点击「根据心情推荐」，AI 生成歌单并自动播放 |
| 阶段7 | 频谱随音乐跳动，灯光随节奏变化 |
| 阶段8 | 歌词同步显示 |
| 阶段9 | 打字「来点开心的」→ 自动推荐欢快歌单并播放 |
| 阶段10 | 切换 QQ 音乐源，搜索和播放正常 |

---

## 十五、前置依赖

在 DJ 开发开始前，以下 aeri 现有模块需要先到位：

| 前置模块 | 当前状态 | 需要程度 | 预计工作量 |
|----------|---------|---------|-----------|
| `systems/emotion/` | 不存在（v0.2 规划） | 🔴 必需（DJ 编排需要情绪数据） | 2~3 天 |
| `stores/useSettingsStore.ts` | 不存在（规划中） | 🔴 必需（存 API Key、平台选择） | 0.5 天 |
| `src/tauri/` 目录 | 不存在（规划中） | 🟡 强烈建议（窗口管理、存储） | 1 天 |
| `systems/ai/chat.ts` 改 DeepSeek | 已有但硬编码 OpenAI | 🟡 强烈建议 | 10 分钟 |

---

## 十六、文件清单（按分支）

### 分支 00: `feature/emotion/core`
| 文件 | 操作 |
|------|------|
| `src/systems/emotion/types.ts` | 新建 |
| `src/systems/emotion/engine.ts` | 新建 |

### 分支 01: `feature/dj/mode-switch`
| 文件 | 操作 |
|------|------|
| `src/stores/useDJStore.ts` | 新建 |
| `src/stores/useSettingsStore.ts` | 新建 |
| `src/systems/dj/persona.ts` | 新建 |
| `src/tauri/window.ts` | 新建（或更新） |

### 分支 02: `feature/music/source`
| 文件 | 操作 |
|------|------|
| `src/systems/music/types.ts` | 新建 |
| `src/systems/music/netease.ts` | 新建 |
| `src-tauri/src/commands/dj.rs` | 新建（代理进程管理） |

### 分支 03: `feature/music/player`
| 文件 | 操作 |
|------|------|
| `src/systems/music/player.ts` | 新建 |
| `src/stores/useMusicStore.ts` | 新建 |
| `src/components/dj-controls/PlayControls.tsx` | 新建 |
| `src/components/dj-controls/NowPlaying.tsx` | 新建 |

### 分支 04: `feature/dj/chat`
| 文件 | 操作 |
|------|------|
| `src/systems/ai/chat.ts` | 修改（baseUrl → DeepSeek） |
| `src/systems/dj/types.ts` | 新建 |

### 分支 05: `feature/music/mixing`
| 文件 | 操作 |
|------|------|
| `src/systems/music/mixing.ts` | 新建 |

### 分支 06: `feature/dj/tts`
| 文件 | 操作 |
|------|------|
| `src/systems/tts/types.ts` | 新建 |
| `src/systems/tts/engine.ts` | 新建 |

### 分支 07: `feature/dj/curation`
| 文件 | 操作 |
|------|------|
| `src/systems/dj/engine.ts` | 新建 |
| `src/systems/dj/mood-mapper.ts` | 新建 |

### 分支 08: `feature/dj/booth-ui`
| 文件 | 操作 |
|------|------|
| `src/components/dj-booth/DJBooth.tsx` | 新建 |
| `src/components/dj-booth/DJCharCanvas.tsx` | 新建 |
| `src/components/dj-booth/Visualizer.tsx` | 新建 |
| `src/components/dj-booth/StageLights.tsx` | 新建 |

### 分支 09: `feature/music/qq-source`
| 文件 | 操作 |
|------|------|
| `src/systems/music/qq.ts` | 新建 |

### 分支 10: `feature/dj/lyrics`
| 文件 | 操作 |
|------|------|
| `src/components/dj-booth/LyricsDisplay.tsx` | 新建 |

### 分支 11: `feature/dj/request`
| 文件 | 操作 |
|------|------|
| `src/components/dj-controls/RequestInput.tsx` | 新建 |
| `src/components/dj-controls/SourceSwitcher.tsx` | 新建 |

**统计：新建约 28 个文件，修改约 2 个文件。**

---

> **下一步：** 本文档审核通过后，使用 writing-plans 技能生成详细的逐任务实现计划。
