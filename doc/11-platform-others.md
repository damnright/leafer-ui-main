# 11. 模块：其他平台入口（Node / Worker / Miniapp / CanvasKit）

本仓库的“平台适配”统一放在 `packages/platform/*`。Web 平台我们已在 `doc/03-platform-web.md` 单独说明；本篇补齐其余平台包的源码阅读与使用要点。

> 注意：这些包的 `package.json` 往往把 `main/exports` 指向 `dist/` 构建产物，但当前工作区没有这些产物目录；阅读源码时以 `src/` 为准。

## 1. 一张对比表：它们的核心差异

| 平台包 | Canvas 实现 | 图片实现 | 交互模型 | 是否“导入即初始化” |
|---|---|---|---|---|
| `@leafer-ui/worker` | `@leafer/canvas-worker`（OffscreenCanvas） | `@leafer/image-worker`（ImageBitmap） | 使用 `InteractionBase`（无 DOM 监听） | 是（`useCanvas('canvas')`） |
| `@leafer-ui/node` | `@leafer/canvas-node`（Skia/N-API） | `@leafer/image-node` | 使用 `InteractionBase`（无 DOM 监听） | 否（需要显式 `useCanvas(...)` 并传入 power） |
| `@leafer-ui/miniapp` | `@leafer/canvas-miniapp`（小程序 offscreen canvas） | `@leafer/image-miniapp` | `Interaction` 通过 `receiveEvent` 桥接 | “通常是”（尝试用全局 `wx` 调用 `useCanvas('miniapp', wx)`） |
| `@leafer-ui/canvaskit` | （当前快照无源码实现） | （同上） | （同上） | 否（`src/index.ts` 为空） |

## 2. `@leafer-ui/worker`（Worker 运行时）

### 2.1 入口文件

- `packages/platform/worker/src/index.ts`
- `packages/platform/worker/src/core.ts`

### 2.2 它做了什么？

1) 在 `core.ts` 里装配 `Creator.canvas/image`：

- `Creator.canvas(...) => new LeaferCanvas(...)`（worker 版）
- `Creator.image(...) => new LeaferImage(...)`

2) `useCanvas(...)` 设置 `Platform.origin`（核心点）：

- `createCanvas`：`new OffscreenCanvas(width, height)`
- `canvasToDataURL`：通过 `convertToBlob` + `FileReader` 转 dataURL
- `loadImage`：XHR 拉取 blob → `createImageBitmap`
- `download/canvasSaveAs`：在 worker 环境不可用，提供空实现/resolve

3) `index.ts` 里扩展 `Creator`：

- `Creator.interaction(...) => new InteractionBase(...)`
- `Creator.hitCanvas(...) => new LeaferCanvas(...)`
- `Creator.hitCanvasManager() => new HitCanvasManager()`

4) `index.ts` 最后调用 `useCanvas('canvas')`：导入即初始化。

### 2.3 阅读/调试建议

- 平台能力：`packages/platform/worker/src/core.ts` 的 `Platform.origin`
- 为什么没有 DOM 事件：看 `packages/platform/worker/src/index.ts` 使用的是 `InteractionBase`

## 3. `@leafer-ui/node`（Node 运行时）

### 3.1 入口文件

- `packages/platform/node/src/index.ts`
- `packages/platform/node/src/core.ts`

### 3.2 关键差异：Node 需要显式 `useCanvas(...)`

在 `core.ts` 中，`useCanvas(canvasType, power)` 会根据 `canvasType` 选择两种模式：

- `canvasType === 'skia'`：使用 `power.Canvas` + `power.loadImage`，并启用 `Platform.roundRectPatch`
- `canvasType === 'napi'`：类似，但 dataURL/buffer/safeAs 走不同 API，保存文件会用 `fs.writeFileSync`

同时它会设置：

- `Platform.name = 'node'`
- `Platform.backgrounder = true`
- `Platform.requestRender = setTimeout(render, 16)`
- `devicePixelRatio = 1`（Node 默认不依赖屏幕 DPR）

### 3.3 `index.ts` 的装配内容

与 worker 类似，Node 端也把 `Creator.interaction` 设为 `InteractionBase`（没有平台监听），并提供 hitCanvas/hitCanvasManager。

此外 Node 入口还 `export * from '@leafer-in/export'`（见 `packages/platform/node/src/index.ts`），用于导出相关能力（需要更完整的插件/构建环境配合）。

### 3.4 你在 Node 端需要自己解决什么？

最重要的是：提供 `power`（Canvas 构造与 loadImage 实现）。这决定了 Node 端具体用哪种 Skia/N-API Canvas 后端。

## 4. `@leafer-ui/miniapp`（小程序/小游戏运行时）

### 4.1 入口文件

- `packages/platform/miniapp/src/index.ts`
- `packages/platform/miniapp/src/core.ts`

### 4.2 平台初始化（`useCanvas('miniapp', wx)`）

`core.ts` 的 `useCanvas(..., app)` 会设置：

- `Platform.origin.createCanvas`：优先 `app.createOffscreenCanvas`，否则兼容 `createOffScreenCanvas`（注意注释里提到的大小写坑）
- `canvasSaveAs/download`：写入用户数据目录，必要时保存到相册
- `loadImage`：通过 `Platform.canvas.view.createImage()`
- `Platform.miniapp`：封装 `select/getBounds/getSizeView/saveToAlbum/onWindowResize` 等宿主能力
- `devicePixelRatio`：从 `wx.getWindowInfo()/getSystemInfoSync()` 获取
- `Platform.requestRender`：优先 `view.requestAnimationFrame`，否则 `setTimeout`（兼容抖音小程序）

### 4.3 交互桥接：`receiveEvent`

`packages/platform/miniapp/src/index.ts` 做了一个关键桥接：

- `Leafer.prototype.receiveEvent = function(event){ this.interaction && this.interaction.receive(event) }`

配合 `@leafer-ui/interaction-miniapp`（见 `doc/08-interaction.md`），宿主只要把触摸事件喂给 `leafer.receiveEvent(rawEvent)`，就能进入 `InteractionBase` 的派发体系。

### 4.4 “导入即初始化”的行为

`index.ts` 末尾：

- `try { if (wx) useCanvas('miniapp', wx) } catch { }`

也就是说：如果运行环境里存在全局 `wx`，导入就会自动初始化平台；否则需要你手动调用 `useCanvas(...)`。

## 5. `@leafer-ui/canvaskit`

当前工作区 `packages/platform/canvaskit/src/index.ts` 为空文件。通常这意味着：

- 该平台实现主要依赖构建产物（`dist/`）或在别的仓库/分支中维护
- 在本仓库快照中只能看到包壳（`package.json`、`README.md`），无法进一步从源码层面解读

