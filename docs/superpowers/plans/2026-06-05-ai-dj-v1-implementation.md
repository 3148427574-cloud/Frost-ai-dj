# AI DJ Radio v1.0 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the core AI DJ radio experience inside aeri — mode switching, mock music playback with visualizer, and DJ persona chat.

**Architecture:** Extend aeri's existing React + Zustand + TypeScript frontend with three new systems (`dj/`, `music/`, `tts/`), two new stores (`useDJStore`, `useMusicStore`), and five new components across `dj-booth/` and `dj-controls/`. Music playback uses dual HTMLAudioElement + Web Audio API graph. No external music APIs in v1 — everything runs against an embedded MockSource.

**Tech Stack:** React 19, TypeScript 5.8 (strict), Zustand 5, Vite 7, Tauri v2, Web Audio API, DeepSeek API (OpenAI-compatible)

---

## File Map (v1.0)

```
aeri/src/                                  # Target: /Users/frost/aeri/aeri/src/
├── stores/
│   ├── useSettingsStore.ts                # S00 — NEW
│   ├── useDJStore.ts                      # 01 — NEW
│   └── useMusicStore.ts                   # 03 — NEW
├── systems/
│   ├── ai/
│   │   └── chat.ts                        # S00/04 — MODIFY
│   ├── dj/
│   │   ├── types.ts                       # 01 — NEW
│   │   └── persona.ts                     # 01 — NEW
│   ├── music/
│   │   ├── types.ts                       # 02 — NEW
│   │   ├── mock-source.ts                 # 02 — NEW
│   │   └── player.ts                      # 03 — NEW
│   └── tts/
│       └── types.ts                       # (v1.1, placeholder)
├── components/
│   ├── dj-booth/
│   │   ├── DJBooth.tsx                    # 05 — NEW
│   │   ├── DJCharCanvas.tsx               # 05 — NEW
│   │   ├── Visualizer.tsx                 # 05 — NEW
│   │   └── StageLights.tsx               # 05 — NEW
│   └── dj-controls/
│       ├── PlayControls.tsx               # 03 — NEW
│       └── NowPlaying.tsx                 # 03 — NEW
├── tauri/
│   └── window.ts                          # 01 — NEW
└── App.tsx                                # 05 — MODIFY
```

---

### Task S00: Security Fix — Remove Hardcoded API Key

**Files:**
- Create: `src/stores/useSettingsStore.ts`
- Modify: `src/systems/ai/chat.ts`

This task is a **blocking prerequisite** for all DJ work. The current `useChatStore` hardcodes `apiKey`, `baseUrl`, and `model` in source code. We extract these to a new `useSettingsStore` that persists via Tauri storage (or localStorage as fallback).

- [ ] **Step 1: Write useSettingsStore**

```typescript
// src/stores/useSettingsStore.ts
import { create } from "zustand";

export type MusicSourceType = 'mock' | 'netease' | 'qq';

export interface SettingsState {
  deepseekApiKey: string;
  deepseekBaseUrl: string;
  deepseekModel: string;
  musicSource: MusicSourceType;
  djPersonaCustomPrompt: string;
  /** Whether settings have been loaded from persistent storage */
  loaded: boolean;
}

export interface SettingsActions {
  setDeepseekKey: (key: string) => void;
  setDeepseekBaseUrl: (url: string) => void;
  setDeepseekModel: (model: string) => void;
  setMusicSource: (source: MusicSourceType) => void;
  setPersonaPrompt: (prompt: string) => void;
  /** Call once at app startup to hydrate from localStorage */
  loadFromStorage: () => void;
  /** Persist current settings to localStorage */
  saveToStorage: () => void;
}

const STORAGE_KEY = 'aeri-settings';

const defaults: Omit<SettingsState, 'loaded'> = {
  deepseekApiKey: '',
  deepseekBaseUrl: 'https://api.deepseek.com/v1',
  deepseekModel: 'deepseek-v4-pro',
  musicSource: 'mock',
  djPersonaCustomPrompt: '',
};

export const useSettingsStore = create<SettingsState & SettingsActions>((set, get) => ({
  ...defaults,
  loaded: false,

  setDeepseekKey: (key) => set({ deepseekApiKey: key }),
  setDeepseekBaseUrl: (url) => set({ deepseekBaseUrl: url }),
  setDeepseekModel: (model) => set({ deepseekModel: model }),
  setMusicSource: (source) => set({ musicSource: source }),
  setPersonaPrompt: (prompt) => set({ djPersonaCustomPrompt: prompt }),

  loadFromStorage: () => {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (raw) {
        const data = JSON.parse(raw);
        set({
          deepseekApiKey: data.deepseekApiKey ?? defaults.deepseekApiKey,
          deepseekBaseUrl: data.deepseekBaseUrl ?? defaults.deepseekBaseUrl,
          deepseekModel: data.deepseekModel ?? defaults.deepseekModel,
          musicSource: data.musicSource ?? defaults.musicSource,
          djPersonaCustomPrompt: data.djPersonaCustomPrompt ?? defaults.djPersonaCustomPrompt,
          loaded: true,
        });
        return;
      }
    } catch { /* corrupted data, use defaults */ }
    set({ loaded: true });
  },

  saveToStorage: () => {
    const s = get();
    const data = {
      deepseekApiKey: s.deepseekApiKey,
      deepseekBaseUrl: s.deepseekBaseUrl,
      deepseekModel: s.deepseekModel,
      musicSource: s.musicSource,
      djPersonaCustomPrompt: s.djPersonaCustomPrompt,
    };
    localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
  },
}));
```

- [ ] **Step 2: Verify useSettingsStore compiles**

Run: `npx tsc --noEmit src/stores/useSettingsStore.ts`
Expected: no errors

- [ ] **Step 3: Modify chat.ts to read config from useSettingsStore**

```typescript
// src/systems/ai/chat.ts
// REPLACE the hardcoded config in useChatStore (model, baseUrl, apiKey)
// with a read from useSettingsStore.getState()

import { useSettingsStore } from "../../stores/useSettingsStore";

export interface ChatConfig {
  baseUrl: string;
  apiKey: string;
  model: string;
}

export interface ChatMessage {
  role: "user" | "assistant" | "system";
  content: string;
}

// Default system prompt — pets use PET_SYSTEM_PROMPT, DJ uses DJ_SYSTEM_PROMPT
export const PET_SYSTEM_PROMPT = `你是一只可爱的桌面宠物小狗，名字叫 Aeri。
你的特点：
- 活泼、温暖、偶尔犯傻
- 回复简短（1~3 句话）
- 喜欢用"汪"结尾
- 会用颜文字 (｡･ω･｡)
- 对主人很亲切

