# Frost AI DJ — UI 界面生成提示词

> 基于 `/Users/frost/frost-ai-dj/index.html` 最终版本提炼。所有数值、色值、时序均从实际代码提取，可直接作为界面还原的系统指令。

---

## 1. 项目概述

Frost 是一个私人 AI DJ 的 PWA 前端界面。视觉风格定为 **"冷调复古电台"**——深色星空背景 + 霜雪粒子 + 中心卡片式字幕流，参考 Spotify AI DJ (DJ X) 的沉浸式体验。

---

## 2. 设计系统

### 2.1 色彩变量

```css
--bg-dark: #060913;       /* 最深背景 */
--bg-light: #111827;      /* 渐变亮端 */
--card-bg-top: rgba(30, 41, 59, 0.55);    /* 卡片顶部 */
--card-bg-bottom: rgba(15, 23, 42, 0.78); /* 卡片底部 */
--text-main: #ffffff;
--green-dot: #4ade80;     /* 呼吸指示灯 */
```

背景使用径向渐变：`radial-gradient(circle at 50% 20%, #111827 0%, #060913 100%)`

### 2.2 字体系统（三层）

| 用途 | 字体 | 来源 |
|---|---|---|
| **Logo** | `Codystar` (sans-serif) | Google Fonts — 点阵字体，复古科技感 |
| **状态栏** | `Share Tech Mono` (monospace) | Google Fonts — 等宽字体，无线电设备感 |
| **字幕 / 歌曲信息** | `Newsreader` (serif) | Google Fonts — 衬线体，文学性、电台 DJ 朗读诗歌感 |

Google Fonts 导入：
```
@import url('https://fonts.googleapis.com/css2?family=Codystar&family=Share+Tech+Mono&family=Newsreader:opsz,wght@6..72,300;6..72,400;6..72,500&display=swap');
```

### 2.3 间距与尺寸

- 卡片宽度：`min(85vw, 460px)`
- 卡片高度：`65vh`
- 卡片距底部：`6vh`
- 卡片圆角：`36px`
- Logo 字号：`46px`，字距 `8px`
- 状态栏字号：`15px`，字距 `1px`

---

## 3. 图层架构 (z-index)

```
z-index  元素
────────────────────────────────
 1       #snow         背景霜雪粒子 (Canvas)
 2       .dust-glow    底部呼吸光晕
10       .top-bar      顶部状态栏
10       .card         字幕卡片容器
12       #snow-fg      前景雪花 + 积雪 (Canvas, pointer-events: none)
15       .card::before/::after  卡片顶/底渐变遮罩
20       .now-playing  歌曲信息 (卡片内部)
```

所有 Canvas 和装饰元素设置 `pointer-events: none`，不阻挡交互。

---

## 4. 组件规格

### 4.1 顶部状态栏 (.top-bar)

- 位置：`top: 9%`，水平居中 (`left: 50%; transform: translateX(-50%)`)
- 布局：`display: flex; align-items: center; gap: 20px`

**Logo (.logo)**：
- 字体 `Codystar`，`font-size: 46px`，`letter-spacing: 8px`
- 颜色 `#f8fafc`，文字发光 `text-shadow: 0 0 12px rgba(255,255,255,0.4)`

**状态指示器 (.status-wrapper)**：
- 字体 `Share Tech Mono`，`font-size: 15px`
- 颜色 `#94a3b8`，`letter-spacing: 1px`
- 绿色呼吸圆点 (`.dot`)：`10×10px`，`border-radius: 50%`，`background: #4ade80`
- 呼吸动画：2.5s 周期，opacity 在 0.3 ↔ 1 之间，box-shadow 在 4px ↔ 12px 之间
- 实时时钟 (`#time-display`)：每 1s 更新，格式 `h:mm AM/PM`

### 4.2 字幕卡片 (.card)

- 定位：`position: absolute; bottom: 6vh; left: 50%; transform: translateX(-50%)`
- 尺寸：`width: min(85vw, 460px); height: 65vh`
- 背景：自上而下线性渐变，从 `rgba(30,41,59,0.55)` 到 `rgba(15,23,42,0.78)`
- 边框：`1px solid rgba(255,255,255,0.1)`，顶部边框稍亮 `rgba(255,255,255,0.18)`
- 圆角：`36px`
- 阴影（三层）：
  1. `0 40px 80px rgba(0,0,0,0.85)` — 外部投影
  2. `inset 0 0 40px rgba(0,0,0,0.4)` — 内阴影
  3. `0 0 80px rgba(56,189,248,0.06)` — 微弱蓝色外发光
- 毛玻璃：`backdrop-filter: blur(24px)`
- `overflow: hidden`

**顶/底渐变遮罩 (::before / ::after)**：
- 高度 `50px`，z-index: 15
- 顶部：`rgba(18,26,43,1)` → `transparent`（从上到下）
- 底部：`rgba(11,16,28,1)` → `transparent`（从下到上）
- 作用：让字幕在卡片边缘自然淡出，不突兀截断

### 4.3 五层级字幕流

