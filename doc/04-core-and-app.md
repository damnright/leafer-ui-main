# 04. 模块：`@leafer-ui/draw` / `@leafer-ui/core` / `@leafer-ui/app`（聚合层）

这三个包的共同点：**它们不是“单一能力实现”，而是把多个包组合成一个更好用的入口**。理解它们能帮你快速定位代码分层、也能理解平台包为什么只要装配 `Creator/Platform` 就能跑起来。

## 1. 三者的关系（一句话版）

- `@leafer-ui/draw`：UI 显示对象与绘制相关的“基础集合”（不含平台适配、不含 web 交互实现）
- `@leafer-ui/app`：在 `Leafer` 之上提供“多画布/多层”应用封装（`App`）
- `@leafer-ui/core`：UI 栈的“核心大集合”（= draw + app + interaction + event + hit）

平台包（如 `@leafer-ui/web`）通常会：

1) 初始化平台（`useCanvas(...)` 设置 `Platform` + 给 `Creator` 注入 canvas/image 实现）
2) 直接 re-export `@leafer-ui/core`（让上层拿到 UI 栈所有能力）
3) 装配平台交互实现（如 Web 的 `Interaction`）
4) 注入可插拔模块（`@leafer-ui/partner`）

## 2. `@leafer-ui/draw`：UI “绘制/显示对象”基础集合

### 2.1 入口文件

- `packages/core/draw/src/index.ts`

它做的事情很“纯粹”：把下面这些包再导出到一个入口里：

- `@leafer/core`（核心引擎能力：Leaf/数据/布局/渲染框架、Creator/Platform、工具库）
- `@leafer-ui/display`（UI 显示对象：Leafer/Group/Rect/Text/Image…）
- `@leafer-ui/display-module`（Data/Bounds/Render 等模块化能力）
- `@leafer-ui/decorator`（UI 属性装饰器：effectType/zoomLayerType 等）
- `@leafer-ui/external`（可插拔能力的“占位符”：Paint/Effect/Filter/Export/State/Transition…）

### 2.2 你在读源码时怎么用它？

当你想从“最底层的 UI 对象实现”开始读，不关心交互/事件/命中检测时，就从 `@leafer-ui/draw` 这条线看：

- `packages/display/src/UI.ts`：UI 基类与属性体系（最关键）
- `packages/display-module/*`：Data/Bounds/Render 模块（UI 的核心机制）

### 2.3 一个重要提醒：`external` 默认是“壳”

`@leafer-ui/draw` 会导出 `@leafer-ui/external`，但其中像 `Paint/Effect/Filter` 多数是占位符或薄封装：

- 真正的实现通常由 `@leafer-ui/partner` 在运行时注入（`Object.assign(...)`）
- 或者通过 `Plugin.need('xxx')` 触发按需加载（在更完整的集成环境中）

所以：**只看 draw，不等于所有能力都已经“有实现”**。

## 3. `@leafer-ui/app`：应用封装（`App`）

### 3.1 入口文件

- `packages/app/src/index.ts`：只导出 `App`
- `packages/app/src/App.ts`：核心实现

### 3.2 `App` 与 `Leafer` 的区别

`Leafer`（`packages/display/src/Leafer.ts`）更像“单一画布上的根节点”，负责：

- 初始化 canvas / renderer / watcher / layouter
- 管理 selector / interaction / canvasManager / hitCanvasManager
- 驱动渲染与布局生命周期

`App`（`packages/app/src/App.ts`）则在 `Leafer` 之上提供“多 Leafer 分层”的组织方式：

- `ground` / `tree` / `sky`：常见的三层结构（背景层、主要内容层、前景/工具层）
- `addLeafer()`：向 app 中增加新的子 leafer（并把子 leafer 的 view/canvas 与 app 的 canvas 关系处理好）
- `start/stop/resize/forceRender/updateLayout`：统一转发到各子 leafer

如果你做“编辑器/组态/白板”这类产品，多层结构通常是刚需，`App` 就是为这类场景准备的。

## 4. `@leafer-ui/core`：UI 栈核心聚合（最常见的内部依赖）

### 4.1 入口文件

- `packages/core/core/src/index.ts`

它导出：

- `@leafer-ui/draw`
- `@leafer-ui/app`
- `@leafer-ui/interaction`（跨平台交互基础：InteractionBase/Dragger/Cursor…）
- `@leafer-ui/event`（UI 事件类型：Pointer/Touch/Drag/Zoom/Key…）
- `@leafer-ui/hit`（命中检测：HitCanvasManager、hit/pick 逻辑与原型扩展）

### 4.2 它在整个系统里的位置

你可以把 `@leafer-ui/core` 理解为：“平台无关的 UI 栈”。它不负责 Web/Node/Worker/小程序的 canvas 实现与事件监听，但它包含了 UI 运行所需的大部分逻辑。

平台包（如 `@leafer-ui/web`）会在其基础上：

- 选择具体平台的 `canvas/image` 实现并装配到 `Creator`
- 选择具体平台的交互实现（`interaction-web` / `interaction-miniapp`）
- 初始化 `Platform.origin`（平台 IO / 图片加载 / 导出能力）

## 5. 推荐阅读顺序（从聚合层快速下钻）

1. `packages/core/core/src/index.ts`：看 core 聚合了哪些能力
2. `packages/core/draw/src/index.ts`：看 draw 聚合了哪些“显示/绘制相关”能力
3. `packages/display/src/Leafer.ts`：看运行时如何 init（Creator/Platform 在这里被使用）
4. `packages/display/src/UI.ts`：看 UI 属性、数据、模块、重写机制（核心中的核心）
5. 继续深入：`packages/display-module/*`、`packages/event/*`、`packages/interaction/*`、`packages/hit/*`

