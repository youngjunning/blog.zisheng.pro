---
title: HyperFrames 的原理：让 HTML 成为可寻址的视频时间线
date: 2026-09-18 16:15:00
description: HyperFrames 不是把网页录成视频的截图工具，而是一套把 HTML、媒体时间、可寻址动画和确定性渲染接在一起的运行时。本文从 composition、clip、timeline 和 media playback 四个边界出发，解释它为什么能从 HTML 生成可编辑、可复现的视频，以及哪些写法看似合理却会破坏渲染结果。
categories:
  - [软件工程]
tags:
  - HyperFrames
  - HTML
  - Video Rendering
  - GSAP
  - Deterministic Rendering
cover: /images/hyperframes-principles.webp
---

最近用 HyperFrames 做视频时，我发现最容易误解它的方式，是把它看成“带时间轴的网页截图器”。这个比喻能解释结果，却解释不了关键约束：为什么媒体播放不应该交给 GSAP，为什么 `data-duration` 和动画 timeline 不是一回事，为什么同样一段 HTML 在预览里看起来正常，渲染时却可能从错误的帧开始。

HyperFrames 真正解决的问题，是把视频重新表达成一种可寻址的网页状态：HTML 声明画面由哪些 clip 组成、每个 clip 在什么时候出现、位于哪一层；动画 Runtime 根据当前时间计算 DOM 状态；框架负责把媒体 seek 到对应位置，再捕获确定的 frame。这个边界一旦清楚，很多“玄学”都会变成可以检查的 contract。

<!-- more -->

## 一句话总结

**HyperFrames 的核心不是“用 HTML 画视频”，而是“用 HTML 声明时间，用 seekable Runtime 计算状态，用框架统一接管媒体播放”。**

一个可渲染的 composition 至少包含四类信息：

1. 画布身份与尺寸：`data-composition-id`、`data-width`、`data-height`。
2. 时间结构：`data-start`、`data-duration`、`data-track-index`。
3. 视觉状态：普通 DOM、SVG、Canvas 或动画 Runtime 产生的可复现状态。
4. 媒体边界：`video`、`audio` 的源文件、起始偏移、音量和裁剪窗口。

渲染器在时间 `t` 上做的事情，可以简化成：

```text
HTML contract + t
  → 解析当前 composition 与 clip
  → 将 video/audio seek 到 t 对应的媒体时间
  → 将动画 timeline seek 到 t
  → 捕获当前 DOM / Canvas / Audio 状态
  → 输出一帧
```

这条链路决定了 HyperFrames 的工程重点：每一帧都必须能够从时间和输入稳定推导出来，而不能依赖“刚才播放到了哪里”。

## 先分清三个对象：composition、clip 和 timeline

HyperFrames 的概念不多，但每个概念的责任边界很窄。把它们混在一起，代码通常还能在浏览器里动起来，到了逐帧渲染就会暴露问题。

| 对象 | 负责什么 | 不负责什么 |
| --- | --- | --- |
| composition | 定义一张画布、总时长、嵌套关系和变量入口 | 不直接决定每个媒体的解码位置 |
| clip | 描述一个媒体或视觉单元何时出现、持续多久、在哪个 track | 不负责动画曲线和 DOM 状态变化 |
| timeline | 在给定时间计算元素的属性、路径、透明度或变换 | 不接管媒体播放和 clip 的可见性 |

composition 是可组合的时间容器。最外层通常是一个带有 `data-composition-id` 的根节点；子 composition 通过 `data-composition-src` 加载，再由框架把自己的局部时间映射到父级时间。

clip 是时间轴上的离散单元。一个最小的视频片段可以这样写：

```html
<video
  id="intro"
  data-start="0"
  data-duration="4"
  data-track-index="0"
  src="./assets/intro.mp4"
></video>
```

`data-start` 和 `data-duration` 描述的是“它占据时间线的哪一段”，不是 CSS 动画的起止值。`data-track-index` 主要表达视觉层级和 Studio 中的轨道归属；它不是一个“同轨道绝对不能重叠”的播放锁。

timeline 则处理另一件事：在当前时间点，标题应该位于哪里，遮罩应该有多大，光晕应该有多亮。以 GSAP 为例，HyperFrames 要求 composition 注册一个 paused timeline：

```js
window.__timelines['intro'] = gsap.timeline({ paused: true })
  .fromTo('#title', { x: -48, opacity: 0 }, {
    x: 0,
    opacity: 1,
    duration: 0.8,
    ease: 'power2.out',
  });
```

这里的关键字是 `paused: true`。播放循环不由 GSAP 自己推进，而由框架在渲染第 `n` 帧时把 timeline 定位到对应时间。这样，渲染第 300 帧和从头播放到第 300 帧，应该得到同一个视觉状态。

## HTML 为什么适合做视频的源代码