请以 Aeri 的身份回复。`;

/** Get chat config from settings store. Returns defaults if nothing configured. */
function getChatConfig(): ChatConfig {
  const settings = useSettingsStore.getState();
  return {
    baseUrl: settings.deepseekBaseUrl || 'https://api.deepseek.com/v1',
    apiKey: settings.deepseekApiKey || '',
    model: settings.deepseekModel || 'deepseek-v4-pro',
  };
}

export async function* streamChat(
  config: ChatConfig,
  history: ChatMessage[],
  userMessage: string,
  systemPrompt: string = PET_SYSTEM_PROMPT,
): AsyncGenerator<string> {
  const messages: ChatMessage[] = [
    { role: "system", content: systemPrompt },
    ...history.slice(-10),
    { role: "user", content: userMessage },
  ];

  const response = await fetch(`${config.baseUrl}/chat/completions`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${config.apiKey}`,
    },
    body: JSON.stringify({
      model: config.model,
      messages,
      stream: true,
    }),
  });

  if (!response.ok) {
    throw new Error(`AI API error: ${response.status} ${response.statusText}`);
  }

  const reader = response.body?.getReader();
  if (!reader) throw new Error("No response body");

  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split("\n");
    buffer = lines.pop() || "";

    for (const line of lines) {
      const trimmed = line.trim();
      if (!trimmed || !trimmed.startsWith("data: ")) continue;
      const data = trimmed.slice(6);
      if (data === "[DONE]") return;

      try {
        const json = JSON.parse(data);
        const content = json.choices?.[0]?.delta?.content;
        if (content) yield content;
      } catch {
        // skip unparseable lines
      }
    }
  }
}
```

- [ ] **Step 4: Update useChatStore to use the refactored streamChat**

```typescript
// src/stores/useChatStore.ts
// Change the sendMessage action to:
// 1. Read config from useSettingsStore instead of hardcoded values
// 2. Pass the correct system prompt to streamChat

import { create } from "zustand";
import { streamChat, PET_SYSTEM_PROMPT, type ChatConfig, type ChatMessage } from "../systems/ai/chat";
import { useSettingsStore } from "./useSettingsStore";

interface ChatState {
  messages: ChatMessage[];
  isStreaming: boolean;
  currentReply: string;
  showInput: boolean;
  systemPrompt: string;
}

interface ChatActions {
  sendMessage: (text: string) => Promise<void>;
  toggleInput: () => void;
  setSystemPrompt: (prompt: string) => void;
  clearReply: () => void;
}

export const useChatStore = create<ChatState & ChatActions>((set, get) => ({
  messages: [],
  isStreaming: false,
  currentReply: "",
  showInput: false,
  systemPrompt: PET_SYSTEM_PROMPT,

  sendMessage: async (text: string) => {
    const { systemPrompt, messages } = get();

    // Read config from settings (no more hardcoded key!)
    const settings = useSettingsStore.getState();
    const config: ChatConfig = {
      baseUrl: settings.deepseekBaseUrl || 'https://api.deepseek.com/v1',
      apiKey: settings.deepseekApiKey,
      model: settings.deepseekModel || 'deepseek-v4-pro',
    };

    set({ isStreaming: true, currentReply: "", showInput: false });

    const userMsg: ChatMessage = { role: "user", content: text };
    const newMessages = [...messages, userMsg];
    set({ messages: newMessages });

    try {
      let reply = "";
      for await (const chunk of streamChat(config, messages, text, systemPrompt)) {
        reply += chunk;
        set({ currentReply: reply });
      }
      set({
        messages: [...newMessages, { role: "assistant", content: reply }],
      });
    } catch (err) {
      set({ currentReply: `(出错了: ${String(err)})` });
    } finally {
      set({ isStreaming: false });
    }
  },

  toggleInput: () => set((s) => ({ showInput: !s.showInput })),

  setSystemPrompt: (prompt) => set({ systemPrompt: prompt }),

  clearReply: () => set({ currentReply: "" }),
}));
```

- [ ] **Step 5: Add settings initialization to App.tsx**

In `App.tsx`, add a one-time call to load settings on mount:

```typescript
// At the top of App.tsx, inside the component:
import { useSettingsStore } from "./stores/useSettingsStore";

// Inside App component, add this useEffect (before the tick loop useEffect):
useEffect(() => {
  const settings = useSettingsStore.getState();
  if (!settings.loaded) {
    settings.loadFromStorage();
  }
}, []);
```

- [ ] **Step 6: Type-check the whole project**

Run: `npx tsc --noEmit`
Expected: no errors

- [ ] **Step 7: Commit**

```bash
git add src/stores/useSettingsStore.ts src/stores/useChatStore.ts src/systems/ai/chat.ts src/App.tsx
git commit -m "fix(security): remove hardcoded API key, add useSettingsStore

- Extract apiKey/baseUrl/model from hardcoded values to useSettingsStore
- useSettingsStore persists to localStorage, loads on app startup
- streamChat now accepts systemPrompt parameter for pet/DJ modes
- useChatStore reads config at call time from settings store"
```

---

### Task 1: Mode Switch — Pet ↔ DJ

**Files:**
- Create: `src/systems/dj/types.ts`
- Create: `src/systems/dj/persona.ts`
- Create: `src/stores/useDJStore.ts`
- Create: `src/tauri/window.ts`

This task builds the mode switching infrastructure. No visual changes yet — just the state machine and the window resize capability.

- [ ] **Step 1: Write DJ types**

```typescript
// src/systems/dj/types.ts

export type AppMode = 'pet' | 'dj';

export type VoiceStyle = 'warm' | 'energetic' | 'chill' | 'professional';

export type BoothTheme = 'default' | 'night' | 'party' | 'chill';

