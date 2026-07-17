# HyperFrames 渲染后端 · 选型边界与操作手册

> 2026-07-17 实测验证通过后引入（工具链/中文字体/代理环境/迁移/3D 五项全过，关键数据已内嵌本文）。
> HyperFrames 是 HeyGen 开源的 HTML→视频框架（Apache 2.0）：纯 HTML + 暂停的 GSAP timeline，headless 浏览器逐帧 seek 确定性渲染。

## 选型边界（先看这张表再开工）

| 场景 | 用哪条渲染路线 |
|---|---|
| 新动画项目（默认） | **HyperFrames**。审计套件白送、3D/GSAP/Lottie/shader 全解锁 |
| 需要 3D / 粒子 / 物理惯性 / shader 转场 | HyperFrames（自研 Stage 做不到） |
| 老 Stage demo 要复用/改版 | 顺手迁移（适配器配方见下，20-30 分钟/个）；只重渲不改就仍用 render-video-seek.js |
| 弱 runtime（无 npm / 无法装依赖 / 单文件交付给用户双击打开） | 自研 Stage（assets/animations.jsx），老流程不变 |
| 交互演示（用户要在浏览器里玩，不导出视频） | 自研 Stage 或普通 HTML，HyperFrames 是渲染管线不是交互框架 |
| 批量参数化视频（千人千面/模板换字） | Remotion（见规划方向5，独立于本 skill 主流程） |

**设计语言永远是甲方**：叙事结构、easing 体系、SFX/BGM 双轨制照旧全部生效（animation-best-practices.md / audio-design-rules.md），HyperFrames 只是实现和渲染工具。GSAP 实现配方见 `references/gsap-recipes.md`。

## 项目脚手架

```bash
npx -y hyperframes init 项目名 --example blank   # 非交互必须带 --example
cd 项目名 && npm install
```

生成 index.html / hyperframes.json / meta.json / package.json（pin 了 CLI 版本）+ 项目级 CLAUDE.md。init 会把 19 个 hyperframes skill 装到 `~/.claude/skills/`（本机已装）。合成写法契约读 `~/.claude/skills/hyperframes-core/SKILL.md`，本地文档 `npx hyperframes docs <topic>`（data-attributes / gsap / rendering / troubleshooting）。

**版本策略**：项目 package.json 会 pin 精确版本（当前实测过的是 0.7.61）。它迭代极快（300+ releases），升级先 `npx hyperframes@latest upgrade --project . --check` 看 delta，跑一遍回归 demo 再动。

## 合成契约速查（完整版读 hyperframes-core）

- 根容器：`data-composition-id` + `data-start` + `data-duration` + `data-width/height`
- 每个计时元素：`class="clip"` + `data-start` + `data-duration` + `data-track-index`
- timeline 必须 paused 并注册：`window.__timelines["合成id"] = gsap.timeline({paused:true})`
- 视频素材用 `muted`，音轨单独 `<audio>` 元素
- **只允许确定性逻辑**：禁 `Date.now()` / `Math.random()` / 运行时网络 fetch；随机用种子函数
- 字体：Google Fonts 会被编译器自动抓取并注入确定性 @font-face（缓存 `~/.cache/hyperframes/fonts/`）；纯系统字体（PingFang SC 等）加一行 `@font-face { font-family:"PingFang SC"; src: local("PingFang SC"); }` 过 lint
- Three.js 走 `hf-seek` 事件适配器（`~/.claude/skills/hyperframes-animation/adapters/three.md`），根容器必须显式 `data-duration`

## 老 demo 迁移 · 适配器配方（实测 20-30 分钟/个）

自研 Stage/纯 render(t) 动画不用重写，四步：

1. **包容器**：外套 `#root` 带合成 data 属性；整个 `.stage` 作为唯一 clip 最省事（`class="stage clip"` + data-start/duration/track-index）；`.stage` 从 fixed 居中改 absolute inset:0，html/body 定死 1920×1080
2. **删自驱**：rAF tick 循环、fitStage/resize 监听、replay 按钮、`__ready/__setTime/__seek` 协议全删（渲染器不需要）
3. **挂代理 tween**（核心 12 行）：
   ```js
   const proxy = { t: 0 };
   const tl = gsap.timeline({ paused: true });
   tl.to(proxy, { t: DURATION, duration: DURATION, ease: "none",
     onUpdate: () => render(proxy.t) }, 0);
   window.__timelines = window.__timelines || {};
   window.__timelines["main"] = tl;
   render(0);   // 必须：timeline 停在 t=0 时 onUpdate 不触发，不补这句首帧可能未初始化
   ```
4. **扫 transition**：全文搜 `transition:` 声明。CSS transition + class 切换走墙钟，逐帧 seek 下不确定，必须改成 render(t) 里对 t 的纯函数（lerp）

## 校验与渲染

```bash
npm run check                        # lint+runtime+layout+motion+contrast 五门审计
npx hyperframes check --no-contrast  # 暗色电影风专用（见下）
npx -y hyperframes@<pin版本> render --fps 60   # 终渲；默认 30fps
```

- **check 必须 0 error 才渲染**（contrast 门除外）。lint 能拦 letterSpacing 抖动、字体缺失、非确定性等一整类「无报警视觉 bug」
- **contrast 门取舍**：它按 WCAG 4.5:1 检查，和暗色电影风的低对比水印/装饰文字（16-40% 透明度）根本冲突，且无逐元素豁免。暗色 cinematic 产出统一 `--no-contrast`，其余四门仍必须 0 error。亮底信息型产出不要跳，contrast 报错通常是真问题
- **两级渲染**：先默认 30fps 快速出片，肉眼+截帧检查通过后再 `--fps 60` 终渲。60fps 600 帧 1080p 实测约 20 秒
- 渲染产物侧校验（audio stream / 黑帧 / 响度 / 时长）用 `scripts/verify-video.sh`（见 verification.md）

## 音频

HyperFrames 合成里 `<audio>` 元素可直接进时间轴（BGM/解说随片渲染）。当前音频流程不变：SFX/BGM 双轨制照 audio-design-rules.md，用 add-music.sh / mix-voiceover.sh 后期混流也可以。哪条路更好在实战中定，先不强制。

## pitfalls 增量（相对自研管线）

自研管线 pitfalls（animation-pitfalls.md §7/10/12/13 录制协议类、§6 字体时序、§15/17 网络类）在 HyperFrames 后端上**不适用**：录制协议由框架内部处理，字体编译期抓取，CDN 实测代理下可通。新增的坑只有三条，已录入 animation-pitfalls.md §18-20：CSS transition 非确定性、代理 tween 首帧、contrast 门冲突。