字幕容器 `.transcript` 为 `position: relative; width: 100%; height: 100%`。

**核心约束**：
- 严格单行：`white-space: nowrap; overflow: hidden`
- 始终同时显示 **恰好 5 行**（历史 2 行 + 当前 1 行 + 预告 2 行）
- 文本超长时通过 JavaScript 按 32 字符截断为多段

**五层级定位与样式**：

| 层级 | 类名 | `top` | `font-size` | `font-weight` | `opacity` | `filter: blur()` | `color` | `scale` |
|---|---|---|---|---|---|---|---|---|
| 历史-2 | `.past-2` | 22% | 14px | 300 | 0.3 | 1.5px | #94a3b8 | 0.8 |
| 历史-1 | `.past-1` | 33% | 16px | 300 | 0.5 | 0.5px | #cbd5e1 | 0.9 |
| 当前 | `.current` | 44% | 20px | 400 | 1 | 0px | #ffffff | 1 |
| 预告+1 | `.future-1` | 55% | 16px | 300 | 0.5 | 0.5px | #cbd5e1 | 0.9 |
| 预告+2 | `.future-2` | 66% | 14px | 300 | 0.3 | 1.5px | #94a3b8 | 0.8 |

**层级间距**：相邻层级 top 差值为 11%，总共覆盖 22% → 66%（44% 跨度），视觉紧凑不松散。

**可见度设计原则**：
- 第一级和第五级（past-2 / future-2）虽然模糊（1.5px），但 opacity 保持 0.3，确保仍可辨认文字内容
- 第二级和第四级（past-1 / future-1）模糊减半（0.5px），opacity 0.5，过渡更自然
- 当前行无模糊、全不透明、轻微文字发光

**隐藏层级**：
- `.hidden-top`：`top: -5%; font-size: 12px; opacity: 0`
- `.hidden-bottom`：`top: 105%; font-size: 12px; opacity: 0`

### 4.4 过渡动画（关键）

基类 `.line` 使用 **分属性差异化过渡**，避免模糊消除时的突兀感：

```css
transition:
  top        1.3s cubic-bezier(0.22, 1, 0.36, 1),  /* 位移 — 舒缓 */
  font-size  1.3s cubic-bezier(0.22, 1, 0.36, 1),  /* 字号 — 同步位移 */
  transform  1.3s cubic-bezier(0.22, 1, 0.36, 1),  /* 缩放 — 同步位移 */
  opacity    0.7s ease-out,                          /* 亮度 — 更快到位 */
  filter     0.5s ease-out,                          /* 模糊 — 最快消散 */
  color      1.0s ease,                              /* 颜色 — 温和过渡 */
  text-shadow 0.8s ease;                            /* 光晕 — 随亮度浮现 */
```

**设计原理**：`filter: blur()` 从 0.5px → 0px 在 0.5s 内完成，远快于位置变化的 1.3s。这意味着字幕在滑动到中央位置之前就已经变清晰，用户不会感知到"到中央突然变亮"的跳跃。使用 `ease-out` 缓动让模糊在最初几帧迅速消散。

缓动曲线 `cubic-bezier(0.22, 1, 0.36, 1)` 是类似 ease-out-expo 的自定义曲线，比标准 ease 更有"惯性减速"感。

### 4.5 歌曲信息 (.now-playing)

- 位置：卡片内左上角，`top: 16px; left: 22px`
- 字体：与字幕相同的 `Newsreader` serif
- 入场动画：`opacity 0→1 + translateY(8px→0)`，duration 1s
- JS 更新时通过移除/添加 `.visible` 类触发淡入

**弱化处理**（不抢字幕焦点）：
- 标签 `.np-label`：`font-size: 8px`，`letter-spacing: 3px`，`color: rgba(148,163,184,0.3)`
- 标题 `.np-title`：`font-size: 13px`，`italic`，`color: rgba(255,255,255,0.48)`
- 副标题 `.np-sub`：`font-size: 11px`，`font-weight: 300`，`color: rgba(203,213,225,0.32)`

`.now-playing` 可复用于展示歌曲名/作者，后续也可展示博客标题等媒体信息。

### 4.6 底部光晕 (.dust-glow)

- 位置：`bottom: -15%`，水平居中
- 尺寸：`500×400px`
- 背景：径向渐变，中心 `rgba(56,189,248,0.08)` → 70% 处完全透明
- 动画：8s 周期，scale 在 0.9 ↔ 1.2，opacity 在 0.6 ↔ 1，`filter: blur(50px)`
- z-index: 2，`pointer-events: none`

---

## 5. 雪花系统（双层 Canvas）

### 5.1 背景霜雪 (#snow, z-index: 1)

- 80 个静止粒子随机分布在画布上
- 每粒 `1-3px`，`opacity: 0.1-0.4`
- 缓慢匀速下落 (`speedY: 0.05-0.15 px/frame`)
- 落出屏幕后重置到顶部

**作用**：营造深邃的星空/霜雪氛围背景。

### 5.2 前景雪 + 积雪 (#snow-fg, z-index: 12)

#### 飘落物理