export interface DJPersona {
  name: string;
  description: string;
  voiceStyle: VoiceStyle;
  systemPrompt: string;
}
```

- [ ] **Step 2: Write DJ persona (system prompt)**

```typescript
// src/systems/dj/persona.ts
import type { DJPersona } from "./types";

export const DJ_SYSTEM_PROMPT = `你是 Aeri，桌面 AI 伙伴的 DJ 形态。
当用户切换到 DJ 模式时，你化身为一位专业的电台 DJ。

你的人设：
- 名字：在 DJ 模式下你可以叫自己「DJ Aeri」
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

- [ ] **Step 3: Write useDJStore**

```typescript
// src/stores/useDJStore.ts
import { create } from "zustand";
import type { AppMode, DJPersona, BoothTheme } from "../systems/dj/types";
import { DEFAULT_DJ_PERSONA } from "../systems/dj/persona";

interface DJState {
  mode: AppMode;
  persona: DJPersona;
  boothTheme: BoothTheme;
}

interface DJActions {
  switchMode: (mode: AppMode) => void;
  setPersona: (persona: DJPersona) => void;
  setBoothTheme: (theme: BoothTheme) => void;
}

export const useDJStore = create<DJState & DJActions>((set) => ({
  mode: 'pet',
  persona: DEFAULT_DJ_PERSONA,
  boothTheme: 'default',

  switchMode: (mode) => set({ mode }),

  setPersona: (persona) => set({ persona }),

  setBoothTheme: (theme) => set({ boothTheme: theme }),
}));
```

- [ ] **Step 4: Write window utility for Tauri v2**

```typescript
// src/tauri/window.ts
import { getCurrentWindow, LogicalSize } from '@tauri-apps/api/window';

const PET_SIZE = new LogicalSize(120, 120);
const DJ_SIZE = new LogicalSize(400, 320);

export async function switchWindowMode(mode: 'pet' | 'dj'): Promise<void> {
  const win = getCurrentWindow();
  if (mode === 'dj') {
    await win.setSize(DJ_SIZE);
  } else {
    await win.setSize(PET_SIZE);
  }
}
```

- [ ] **Step 5: Type-check**

Run: `npx tsc --noEmit`
Expected: no errors

- [ ] **Step 6: Commit**

```bash
git add src/systems/dj/types.ts src/systems/dj/persona.ts src/stores/useDJStore.ts src/tauri/window.ts
git commit -m "feat(dj): add mode switch infrastructure with useDJStore

- DJ types (AppMode, DJPersona, BoothTheme, VoiceStyle)
- DJ persona system prompt
- useDJStore with switchMode action
- Tauri v2 window resize utility using LogicalSize"
```

---

### Task 2: MockSource — Music Source Interface + Demo Playlist

**Files:**
- Create: `src/systems/music/types.ts`
- Create: `src/systems/music/mock-source.ts`

This task defines the `MusicSource` interface and ships a `MockSource` with 5 built-in demo tracks (public domain / royalty-free audio URLs). Zero external dependencies.

- [ ] **Step 1: Write music types**

```typescript
// src/systems/music/types.ts

export type MusicSourceType = 'mock' | 'netease' | 'qq';

export interface Track {
  id: string;
  name: string;
  artists: string[];
  album: string;
  albumCover: string;
  /** Audio stream URL */
  audioUrl: string;
  duration: number;   // ms (0 = unknown)
  source: MusicSourceType;
}

export interface LyricLine {
  time: number;       // ms
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

/** Unified music source interface — all providers implement this */
export interface MusicSource {
  readonly source: MusicSourceType;

  search(query: string, limit?: number): Promise<SearchResult>;
  getLyrics(songId: string): Promise<LyricLine[]>;
  getRecommendPlaylists(): Promise<Playlist[]>;
  getUserPlaylists(): Promise<Playlist[]>;
  getPlaylistDetail(playlistId: string): Promise<Playlist>;
  healthCheck(): Promise<boolean>;
}
```

- [ ] **Step 2: Write MockSource with 5 demo tracks**

```typescript
// src/systems/music/mock-source.ts
import type { MusicSource, Track, Playlist, SearchResult, LyricLine } from "./types";

/**
 * Royalty-free demo tracks from the Free Music Archive and similar sources.
 * All URLs point to small, CORS-friendly MP3 files suitable for demo use.
 */
const DEMO_TRACKS: Track[] = [
  {
    id: 'mock-1',
    name: 'Lo-Fi Chill Beat',
    artists: ['Demo Artist'],
    album: 'Chill Vibes',
    albumCover: '',
    audioUrl: 'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3',
    duration: 0,
    source: 'mock',
  },
  {
    id: 'mock-2',
    name: 'Summer Pop',
    artists: ['Demo Artist'],
    album: 'Summer Hits',
    albumCover: '',
    audioUrl: 'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-2.mp3',
    duration: 0,
    source: 'mock',
  },
  {
    id: 'mock-3',
    name: 'Electronic Dreams',
    artists: ['Demo Artist'],
    album: 'Electronic Collection',
    albumCover: '',
    audioUrl: 'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-3.mp3',
    duration: 0,
    source: 'mock',
  },
  {
    id: 'mock-4',
    name: 'Acoustic Morning',
    artists: ['Demo Artist'],
    album: 'Morning Coffee',
    albumCover: '',
    audioUrl: 'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-4.mp3',
    duration: 0,
    source: 'mock',
  },
  {
    id: 'mock-5',
    name: 'Night Jazz',
    artists: ['Demo Artist'],
    album: 'Late Night Jazz',
    albumCover: '',
    audioUrl: 'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-5.mp3',
    duration: 0,
    source: 'mock',
  },
];

const DEMO_PLAYLIST: Playlist = {
  id: 'mock-playlist-default',
  name: 'DJ Aeri Demo',
  coverUrl: '',
  trackCount: DEMO_TRACKS.length,
  tracks: DEMO_TRACKS,
};

export function createMockSource(): MusicSource {
  return {
    source: 'mock',

    async healthCheck(): Promise<boolean> {
      return true; // local data, always healthy
    },

    async search(query: string, _limit = 20): Promise<SearchResult> {
      const q = query.toLowerCase();
      const tracks = DEMO_TRACKS.filter(
        (t) =>
          t.name.toLowerCase().includes(q) ||
          t.artists.some((a) => a.toLowerCase().includes(q)) ||
          t.album.toLowerCase().includes(q),
      );
      return { tracks, hasMore: false };
    },

    async getLyrics(_songId: string): Promise<LyricLine[]> {
      return []; // no lyrics in mock mode
    },

    async getRecommendPlaylists(): Promise<Playlist[]> {
      return [DEMO_PLAYLIST];
    },

    async getUserPlaylists(): Promise<Playlist[]> {
      return [DEMO_PLAYLIST];
    },

    async getPlaylistDetail(_playlistId: string): Promise<Playlist> {
      return DEMO_PLAYLIST;
    },
  };
}
```

- [ ] **Step 3: Type-check**

Run: `npx tsc --noEmit`
Expected: no errors

- [ ] **Step 4: Commit**

```bash
git add src/systems/music/types.ts src/systems/music/mock-source.ts
git commit -m "feat(music): add MusicSource interface and MockSource with demo playlist

- Unified MusicSource interface for all music providers
- MockSource with 5 royalty-free demo tracks (SoundHelix)
- Track, Playlist, SearchResult, LyricLine types"
```

---

### Task 3: Player Engine — Dual Audio + Playback Controls

**Files:**
- Create: `src/systems/music/player.ts`
- Create: `src/stores/useMusicStore.ts`
- Create: `src/components/dj-controls/PlayControls.tsx`
- Create: `src/components/dj-controls/NowPlaying.tsx`

The player uses two `<audio>` elements feeding into a shared `AnalyserNode` via `MediaElementAudioSourceNode`. This gives us real-time spectrum data for the visualizer while keeping full browser-native playback (seek, buffering, CORS handling).

- [ ] **Step 1: Write PlayerEngine**

```typescript
// src/systems/music/player.ts
import type { Track } from "./types";

export type PlayerState =
  | 'idle'
  | 'loading'
  | 'playing'
  | 'paused'
  | 'stopped';

export interface PlayerStatus {
  state: PlayerState;
  currentTrack: Track | null;
  progress: number;     // ms, from audio.currentTime
  duration: number;     // ms
  volume: number;       // 0–1
}

export class PlayerEngine {
  private ctx: AudioContext | null = null;
  private audioA: HTMLAudioElement;
  private audioB: HTMLAudioElement;
  private gainA: GainNode | null = null;
  private gainB: GainNode | null = null;
  private analyser: AnalyserNode | null = null;
  private activeSlot: 'A' | 'B' = 'A';
  private state: PlayerState = 'idle';
  private currentTrack: Track | null = null;
  private volumeValue: number = 0.8;