HTML 的优势不在于它比专业剪辑软件“更强”，而在于它能把视频里常被藏在时间轴 UI 里的信息，变成可检查的文本结构。

例如下面的片段同时表达了三个事实：主画面从第 4 秒开始，片尾紧接着主画面，字幕和主画面在两个视觉层级上叠加。

```html
<div
  data-composition-id="demo"
  data-width="1920"
  data-height="1080"
  data-duration="12"
>
  <video id="main" data-start="0" data-duration="8" data-track-index="0" src="main.mp4"></video>
  <img id="outro" data-start="main" data-duration="4" data-track-index="0" src="outro.png" />
  <div id="caption" class="clip" data-start="main" data-duration="4" data-track-index="1">
    See you next time
  </div>
</div>
```

`data-start="main"` 是相对时间引用：`outro` 在 `main` 结束时开始。它比手写 `8` 更接近编辑意图，也减少了前面片段改时长后留下的魔法数字。

但“声明式”不等于“所有东西都写成属性”。HTML 负责描述结构和媒体窗口，脚本负责画面内部的连续变化。这个分工让检查器可以分别回答两个问题：时间轴是否完整，动画是否可以被 seek。

## 最关键的原理：seek，而不是播放

普通网页动画通常依赖 `requestAnimationFrame`：浏览器上一帧结束后，下一帧再推进一点时间。视频渲染需要反过来：给定目标时间，系统直接跳到目标状态。

这会带来三个约束。

### 不能读取真实时钟

以下写法在浏览器里很自然，在逐帧渲染里却没有稳定含义：

```js
const drift = performance.now() % 1000;
el.style.transform = `translateX(${drift}px)`;
```

同一帧在不同机器、不同负载下会得到不同结果。渲染器需要的是 `state = f(input, time)`，而不是 `state = f(input, wallClock)`。

### 随机数必须可复现

粒子、噪声和装饰可以使用随机分布，但随机种子必须固定，或者在初始化时根据稳定输入生成。未设种子的 `Math.random()` 会让截图对比、失败重试和缓存失去意义。

### 无限循环必须有边界

一个永不结束的 CSS 或 GSAP 循环无法告诉渲染器总时长。需要明确的 `data-duration` 或有限的重复次数；“看起来一直动”不等于“渲染器知道渲染到哪里”。

所以 HyperFrames 的动画不是“播放一段 tween”，而是“注册一套时间到状态的映射”。这也是为什么一个动画 timeline 可以早于 composition 结束，也可以超过 composition 时长：最终输出长度由 composition 的 duration contract 决定，timeline 超出的部分会被截断，timeline 提前结束则保持最后状态。

## 媒体为什么必须由框架接管

`video.currentTime` 和 `audio.currentTime` 看起来也是普通 DOM 属性，但媒体解码有自己的缓冲、关键帧和异步状态。若让业务脚本自行播放媒体，截图时可能出现“DOM 已经到第 5 秒，视频还停在第 4.8 秒”的竞态。

HyperFrames 的做法是把媒体当作有时间窗口的资源：

```html
<video
  id="broll"
  data-start="2"
  data-duration="5"
  data-media-start="13.5"
  src="broll.mp4"
></video>
```

这里的 `data-start="2"` 表示它在 composition 的第 2 秒出现；`data-media-start="13.5"` 表示出现时从源文件的 13.5 秒开始读。两者分别属于“时间线坐标”和“源媒体坐标”，不能混用。

这也解释了一个常见 lint 错误：不要给 `video` 和它的 timed ancestor 同时设置 `data-start`。如果外层 wrapper 已经移动了时间窗口，内层 video 再移动一次，渲染器在计算媒体起始位置和可见性时会得到两套不一致的坐标。

音频还有一个容易被忽略的 contract：每个 `<audio>` 都应有唯一 `id`。没有 id 的音频可能无法被 mixer 发现，最终画面正常但输出无声。这类问题不是“音频文件坏了”，而是资源没有进入框架的媒体管理边界。

## 两层结构：声明层和表现层

我更愿意把 HyperFrames composition 看成两层：

```text
声明层：HTML + data-* attributes
  解决“有什么、何时出现、从源文件哪里开始”

表现层：CSS / SVG / Canvas / GSAP / WebGL
  解决“当前时刻长什么样”
```

声明层应该尽量简单、稳定、可读。表现层可以复杂，但必须满足 seek-safe 和 deterministic 的要求。

例如，淡出字幕时应该动画字幕内部节点的 `opacity`，而不是用 GSAP 把 `.clip` 自己设置成 `visibility: hidden`。clip 的可见性由时间窗口负责；Runtime 需要知道它什么时候仍然存在，才能正确组合轨道和子 composition。

同理，CSS 的初始 `transform` 和 GSAP 对同一属性的 tween 也不应互相竞争。更稳妥的写法是把初始状态写进 `fromTo`，让唯一的动画 Runtime 成为状态来源：

