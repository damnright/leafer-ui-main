# 03. 模块：`@leafer-ui/web`（Web 运行时装配）

> 结论先行：`leafer-ui`（根包）本质上就是 `@leafer-ui/web` 的再导出；因此**你在浏览器里 `import 'leafer-ui'` 时，默认就走 Web 平台的运行时初始化逻辑**。

## 1. 这个模块负责什么？

`@leafer-ui/web` 的职责是把“UI 栈”在 **Web 端** 组装成可用的运行时：

- 选择并初始化 Web 平台的 `Canvas/Image` 实现（来自 `@leafer/canvas-web`、`@leafer/image-web`）
- 设置 `Platform`（来自 `@leafer/core`）的 Web 端能力：创建 canvas、下载、加载图片、事件 stop 等
- 扩展 `Creator`（来自 `@leafer/core`）的工厂方法：`canvas`、`image`、`interaction`、`hitCanvas`、`hitCanvasManager`
- 选择 Web 端交互实现：`@leafer-ui/interaction-web`
- 注入 UI 的可插拔能力模块：`@leafer-ui/partner`

一句话：它把“抽象的核心 + UI 功能 + Web 平台实现”拼装在一起，让上层可以直接 new `Leafer/App` 并渲染、交互。

## 2. 入口文件与导出内容

### 2.1 入口：`packages/platform/web/src/index.ts`

这个文件做了三类事情：

1) **聚合导出（re-export）**：让用户从 `@leafer-ui/web` 一次性拿到常用类型与能力

- `export * from '@leafer-ui/interface'`（UI 层接口/类型）
- `export * from './core'`（Web 平台的 `useCanvas` + 导出 core/canvas-web/image-web）
- `export * from '@leafer/partner'`（Leafer 核心生态的 partner 能力）
- `export * from '@leafer-ui/core'`（UI 核心聚合：draw/app/interaction/event/hit）
- `export * from '@leafer-ui/interaction-web'`（Web 端交互实现）
- `export * from '@leafer-ui/partner'`（UI 可插拔能力注入）

2) **扩展 Creator（工厂装配）**：把 Web 端交互、命中画布等能力挂到 `Creator` 上

- `Creator.interaction(...) => new Interaction(...)`
- `Creator.hitCanvas(...) => new LeaferCanvas(...)`
- `Creator.hitCanvasManager() => new HitCanvasManager()`

3) **执行平台初始化的副作用**：最后一行 `useCanvas('canvas')` 会立刻把 `Platform` 配置成 Web 环境。

这也是为什么：在浏览器里“只要 import 一下 `leafer-ui`”通常就能直接跑起来（平台已经被设置好了）。

### 2.2 平台核心：`packages/platform/web/src/core.ts`

`core.ts` 的角色是：提供 Web 平台的 `Platform.origin` 实现 + `Creator.canvas/image` 的具体实现。

关键点：

- `Object.assign(Creator, { canvas: () => new LeaferCanvas(), image: () => new LeaferImage() })`
- `useCanvas('canvas')` 内部设置：
  - `Platform.origin.createCanvas()`：用 DOM `document.createElement('canvas')`
  - `Platform.origin.canvasToDataURL/canvasToBlob/canvasSaveAs/download()`：用于导出与下载
  - `Platform.origin.loadImage()`：基于 `new Image()` 加载
  - `Platform.event.stopDefault/stop/stopNow()`：事件控制
  - `Platform.canvas = Creator.canvas()`：提供一个默认 canvas 实例用于能力探测
  - `Platform.conicGradientSupport`、UA/OS 特性开关、`devicePixelRatio` 等

## 3. 典型运行链路（导入 → 初始化 → 创建实例）

以根包 `leafer-ui` 为例（`src/index.ts` 只是转导出 `@leafer-ui/web`）：

1. 业务代码 `import { Leafer, Rect } from 'leafer-ui'`
2. 实际加载到 `@leafer-ui/web`：
   - 执行 `useCanvas('canvas')`，完成 `Platform` 的 Web 端初始化
   - 执行 `Object.assign(Creator, {...})`，装配工厂
3. 业务创建 `new Leafer({ view: ... })`（在 `packages/display/src/Leafer.ts`）：
   - 调用 `Creator.canvas/config` 创建画布与渲染器/布局器/观察器
   - 调用 `Creator.selector`、`Creator.interaction` 创建选择器与交互实例（这里的 `interaction` 就是 web 装配进去的）
   - 创建 `HitCanvasManager` 用于命中检测的画布缓存

## 4. 阅读与调试建议（建议断点位置）

想快速摸清“为什么 import 后就能用”，建议从这些点下断点：

- `packages/platform/web/src/index.ts`：看 `Creator` 被扩展、`useCanvas('canvas')` 被调用
- `packages/platform/web/src/core.ts`：看 `Platform.origin` 是怎么被设置的
- `packages/display/src/Leafer.ts` 的 `init()`：看 `Creator.canvas/renderer/watcher/layouter/selector/interaction` 的创建顺序

## 5. 常见坑与注意事项

- **不要在非浏览器环境导入 `leafer-ui` / `@leafer-ui/web`**：`core.ts` 直接访问 `window/document/navigator/Image`，SSR/Node 会报错。非 Web 环境应使用对应平台包（例如 `@leafer-ui/node` / `@leafer-ui/worker` / `@leafer-ui/miniapp`）。
- **副作用初始化是“导入即执行”**：`useCanvas('canvas')` 会在模块加载时运行；如果你在同一进程里想切换平台（不常见），需要非常谨慎地控制导入顺序与全局状态。