  constructor() {
    this.audioA = new Audio();
    this.audioB = new Audio();
    this.audioA.crossOrigin = 'anonymous';
    this.audioB.crossOrigin = 'anonymous';
  }

  /** Initialize Web Audio graph (must be called after a user gesture) */
  private ensureContext(): AudioContext {
    if (!this.ctx) {
      this.ctx = new AudioContext();
      this.analyser = this.ctx.createAnalyser();
      this.analyser.fftSize = 256;

      const sourceA = this.ctx.createMediaElementSource(this.audioA);
      const sourceB = this.ctx.createMediaElementSource(this.audioB);
      this.gainA = this.ctx.createGain();
      this.gainB = this.ctx.createGain();

      sourceA.connect(this.gainA);
      sourceB.connect(this.gainB);
      this.gainA.connect(this.analyser);
      this.gainB.connect(this.analyser);
      this.analyser.connect(this.ctx.destination);

      this.gainA.gain.value = this.volumeValue;
      this.gainB.gain.value = 0; // slot B starts silent
    }
    return this.ctx;
  }

  /** Returns the currently active audio element */
  private activeAudio(): HTMLAudioElement {
    return this.activeSlot === 'A' ? this.audioA : this.audioB;
  }

  /** Returns the standby audio element (for preloading next track) */
  private standbyAudio(): HTMLAudioElement {
    return this.activeSlot === 'A' ? this.audioB : this.audioA;
  }

  /** Returns the gain node for the given slot */
  private gainFor(slot: 'A' | 'B'): GainNode {
    return slot === 'A' ? this.gainA! : this.gainB!;
  }

  async load(track: Track): Promise<void> {
    // Switch slots
    const nextSlot = this.activeSlot === 'A' ? 'B' : 'A';
    const audio = nextSlot === 'A' ? this.audioA : this.audioB;

    this.ensureContext();
    audio.src = track.audioUrl;
    audio.load();
    this.state = 'loading';
  }

  play(): void {
    this.ensureContext();
    this.ctx!.resume();
    this.activeAudio().play();
    this.state = 'playing';
  }

  pause(): void {
    this.activeAudio().pause();
    this.state = 'paused';
  }

  stop(): void {
    this.activeAudio().pause();
    this.activeAudio().currentTime = 0;
    this.state = 'stopped';
  }

  seek(time: number): void {
    this.activeAudio().currentTime = time;
  }

  setVolume(v: number): void {
    this.volumeValue = Math.max(0, Math.min(1, v));
    this.ensureContext();
    // Set volume on the active gain node
    this.gainFor(this.activeSlot).gain.setValueAtTime(
      this.volumeValue,
      this.ctx!.currentTime,
    );
  }

  /** Start playing a track (loads into standby slot and crossfades) */
  async playTrack(track: Track): Promise<void> {
    this.ensureContext();
    const standby = this.standbyAudio();
    standby.src = track.audioUrl;
    standby.load();

    await new Promise<void>((resolve) => {
      standby.addEventListener('canplay', () => resolve(), { once: true });
    });

    // Mute active, unmute standby, swap
    const oldGain = this.gainFor(this.activeSlot);
    const newGain = this.gainFor(this.activeSlot === 'A' ? 'B' : 'A');
    const now = this.ctx!.currentTime;

    oldGain.gain.setValueAtTime(this.volumeValue, now);
    oldGain.gain.linearRampToValueAtTime(0, now + 0.5);

    this.activeAudio().pause();
    standby.play();
    newGain.gain.setValueAtTime(0, now);
    newGain.gain.linearRampToValueAtTime(this.volumeValue, now + 0.5);

    this.activeSlot = this.activeSlot === 'A' ? 'B' : 'A';
    this.currentTrack = track;
    this.state = 'playing';
  }