- 70 颗雪花，尺寸 `1.0-3.5px`，不透明度 `0.35-0.8`
- 从屏幕顶端随机位置生成（初始 y 分布在负值区域，分批入场）
- 每片独立物理参数：
  - `speedY: 0.25-0.95` — 变速下落
  - `wobbleAmp: 0.15-0.8` — 水平摆动幅度
  - `wobbleSpeed: 0.006-0.031` — 摆动频率
  - `wobblePhase` — 每片独立相位
- 风：全局变量 `wind`，每 2.5-7.5 秒随机切换目标值（范围 ±0.35），平滑插值过渡
- 水平运动：`x += wind * (1.2 - size/4) + sin(wobblePhase) * wobbleAmp`
- 水平循环：飘出左右边界后从对侧重新进入

#### 落地与积雪

- **落地判定**：使用 `getCardTopAtX(x, bounds)` 函数，严格按卡片 36px 圆角计算边界
- 圆角公式：
  - 平直段（`left+36` → `right-36`）：`y = cardTop`
  - 左圆角（`left` → `left+36`）：`y = cardTop + 36 - √(36² - (x - left - 36)²)`
  - 右圆角（`right-36` → `right`）：`y = cardTop + 36 - √(36² - (x - right + 36)²)`
- 落地时：将该 x 坐标的雪堆高度 (`snowHeightMap[x]`) 增加 `size × 0.5`，相邻像素微增 `size × 0.12`
- 积雪数量上限 500 粒，超出后移除最早落地粒子并降低对应高度

#### 两侧积雪 (drawSideDrift)

- 卡片左右各扩展 `30px` 的积雪区
- 基准高度从圆角底端 (`cardEdgeY`) 开始，越远离卡片越下沉（最大下沉 16px）
- 绘制为小圆点，模拟积雪沿圆角边缘溢出的效果

#### 渲染

- 每帧：清空画布 → 更新风 → 遍历雪花（飘落 + 落地检测）→ 绘制飘落中雪花 → 绘制已落积雪 → 绘制两侧溢出雪
- 落地的雪花即从活跃列表重生到顶端

---

## 6. JavaScript 逻辑

### 6.1 时钟

```js
setInterval(() => {
  timeDisplay.innerText = new Date().toLocaleTimeString('en-US',
    { hour: 'numeric', minute: '2-digit' });
}, 1000);
```

### 6.2 字幕截断 (chunkTexts)

输入文本数组，按 32 字符/行截断。按单词边界分割（`split(' ')`），单词不跨行。超长单词独占一行。

### 6.3 字幕滚动 (updateTranscript)

- `currentIndex` 从 2 开始（确保初始屏幕有 2 行历史）
- 每 4 秒 `currentIndex++`
- 根据 `index - currentIndex` 的 offset 分配 CSS 类（-2 → past-2, -1 → past-1, 0 → current, +1 → future-1, +2 → future-2）
- 超出范围的分配 `hidden-top` / `hidden-bottom`

### 6.4 歌曲信息同步 (updateNowPlaying)

- 当当前字幕行匹配 `Next track: <title> by <artist>` 模式时自动提取并更新
- 使用强制回流 (`void el.offsetWidth`) 技巧实现重复淡入动画
- 提供公开函数 `updateNowPlaying(title, sub)` 供后续 WebSocket 调用

---

## 7. 关键设计决策与踩坑记录

1. **积雪不跟随水平线，必须严格按圆角边界**：卡片有 `border-radius: 36px`，落地判定不能用固定 `cardTop`，必须通过圆方程逐像素计算，否则积雪漂浮在圆角之上。

2. **模糊过渡不能与位移同步**：如果 `filter: blur()` 和 `top` 都用 1.2s，模糊在位移末端才消失，产生"到中央突然变清晰"的突兀感。解决：模糊 0.5s ease-out 独立过渡，在字幕到达中央之前就已完全清晰。

3. **歌曲信息必须弱化**：卡片左上角的媒体信息与字幕同字体但极低调（opacity 0.3-0.48），避免分散用户对中央字幕流的注意力。信息展示是辅助性的，字幕流才是核心。

4. **past-1/future-1 模糊只设 0.5px**：从第二/四层到当前层只需消除 0.5px 模糊，配合 0.5s 过渡非常丝滑。第一/五层保持 1.5px。

5. **双层雪花系统**：背景雪（z-index:1）营造氛围，前景雪（z-index:12）严格模拟飘落 + 积雪。前景画布必须设置 `pointer-events: none`。

6. **卡片 ::before/::after 遮罩**：防止字幕在卡片上下边缘硬截断，50px 渐变遮罩让文字自然消隐。

---

## 8. 可扩展接口

- `updateNowPlaying(title, sub)` — 更新左上角媒体信息
- `updateTranscript()` — 推进字幕流（当前由 setInterval 4s 驱动，可改为 WebSocket 驱动）
- `rawTexts` 数组 — 字幕内容源（可从后端 API 拉取替换）
- `.now-playing` 组件 — 结构支持歌曲名/作者，可扩展为博客标题/来源等通用媒体信息展示