```js
gsap.fromTo(
  '#card',
  { x: -40, opacity: 0 },
  { x: 0, opacity: 1, duration: 0.6 },
);
```

这不是语法偏好，而是为了避免“浏览器首次布局值”和“时间轴首帧值”在不同阶段互相覆盖。

## 子 composition 解决的是复用，不是逃避时间

当一个视频由片头、主体、解释卡片和片尾组成时，把所有 DOM 塞进一个 HTML 文件很快会失去边界。子 composition 可以把每个场景封装成独立模板，再由父级按时间组合。

但子 composition 仍然必须是完整的时间函数。父级传入实例变量、素材或起始时间后，子级的 timeline 仍然要能在任意目标时间被直接求值。不能假设“父级一定从 0 秒开始顺序播放”，否则单独预览、拖动时间轴和并行渲染都会产生差异。

一个实用的拆分标准是：如果场景有自己的素材、变量、动效和验收条件，就值得成为 composition；如果只是同一场景里的一块装饰，不要为了文件数量而拆分。

## 从原理推导出的验证顺序

HyperFrames 项目的验证应该沿着渲染链路进行，而不是只看页面能否打开。

1. **先查结构。** 根节点是否有唯一的 `data-composition-id`、尺寸和可推导的总时长？所有媒体是否有 id？
2. **再查时间。** clip 的 `data-start`、`data-duration`、相对引用和嵌套偏移是否符合预期？是否给媒体套了重复的 timed wrapper？
3. **再查 Runtime。** `window.__timelines[id]` 是否与 composition id 一致？timeline 是否 paused、有限、可 seek？
4. **再查视觉状态。** 在 0%、50%、100% 等中间点做 snapshot，而不是只看首帧和末帧。
5. **最后查媒体与声音。** 视频是否从正确的 `data-media-start` 开始，音频是否被 mixer 发现，轨道叠加和音量是否符合预期。

常用命令可以概括为：

```bash
npx hyperframes lint
npx hyperframes check
npx hyperframes snapshot --at 0,50%,100%
npx hyperframes render --output output.mp4
```

`check` 通过只说明当前 contract 和审计项没有发现问题，不等于已经完成完整的观感验收。真正交付前仍应检查代表性帧、完整播放、字体加载、媒体源和输出音轨。

## 三个看似合理、实际危险的做法

### 把 HyperFrames 当成录屏器

录屏只关心屏幕此刻显示什么；HyperFrames 需要知道每个元素为什么在这一帧出现。把所有动画塞进 `setInterval` 或依赖用户拖动，短期能出片，长期无法复现。

**修复方向：** 把时间窗口写进 `data-*`，把连续变化写进可 seek 的 Runtime。

### 用 timeline 控制视频播放

timeline 适合控制 DOM 属性，不适合替代框架对媒体时间的统一管理。视频的源时间、关键帧和音画同步有自己的 contract。

**修复方向：** 用 `data-media-start` 和 `data-duration` 描述媒体窗口，让 timeline 只做画面表现。

### 只验证“浏览器里看起来对”

交互式预览允许很多隐式状态存在：字体已经缓存、视频刚好播放到某个位置、上一段动画留下了 inline style。逐帧渲染会把这些隐式依赖暴露出来。

**修复方向：** 使用 lint、check、snapshot 和最终 render 组成验证链，并保留代表性帧作为回归样本。

## 要不要用 / 我的判断框架

值得上：视频画面本身就是 UI、网页、代码、图表、字幕或可复用组件；团队希望用 Git 管理时间线，并且需要精确到帧的渲染、批量生成或自动化验证。

可以再等：任务只是一次简单剪辑，素材和转场已经在成熟的 NLE 中完成，用 HTML 表达反而增加工程成本。

重点关注：不要先问“HyperFrames 能不能替代剪辑软件”，先问项目是否需要**可组合、可寻址、可复现的画面状态**。如果答案是肯定的，HTML 就不再只是展示层，而可以成为视频的 source of truth；如果答案是否定的，继续使用传统时间线工具通常更直接。

HyperFrames 最值得借鉴的地方，也许不是某个 CLI 命令，而是这条边界：**把结构、时间、媒体和表现分别交给最适合它们的系统，再用确定性渲染把四者重新合成一帧。**

参考资料：

- [HyperFrames Core Documentation](https://github.com/heygen-com/hyperframes/blob/main/packages/core/docs/core.md)
- [HyperFrames Core Skill](https://github.com/heygen-com/hyperframes/blob/main/skills/hyperframes-core/SKILL.md)
- [Data Attributes Reference](https://github.com/heygen-com/hyperframes/blob/main/skills/hyperframes-core/references/data-attributes.md)

> 本文使用 [writting-skill](https://github.com/zisheng-ai/writting-skill) 辅助写作。项目已开源，欢迎在 GitHub 点个 Star。