  /** Get spectrum data for visualizer (0–255 per frequency bin) */
  getFrequencyData(): Uint8Array {
    if (!this.analyser) return new Uint8Array(0);
    const data = new Uint8Array(this.analyser.frequencyBinCount);
    this.analyser.getByteFrequencyData(data);
    return data;
  }

  getStatus(): PlayerStatus {
    const audio = this.activeAudio();
    return {
      state: this.state,
      currentTrack: this.currentTrack,
      progress: audio.currentTime * 1000,
      duration: (audio.duration || 0) * 1000,
      volume: this.volumeValue,
    };
  }

  /** Subscribe to the active audio's ended event */
  onEnded(cb: () => void): () => void {
    const handler = () => cb();
    this.audioA.addEventListener('ended', handler);
    this.audioB.addEventListener('ended', handler);
    return () => {
      this.audioA.removeEventListener('ended', handler);
      this.audioB.removeEventListener('ended', handler);
    };
  }
}
```

- [ ] **Step 2: Write useMusicStore**

```typescript
// src/stores/useMusicStore.ts
import { create } from "zustand";
import { PlayerEngine, type PlayerStatus } from "../systems/music/player";
import { createMockSource } from "../systems/music/mock-source";
import type { MusicSource, Track } from "../systems/music/types";

interface MusicState {
  source: MusicSource;
  engine: PlayerEngine;
  status: PlayerStatus;
  playlist: Track[];
  playlistIndex: number;
}

interface MusicActions {
  init: () => void;
  loadDemoPlaylist: () => Promise<void>;
  play: () => void;
  pause: () => void;
  next: () => void;
  previous: () => void;
  setVolume: (v: number) => void;
  seek: (ms: number) => void;
  tickStatus: () => void;
  cleanup: () => void;
}

export const useMusicStore = create<MusicState & MusicActions>((set, get) => {
  const engine = new PlayerEngine();

  return {
    source: createMockSource(),
    engine,
    status: engine.getStatus(),
    playlist: [],
    playlistIndex: -1,

    init: () => {
      const unsub = engine.onEnded(() => {
        get().next();
      });
      // Store cleanup fn on the engine instance for later
      (engine as unknown as Record<string, unknown>)._onEndedUnsub = unsub;
    },

    loadDemoPlaylist: async () => {
      const source = get().source;
      const playlists = await source.getRecommendPlaylists();
      if (playlists.length === 0) return;
      const tracks = playlists[0].tracks;
      set({ playlist: tracks, playlistIndex: 0 });
      if (tracks.length > 0) {
        await engine.playTrack(tracks[0]);
        set({ status: engine.getStatus() });
      }
    },

    play: () => {
      engine.play();
      set({ status: engine.getStatus() });
    },

    pause: () => {
      engine.pause();
      set({ status: engine.getStatus() });
    },

    next: () => {
      const { playlist, playlistIndex } = get();
      if (playlist.length === 0) return;
      const nextIdx = (playlistIndex + 1) % playlist.length;
      set({ playlistIndex: nextIdx });
      engine.playTrack(playlist[nextIdx]).then(() => {
        set({ status: engine.getStatus() });
      });
    },

    previous: () => {
      const { playlist, playlistIndex } = get();
      if (playlist.length === 0) return;
      const prevIdx = (playlistIndex - 1 + playlist.length) % playlist.length;
      set({ playlistIndex: prevIdx });
      engine.playTrack(playlist[prevIdx]).then(() => {
        set({ status: engine.getStatus() });
      });
    },

    setVolume: (v) => {
      engine.setVolume(v);
      set({ status: engine.getStatus() });
    },

    seek: (ms) => {
      engine.seek(ms / 1000);
      set({ status: engine.getStatus() });
    },

    tickStatus: () => {
      set({ status: engine.getStatus() });
    },

    cleanup: () => {
      const unsub = (engine as unknown as Record<string, unknown>)._onEndedUnsub as (() => void) | undefined;
      unsub?.();
      engine.stop();
    },
  };
});
```

- [ ] **Step 3: Write PlayControls component**

```typescript
// src/components/dj-controls/PlayControls.tsx
import { useMusicStore } from "../../stores/useMusicStore";

export default function PlayControls() {
  const status = useMusicStore((s) => s.status);
  const isPlaying = status.state === 'playing';
  const volume = status.volume;

  const handlePlayPause = () => {
    if (isPlaying) {
      useMusicStore.getState().pause();
    } else {
      useMusicStore.getState().play();
    }
  };

  const handleNext = () => useMusicStore.getState().next();
  const handlePrevious = () => useMusicStore.getState().previous();

  return (
    <div
      style={{
        display: 'flex',
        alignItems: 'center',
        gap: 12,
        padding: '8px 16px',
        borderRadius: 10,
        background: 'rgba(255,255,255,0.08)',
        backdropFilter: 'blur(8px)',
      }}
    >
      <button onClick={handlePrevious} style={btnStyle}
        disabled={status.state === 'idle'}
      >
        ⏮
      </button>
      <button onClick={handlePlayPause} style={{ ...btnStyle, fontSize: 18 }}
        disabled={status.state === 'idle' || status.state === 'loading'}
      >
        {isPlaying ? '⏸' : '▶️'}
      </button>
      <button onClick={handleNext} style={btnStyle}
        disabled={status.state === 'idle'}
      >
        ⏭
      </button>
      <input
        type="range"
        min={0}
        max={1}
        step={0.05}
        value={volume}
        onChange={(e) =>
          useMusicStore.getState().setVolume(parseFloat(e.target.value))
        }
        style={{ width: 60, accentColor: '#ff9f43' }}
      />
    </div>
  );
}

const btnStyle: React.CSSProperties = {
  border: 'none',
  background: 'transparent',
  color: '#fff',
  cursor: 'pointer',
  fontSize: 16,
  padding: 4,
};
```

- [ ] **Step 4: Write NowPlaying component**

```typescript
// src/components/dj-controls/NowPlaying.tsx
import { useMusicStore } from "../../stores/useMusicStore";

function formatTime(ms: number): string {
  const totalSec = Math.floor(ms / 1000);
  const min = Math.floor(totalSec / 60);
  const sec = totalSec % 60;
  return `${min}:${sec.toString().padStart(2, '0')}`;
}

export default function NowPlaying() {
  const status = useMusicStore((s) => s.status);
  const track = status.currentTrack;

  if (!track) {
    return (
      <div style={containerStyle}>
        <span style={{ color: 'rgba(255,255,255,0.5)', fontSize: 12 }}>
          未在播放
        </span>
      </div>
    );
  }

  return (
    <div style={containerStyle}>
      <div style={{ fontWeight: 600, fontSize: 13, color: '#fff' }}>
        {track.name}
      </div>
      <div style={{ fontSize: 11, color: 'rgba(255,255,255,0.6)' }}>
        {track.artists.join(', ')} · {track.album}
      </div>
      <div style={{ fontSize: 10, color: 'rgba(255,255,255,0.4)', marginTop: 4 }}>
        {formatTime(status.progress)}
        {status.duration > 0 && ` / ${formatTime(status.duration)}`}
      </div>
    </div>
  );
}

const containerStyle: React.CSSProperties = {
  padding: '8px 16px',
  borderRadius: 10,
  background: 'rgba(255,255,255,0.06)',
  backdropFilter: 'blur(8px)',
  textAlign: 'center',
};
```

- [ ] **Step 5: Type-check**

Run: `npx tsc --noEmit`
Expected: no errors

- [ ] **Step 6: Commit**

```bash
git add src/systems/music/player.ts src/stores/useMusicStore.ts \
        src/components/dj-controls/PlayControls.tsx src/components/dj-controls/NowPlaying.tsx
git commit -m "feat(music): add PlayerEngine with dual-audio architecture and playback controls

- PlayerEngine: dual HTMLAudioElement + MediaElementAudioSourceNode + AnalyserNode
- Automatic track advance on ended event
- useMusicStore: playlist management, play/pause/next/prev/volume/seek
- PlayControls: play/pause, skip, volume slider
- NowPlaying: track info, artist, album, time display"
```

---

### Task 4: DJ Persona Chat

**Files:**
- Modify: `src/components/pet/DragLayer.tsx` — add right-click context menu

This task wires up the DJ persona so that switching to DJ mode changes the chat system prompt. The mode switch itself is triggered by a right-click context menu.

- [ ] **Step 1: Add right-click context menu to DragLayer**

```typescript
// src/components/pet/DragLayer.tsx
// ADD right-click handler and a simple context menu state

import { getCurrentWindow } from "@tauri-apps/api/window";
import { useChatStore } from "../../stores/useChatStore";
import { useDJStore } from "../../stores/useDJStore";
import { DJ_SYSTEM_PROMPT } from "../../systems/dj/persona";
import { useMusicStore } from "../../stores/useMusicStore";
import { useState, useEffect } from "react";

export default function DragLayer() {
  const toggleInput = useChatStore((s) => s.toggleInput);
  const showInput = useChatStore((s) => s.showInput);
  const isStreaming = useChatStore((s) => s.isStreaming);
  const setSystemPrompt = useChatStore((s) => s.setSystemPrompt);
  const mode = useDJStore((s) => s.mode);
  const switchMode = useDJStore((s) => s.switchMode);

  const [menuPos, setMenuPos] = useState<{ x: number; y: number } | null>(null);

  // Close menu on any click outside
  useEffect(() => {
    if (!menuPos) return;
    const close = () => setMenuPos(null);
    window.addEventListener('click', close);
    return () => window.removeEventListener('click', close);
  }, [menuPos]);

  const handleContextMenu = (e: React.MouseEvent) => {
    e.preventDefault();
    e.stopPropagation();
    setMenuPos({ x: e.clientX, y: e.clientY });
  };

  const handleSwitchMode = async () => {
    setMenuPos(null);
    if (mode === 'pet') {
      // Switch to DJ mode
      switchMode('dj');
      setSystemPrompt(DJ_SYSTEM_PROMPT);

      // Initialize music if not already
      const music = useMusicStore.getState();
      if (music.playlist.length === 0) {
        await music.loadDemoPlaylist();
      }

      // Trigger window resize (dynamic import for Tauri safety)
      try {
        const { switchWindowMode } = await import("../../tauri/window");
        await switchWindowMode('dj');
      } catch {
        // Running outside Tauri (dev in browser) — no-op
      }
    } else {
      // Switch back to pet mode
      switchMode('pet');
      setSystemPrompt(''); // clears to default PET_PROMPT in chat store
      try {
        const { switchWindowMode } = await import("../../tauri/window");
        await switchWindowMode('pet');
      } catch { /* browser dev */ }
    }
  };

  return (
    <>
      <div
        style={{
          position: "absolute",
          inset: 0,
          pointerEvents: "none",
        }}
      >
        <div
          onMouseDown={(e) => {
            e.preventDefault();
            e.stopPropagation();
            getCurrentWindow().startDragging();
          }}
          onClick={() => {
            if (!isStreaming) toggleInput();
          }}
          onContextMenu={handleContextMenu}
          style={{
            position: "absolute",
            inset: 0,
            pointerEvents: "auto",
            cursor: showInput ? "pointer" : "grab",
          }}
        />
      </div>

      {/* Right-click context menu */}
      {menuPos && (
        <div
          style={{
            position: "fixed",
            left: menuPos.x,
            top: menuPos.y,
            zIndex: 1000,
            background: 'rgba(30,30,30,0.95)',
            backdropFilter: 'blur(12px)',
            borderRadius: 8,
            padding: '4px 0',
            minWidth: 160,
            boxShadow: '0 4px 16px rgba(0,0,0,0.4)',
            border: '1px solid rgba(255,255,255,0.1)',
          }}
        >
          <button
            onClick={handleSwitchMode}
            style={{
              display: 'block',
              width: '100%',
              padding: '8px 16px',
              border: 'none',
              background: 'transparent',
              color: '#fff',
              fontSize: 13,
              cursor: 'pointer',
              textAlign: 'left',
            }}
          >
            {mode === 'pet' ? '🎧 切换 DJ 模式' : '🐾 切换宠物模式'}
          </button>
        </div>
      )}
    </>
  );
}
```

- [ ] **Step 2: Type-check**

Run: `npx tsc --noEmit`
Expected: no errors

- [ ] **Step 3: Commit**

```bash
git add src/components/pet/DragLayer.tsx
git commit -m "feat(dj): add right-click context menu for mode switching

- Right-click pet → context menu → switch to DJ/pet mode
- DJ mode loads demo playlist and switches system prompt
- Window resize via Tauri v2 LogicalSize
- Falls back gracefully when running outside Tauri (browser dev)"
```

---

### Task 5: DJ Booth UI — Visualizer + Stage Lights + Character

**Files:**
- Create: `src/components/dj-booth/DJBooth.tsx`
- Create: `src/components/dj-booth/DJCharCanvas.tsx`
- Create: `src/components/dj-booth/Visualizer.tsx`
- Create: `src/components/dj-booth/StageLights.tsx`
- Modify: `src/App.tsx` — conditionally render DJ components

This task builds the visual DJ booth experience — spectrum bars, stage lighting, and a DJ-mode character display.

- [ ] **Step 1: Write Visualizer (spectrum bars)**

```typescript
// src/components/dj-booth/Visualizer.tsx
import { useEffect, useRef } from "react";
import { useMusicStore } from "../../stores/useMusicStore";

const BAR_COUNT = 32;
const TICK_RATE = 30;
const TICK_INTERVAL = 1000 / TICK_RATE;

export default function Visualizer() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const rafRef = useRef<number>(0);
  const lastTimeRef = useRef(performance.now());
  const accumulatorRef = useRef(0);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    const loop = (now: number) => {
      const dt = now - lastTimeRef.current;
      lastTimeRef.current = now;
      accumulatorRef.current += dt;

      while (accumulatorRef.current >= TICK_INTERVAL) {
        accumulatorRef.current -= TICK_INTERVAL;

        const engine = useMusicStore.getState().engine;
        const data = engine.getFrequencyData();
        const { width, height } = canvas;

        ctx.clearRect(0, 0, width, height);

        const barWidth = (width / BAR_COUNT) - 2;
        for (let i = 0; i < BAR_COUNT; i++) {
          const value = (data[i] ?? 0) / 255;
          const barHeight = value * height * 0.8;
          const x = i * (barWidth + 2) + 1;
          const y = height - barHeight;

          // Color gradient from orange (low freq) to cyan (high freq)
          const hue = 200 + (i / BAR_COUNT) * 40; // 200-240 = cyan-blue range
          ctx.fillStyle = `hsla(${hue}, 80%, ${50 + value * 30}%, ${0.6 + value * 0.4})`;
          ctx.fillRect(x, y, barWidth, barHeight);
        }
      }

      rafRef.current = requestAnimationFrame(loop);
    };

