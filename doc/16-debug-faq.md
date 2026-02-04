# 16. 调试清单与常见问题（FAQ）

这份清单尽量按“症状 → 排查路径 → 关键文件”组织，适合你边跑边断点。

## 1. 一键排查清单（从外到内）

### 1) 平台是否选对？

- Web 项目是否误导入了 Node/Worker 包，或在 SSR 环境导入了 `leafer-ui/@leafer-ui/web`？
  - Web 平台入口会访问 `window/document/navigator/Image`（见 `packages/platform/web/src/core.ts`）
  - SSR/Node 下应使用 `@leafer-ui/node`，Worker 下应使用 `@leafer-ui/worker`

### 2) 平台是否完成初始化？

如果你不是从 `leafer-ui` 根包导入，而是导入了某个底层包（例如 `@leafer-ui/draw`），要确认：

- 是否执行过 `useCanvas(...)`（平台初始化）
  - Web：`packages/platform/web/src/index.ts` 会自动 `useCanvas('canvas')`
  - Node：需要显式 `useCanvas(canvasType, power)`（见 `packages/platform/node/src/core.ts`）

### 3) external 能力是否注入？

如果渲染里涉及渐变/图片/阴影/滤镜等，通常依赖 `@leafer-ui/external` 的实现：

- 默认注入发生在 `@leafer-ui/partner`（`packages/partner/partner/src/index.ts`）
- 如果你只导入了 `@leafer-ui/draw` 而没有导入 partner，`Paint/Effect/Filter` 可能还是空对象（壳）

### 4) hit/pick 是否启用？

命中检测与拾取依赖副作用注入（prototype 扩展）：

- `@leafer-ui/hit` 入口：`packages/hit/src/index.ts`

平台聚合入口（如 `@leafer-ui/web` → `@leafer-ui/core`）通常会把它带上；但如果你“按包自行拼装”，可能会漏掉。

### 5) 交互是否真的在监听？

Web 端交互监听在：

- `packages/interaction/interaction-web/src/Interaction.ts` 的 `__listenEvents()`

如果你创建了 `Leafer`，但 `this.interaction` 为 `undefined`，通常意味着：

- `Creator.interaction` 没被平台装配（没走 platform 包）
- 或 config 禁用了交互相关能力（需要结合 `@leafer/core` 的 config 约定排查）

## 2. 常见问题（症状 → 原因 → 去哪看）

### Q1：画布啥也不显示 / 只显示一帧就不更新

优先检查：

- `Leafer.init()` 是否创建了 `renderer/watcher/layouter`：
  - `packages/display/src/Leafer.ts`
- 是否调用了 `start()`（默认 config.start=true）：
  - `packages/display/src/Leafer.ts#start`
- 是否触发了 `renderer.update()/render()`（具体在 `@leafer/core`，本仓库只看到调用点）

### Q2：fill/stroke 设置为对象（渐变/图片）后不生效

常见原因：

- `UIData` 标记了 `__needComputePaint`，但 `Paint.compute(...)` 没有真实实现（external 未注入）
  - 入口：`packages/display-module/data/src/UIData.ts#__computePaint`
  - 注入：`packages/partner/partner/src/index.ts`
  - 实现：`packages/partner/paint/src/index.ts`、`packages/partner/image/src/index.ts`、`packages/partner/gradient/src/index.ts`

### Q3：阴影/滤镜渲染被裁剪（边缘缺一圈）

优先检查 render bounds 的 spread 计算：

- `packages/display-module/bounds/src/UIBounds.ts#__updateRenderSpread`
  - 是否正确计算 shadow/blur/filter/stroke/renderSpread
  - 注意它依赖 `Effect.getShadowRenderSpread`、`Filter.getSpread`

### Q4：点击/hover 命中不准，透明 PNG 点不到 / 点到边缘外

这是 hitFill/hitStroke 策略导致的常见现象：

- pixel hit：精确但更耗性能（依赖 hitCanvas + getImageData）
- path hit：快但不看纹理 alpha

相关代码：

- `packages/hit/src/UIHit.ts`：决定走 pixel hit 还是 path hit，并更新 hitCanvas
- `packages/hit/src/canvas.ts`：`hitPixel` 的实现（getImageData）

### Q5：事件触发顺序怪 / capture/bubble 不符合预期

事件派发逻辑在：

- `packages/interaction/interaction/src/emit.ts`

它明确分两段：

1) capture：从根到目标（反向遍历 path）
2) bubble：从目标到根（正向遍历 path）

并且 bubble 阶段会尝试调用：

- `State.updateEventStyle(leaf, type)`（如果存在）用于 hover/press 等状态样式更新

### Q6：Web 端滚轮缩放/手势缩放没反应

先确认事件是否监听到：

- `packages/interaction/interaction-web/src/Interaction.ts`：
  - `onWheel`
  - `gesturestart/gesturechange/gestureend`

再确认 `InteractionBase` 的 transform 接口是否被某个插件重写：

- `packages/interaction/interaction/src/Interaction.ts` 中 `move/zoom/rotate/wheel/multiTouch` 默认是空实现（注释提示 `@leafer-in/viewport will rewrite`）

结论：仅靠 `interaction` 本体，不一定会提供“视图缩放/旋转”的业务行为，它通常由 viewport/editor 等插件完成。

### Q7：小程序里事件不触发

小程序交互是“桥接接收”模式：

- `@leafer-ui/miniapp` 会把 `Leafer.prototype.receiveEvent` 指向 `this.interaction.receive(event)`（见 `packages/platform/miniapp/src/index.ts`）
- `@leafer-ui/interaction-miniapp` 的 `Interaction.receive(e)` 再把事件分发到 `onTouchStart/Move/End`

因此你需要确认：

1) 平台是否初始化成功（`useCanvas('miniapp', wx)`）
2) 宿主是否在把触摸事件喂给 `leafer.receiveEvent(rawEvent)`

## 3. 常用源码定位命令（阅读效率翻倍）

你可以在仓库根目录用这些搜索快速定位入口：

- 平台初始化：搜索 `useCanvas(`、`Platform.origin`  
  - 例如：`rg "useCanvas\\(" packages/platform -n`
- external 注入：搜索 `Object.assign(`（partner）  
  - 例如：`rg "Object\\.assign\\(Paint|Object\\.assign\\(Effect" packages/partner -n`
- hit 副作用：搜索 `import './`（hit/index.ts）  
  - 例如：`rg "import '\\./" packages/hit/src/index.ts -n`
- 模块注入：搜索 `@useModule(`  
  - 例如：`rg "@useModule\\(" packages/display -n`

