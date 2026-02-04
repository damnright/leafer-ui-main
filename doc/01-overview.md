# 01. 总览：这是什么项目？如何开始读/用源码？

## 1. 项目定位

`leafer-ui` 是 LeaferJS 生态中的 UI/交互层：在 `@leafer/core` 的基础上，提供一套面向“画布 UI（Canvas UI）”的组件与能力，包括：

- UI 显示对象（`Rect`、`Text`、`Image`、`Group`、`Frame` 等）
- 数据层与渲染层的模块化拆分（Data / Bounds / Render）
- 交互事件与手势体系（拖拽、缩放、旋转、滑动、键盘等）
- 命中检测（HitTest / Pick）
- 跨平台运行时适配（Web / Worker / Node / 小程序等）
- 可插拔的“能力模块”（如 Paint、Gradient、Effect、TextConvert、Export 等）

## 2. 本仓库里有什么（你现在看到的这份源码）

你当前打开的是 `leafer-ui` 的源码仓库（或其裁剪版本），主要特征：

- 根包 `leafer-ui` 只做“入口聚合”：`src/index.ts` 直接 `export * from '@leafer-ui/web'`
- 主要源码分布在 `packages/**/src`
- 许多 `package.json` 的 `main/exports/types` 指向 `dist/`、`lib/`、`types/` 等发布产物，但这些目录在当前工作区并不存在（通常由外部构建流程生成）

因此，这份仓库更适合用于：

- 阅读源码、理解分层与模块边界
- 在“更完整的主集成仓库”里联调（它会提供 workspace/构建脚本/依赖拉取方式）

## 3. 代码分层与核心包关系（从上到下）

下面用“从最终入口到基础能力”的方式描述依赖关系（有助于你建立心智模型）：

### 3.1 最终入口：`leafer-ui` → `@leafer-ui/web`

- 根目录 `src/index.ts`：只做一件事 —— 把 `@leafer-ui/web` 的能力整体 re-export。
- `@leafer-ui/web`（`packages/platform/web`）：
  - 组装 `Creator`（工厂/创建器）与 `Platform`（运行时平台能力）
  - 装配 Web 端交互实现 `@leafer-ui/interaction-web`
  - 装配 UI 核心能力 `@leafer-ui/core`
  - 装配“可插拔能力模块” `@leafer-ui/partner`

### 3.2 UI 核心聚合：`@leafer-ui/core`

`@leafer-ui/core`（`packages/core/core`）是“UI 核心能力集合”，其 `src/index.ts` 可以理解为：

- `draw`（绘制与显示对象层）
- `app`（应用层封装）
- `interaction`（交互基础）
- `event`（事件类型与工具）
- `hit`（命中检测）

### 3.3 绘制与显示对象层：`@leafer-ui/draw` / `@leafer-ui/display`

- `@leafer-ui/display`（`packages/display`）提供最直观的 UI 显示对象：`Leafer`、`Group`、`Rect`、`Text`、`Image` 等。
- `@leafer-ui/display-module`（`packages/display-module/*`）把显示对象相关能力拆成可组合模块（如 Data、Bounds、Render）。
- `@leafer-ui/draw`（`packages/core/draw`）负责把 `@leafer/core` + `display` + `display-module` + `decorator` + `external` 聚合到一起，形成“绘制层入口”。

### 3.4 交互与事件：`@leafer-ui/interaction*` / `@leafer-ui/event`

- `@leafer-ui/event`（`packages/event`）定义 UI 事件类型（Pointer/Touch/Drag/Zoom/Rotate/Key 等）以及相关工具类。
- `@leafer-ui/interaction`（`packages/interaction/interaction`）提供跨平台的交互基础实现（如 `Cursor`、`Dragger`）。
- `@leafer-ui/interaction-web`、`@leafer-ui/interaction-miniapp` 提供平台相关实现（Web 指针事件、小程序事件桥接等）。

### 3.5 命中检测：`@leafer-ui/hit`

- `@leafer-ui/hit`（`packages/hit`）提供拾取（pick）与命中检测相关的实现与注册逻辑（从 `src/index.ts` 的一系列 `import './xxx'` 可以看到它通过副作用完成模块挂载/注册）。

### 3.6 可插拔能力：`@leafer-ui/external` + `@leafer-ui/partner/*`

这一层是理解 Leafer UI 的关键设计之一：

- `@leafer-ui/external`（`packages/external`）导出一组“能力模块占位符”（例如 `Paint`、`Effect`、`TextConvert`、`Export`、`State`、`Transition` 等）。这些对象本身大多是空实现/壳，用于在运行时被“真正实现”覆盖。
- `@leafer-ui/partner`（`packages/partner/partner`）会把各个能力模块（如 `@leafer-ui/paint`、`@leafer-ui/gradient`、`@leafer-ui/effect`、`@leafer-ui/text`、`@leafer-ui/color` 等）通过 `Object.assign(...)` 注入到上述占位符中，实现“可替换/可裁剪/可按需加载”的效果。

## 4. 从哪里开始读源码（推荐路线）

如果你的目标是“快速理解整体架构 + 能跑起来”，推荐顺序：

1. `src/index.ts`：确认根包入口指向 `@leafer-ui/web`
2. `packages/platform/web/src/index.ts`：看 Web 端是如何装配 `Creator` / `Interaction` / `HitCanvasManager`
3. `packages/core/core/src/index.ts` 与 `packages/core/draw/src/index.ts`：看核心聚合层的边界
4. `packages/display/src/index.ts`：看提供了哪些 UI 对象（Rect/Text/Image/Group/Frame/Leafer…）
5. `packages/partner/partner/src/index.ts` + `packages/external/src/index.ts`：理解“占位符 + 注入”的插件化机制

## 5. 如何“使用源码”（两种常见方式）

> 由于当前仓库缺少 workspace/构建脚本/产物目录，你更可能在“消费方工程”或“主集成仓库”里使用它。

### 5.1 方式 A：作为 npm 包使用（业务接入最快）

当你只想在业务项目里使用 Leafer UI，优先选择安装发布包：

- 安装：`npm i leafer-ui`（或安装全量包 `leafer`）
- 使用：在 Web 环境通常直接从 `leafer-ui` 入口引入（它会带上 `@leafer-ui/web` 的平台适配）

最小示例（仅用于理解入口，不作为 API 最终权威，以官网指南为准）：

```ts
import { Leafer, Rect } from 'leafer-ui'

const leafer = new Leafer({ view: 'canvas' })
leafer.add(new Rect({ x: 50, y: 50, width: 100, height: 100, fill: '#4e7' }))
```

### 5.2 方式 B：在主集成仓库中联调（贡献/深度调试推荐）

当你要改源码、追调用链、调试跨包依赖时，建议在官方“主集成仓库”中进行（它通常会把 `leafer`、`leafer-ui`、`leafer-in` 等放入同一 workspace，并提供构建/示例/调试脚本）。

你可以把本仓库理解为“UI 侧源码包集合”，而不是一个完整可运行的应用工程。