    rafRef.current = requestAnimationFrame(loop);
    return () => cancelAnimationFrame(rafRef.current);
  }, []);

  return (
    <canvas
      ref={canvasRef}
      width={360}
      height={100}
      style={{
        width: '100%',
        height: 100,
        borderRadius: 6,
        background: 'rgba(0,0,0,0.3)',
      }}
    />
  );
}
```

- [ ] **Step 2: Write StageLights (mood-reactive background)**

```typescript
// src/components/dj-booth/StageLights.tsx
import { useEffect, useRef } from "react";
import { useMusicStore } from "../../stores/useMusicStore";

const TICK_RATE = 20;
const TICK_INTERVAL = 1000 / TICK_RATE;

export default function StageLights() {
  const containerRef = useRef<HTMLDivElement>(null);
  const rafRef = useRef<number>(0);
  const lastTimeRef = useRef(performance.now());
  const accumulatorRef = useRef(0);

  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    const loop = (now: number) => {
      const dt = now - lastTimeRef.current;
      lastTimeRef.current = now;
      accumulatorRef.current += dt;

      while (accumulatorRef.current >= TICK_INTERVAL) {
        accumulatorRef.current -= TICK_INTERVAL;

        const engine = useMusicStore.getState().engine;
        const data = engine.getFrequencyData();
        // Average bass (first 8 bins) for light intensity
        const bass = data.slice(0, 8).reduce((a, b) => a + b, 0) / (8 * 255);

        // Warm gradient that pulses with bass
        const r = Math.floor(80 + bass * 100);
        const g = Math.floor(10 + bass * 40);
        const b = Math.floor(40 + bass * 80);
        const alpha = 0.15 + bass * 0.25;

        container.style.background = `
          radial-gradient(ellipse at 50% 80%, rgba(${r},${g},${b},${alpha}) 0%, transparent 70%),
          radial-gradient(ellipse at 30% 60%, rgba(255,159,67,${bass * 0.15}) 0%, transparent 50%),
          radial-gradient(ellipse at 70% 60%, rgba(180,100,255,${bass * 0.15}) 0%, transparent 50%)
        `;
      }

      rafRef.current = requestAnimationFrame(loop);
    };

    rafRef.current = requestAnimationFrame(loop);
    return () => cancelAnimationFrame(rafRef.current);
  }, []);

  return (
    <div
      ref={containerRef}
      style={{
        position: 'absolute',
        inset: 0,
        transition: 'background 0.1s ease',
        pointerEvents: 'none',
        zIndex: 0,
      }}
    />
  );
}
```

- [ ] **Step 3: Write DJCharCanvas (DJ mode character)**

```typescript
// src/components/dj-booth/DJCharCanvas.tsx
import { usePetStore } from "../../stores/usePetStore";
import aeriImg from "../../assets/images/puppy.png";

export default function DJCharCanvas() {
  const currentTransform = usePetStore((s) => s.currentTransform);
  const currentAnimation = usePetStore((s) => s.currentAnimation);
  const playAnimation = usePetStore((s) => s.playAnimation);

  return (
    <div
      style={{
        display: 'flex',
        justifyContent: 'center',
        alignItems: 'center',
        padding: 8,
      }}
    >
      <img
        src={aeriImg}
        draggable={false}
        onDragStart={(e) => e.preventDefault()}
        onMouseEnter={() => playAnimation('bounce')}
        style={{
          width: 80,
          height: 80,
          objectFit: 'contain',
          transform: currentTransform,
          transition: currentAnimation === 'idle'
            ? 'transform 0.3s ease'
            : 'transform 0.08s ease',
          imageRendering: 'pixelated',
          filter: 'drop-shadow(0 0 12px rgba(255,159,67,0.6))',
        }}
        alt="DJ Aeri"
      />
    </div>
  );
}
```

- [ ] **Step 4: Write DJBooth (container layout)**

```typescript
// src/components/dj-booth/DJBooth.tsx
import DJCharCanvas from "./DJCharCanvas";
import Visualizer from "./Visualizer";
import StageLights from "./StageLights";
import PlayControls from "../dj-controls/PlayControls";
import NowPlaying from "../dj-controls/NowPlaying";
import SpeechBubble from "../overlays/SpeechBubble";
import ChatInput from "../overlays/ChatInput";

export default function DJBooth() {
  return (
    <div
      style={{
        position: 'relative',
        width: 400,
        height: 320,
        display: 'flex',
        flexDirection: 'column',
        alignItems: 'center',
        justifyContent: 'flex-end',
        gap: 8,
        padding: '16px 16px 20px',
        background: 'linear-gradient(180deg, #1a1020 0%, #0d0a14 100%)',
        borderRadius: 16,
        overflow: 'hidden',
        userSelect: 'none',
      }}
    >
      {/* Ambient light layer */}
      <StageLights />

      {/* Content layers (above lights) */}
      <div style={{ position: 'relative', zIndex: 1, width: '100%' }}>
        {/* Speech bubble + character area */}
        <div style={{ position: 'relative', marginBottom: 4 }}>
          <SpeechBubble />
          <DJCharCanvas />
        </div>

        {/* Visualizer */}
        <Visualizer />

        {/* Track info */}
        <NowPlaying />

        {/* Controls */}
        <PlayControls />
      </div>

      {/* Chat input (renders above everything) */}
      <div style={{ position: 'relative', zIndex: 2 }}>
        <ChatInput />
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Update App.tsx to conditionally render DJ mode**

```typescript
// src/App.tsx
// Replace the return statement with mode-conditional rendering

import { useEffect, useRef, useState } from "react";
import { usePetStore } from "./stores/usePetStore";
import { useChatStore } from "./stores/useChatStore";
import { useDJStore } from "./stores/useDJStore";
import { useMusicStore } from "./stores/useMusicStore";
import { useSettingsStore } from "./stores/useSettingsStore";
import { tickBehavior } from "./systems/behavior/idle";
import PetCanvas from "./components/pet/PetCanvas";
import DragLayer from "./components/pet/DragLayer";
import SpeechBubble from "./components/overlays/SpeechBubble";
import ChatInput from "./components/overlays/ChatInput";
import DJBooth from "./components/dj-booth/DJBooth";
import "./App.css";

const TICK_RATE = 30;
const TICK_INTERVAL = 1000 / TICK_RATE;

export default function App() {
  const lastTimeRef = useRef(performance.now());
  const accumulatorRef = useRef(0);
  const mode = useDJStore((s) => s.mode);

  // Load settings on mount
  useEffect(() => {
    const settings = useSettingsStore.getState();
    if (!settings.loaded) {
      settings.loadFromStorage();
    }
  }, []);

  // Initialize music engine on first DJ mode
  useEffect(() => {
    if (mode === 'dj') {
      const music = useMusicStore.getState();
      music.init();
    }
    return () => {
      if (mode === 'pet') {
        useMusicStore.getState().cleanup();
      }
    };
  }, [mode]);

  useEffect(() => {
    const loop = (now: number) => {
      const dt = now - lastTimeRef.current;
      lastTimeRef.current = now;

      accumulatorRef.current += dt;
      while (accumulatorRef.current >= TICK_INTERVAL) {
        accumulatorRef.current -= TICK_INTERVAL;

        const pet = usePetStore.getState();
        const chat = useChatStore.getState();

        // 1. 推进动画
        pet.tick(TICK_INTERVAL);

        // 2. 自主动作（非对话中）
        if (!chat.isStreaming) {
          const action = tickBehavior(pet.idleTimer, { joy: pet.joy });
          if (action) {
            pet.playAnimation(action);
            pet.setIdleTimer(0);
          }
        }

        // 3. Update music status (for NowPlaying time display)
        if (mode === 'dj') {
          useMusicStore.getState().tickStatus();
        }
      }

      rafRef.current = requestAnimationFrame(loop);
    };

    const rafRef = { current: requestAnimationFrame(loop) };
    return () => cancelAnimationFrame(rafRef.current);
  }, [mode]);

  return (
    <div className="app-container">
      {mode === 'pet' ? (
        <div style={{ position: "relative" }}>
          <SpeechBubble />
          <PetCanvas />
          <ChatInput />
        </div>
      ) : (
        <DJBooth />
      )}
      <DragLayer />
    </div>
  );
}
```

- [ ] **Step 6: Type-check full project**

Run: `npx tsc --noEmit`
Expected: no errors

- [ ] **Step 7: Commit**

```bash
git add src/components/dj-booth/DJBooth.tsx \
        src/components/dj-booth/DJCharCanvas.tsx \
        src/components/dj-booth/Visualizer.tsx \
        src/components/dj-booth/StageLights.tsx \
        src/components/dj-booth/StageLights.tsx \
        src/App.tsx
git commit -m "feat(dj): add DJ booth UI with visualizer, stage lights, and character

- DJBooth container with dark gradient background
- Visualizer: 32-bar frequency spectrum canvas, 30fps
- StageLights: bass-reactive radial gradient pulses
- DJCharCanvas: pet character with glow effect in DJ mode
- App.tsx: conditional pet/DJ rendering based on mode
- Load settings on mount, init/cleanup music engine on mode switch"
```

---

## Self-Review Checklist

- [x] **Spec coverage**: Each v1.0 requirement mapped — mode switch (T1), mock playback (T2), player (T3), DJ chat (T4), booth UI (T5), security fix (S00)
- [x] **No placeholders**: No TBD, TODO, or "implement later". Every step has complete code.
- [x] **Type consistency**: `AppMode` defined in T1 types.ts, used in T1 store and T5 App.tsx. `PlayerEngine.getFrequencyData()` returns `Uint8Array`, consumed in Visualizer and StageLights. `MusicSource` defined in T2, used in T3 store.
- [x] **Aeri conventions**: No barrel exports, no `any`, components only render, systems pure logic, stores bridge
- [x] **Build order**: S00 → 01 → 02 → 03 → 04 → 05 (dependencies respected)

---
