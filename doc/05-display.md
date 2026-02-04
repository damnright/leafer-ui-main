# 05. 模块：`@leafer-ui/display`（UI 显示对象层）

`@leafer-ui/display` 是你“看得见摸得着”的 UI 对象集合：`Leafer / Group / Box / Rect / Text / Image ...`。它把 UI 树的结构、属性体系、布局/渲染生命周期的核心入口都放在这里。

> 建议你把它理解为：在 `@leafer/core` 的 `Leaf` 体系之上，构建的一套 Canvas UI 组件模型。

## 1. 入口与导出

- 入口：`packages/display/src/index.ts`
- 主要导出：
  - `UI`（基类）
  - `Leafer`（根节点/运行时实例）
  - `Group`（分支节点）
  - `Box` / `Frame`（常用容器）
  - `Rect/Ellipse/Polygon/Star/Line/Path/Pen/Text/Image/Canvas` 等基础图形

## 2. 你必须先理解的三个核心概念

### 2.1 `UI` 是“所有图形”的基类（最关键）

文件：`packages/display/src/UI.ts`

`UI` 的职责是把一套“声明式属性”映射到引擎内部的：

- 数据对象（`this.__`，由 `@dataProcessor(UIData)` 提供）
- 布局对象（`this.__layout`，来自 `@leafer/core`）
- 渲染流程（通过模块 `UIBounds/UIRender` 注入）

它最重要的三个装饰器：

- `@dataProcessor(UIData)`：把属性写入/读取委托给 Data 类（`packages/display-module/data/src/UIData.ts`）
- `@useModule(UIBounds)` + `@useModule(UIRender)`：把 Bounds/Render 的实现“混入”到 UI 上（模块来自 `@leafer-ui/display-module`）
- `@rewriteAble()`：允许后续的插件/模块通过“重写机制”替换 UI 的部分行为（代码里大量注释 `@leafer-in/* will rewrite`）

你在 `UI.ts` 里会看到大量形如 `@xxxType(default)` 的属性装饰器，它们来自 `@leafer/core`（例如 `@boundsType`、`@positionType`、`@maskType` …）。这些装饰器的共同点是：

- 统一走 `__setAttr(...)`、触发布局/渲染变更标记
- 把“简单的 UI 属性赋值”变成“可被观察、可增量计算”的系统

### 2.2 `Leafer` 是运行时的根节点（负责 init/start/stop/resize）

文件：`packages/display/src/Leafer.ts`

`Leafer` 继承自 `Group`，但它额外负责：

- 按 `config` 创建 canvas 与控制器（renderer/watcher/layouter）
- 在“非 App 场景”下创建 selector/interaction/hitCanvasManager
- 管理生命周期状态：`created/ready/viewReady/viewCompleted`
- `start()/stop()` 驱动渲染循环
- `resize()` 与自动布局（AutoBounds）联动

它是你读“运行时如何拼起来”的最佳入口（建议配合 `@leafer-ui/web` 文档一起看）。

### 2.3 `Group/Box` 决定了 UI 树如何组织与渲染

#### `Group`

文件：`packages/display/src/Group.ts`

要点：

- `@useModule(Branch)`：借助 `@leafer/core` 的 `Branch` 模块提供 children 管理能力
- 通过 `set({ children })` 支持“带 children 的一次性数据赋值”
- `toJSON()` 默认会序列化 children（可通过 `childlessJSON` 控制）
- `pick(...)` 在这里是空实现，真正的 pick 由 `@leafer-ui/hit` 用原型扩展补上

#### `Box`

文件：`packages/display/src/Box.ts`

`Box` 是一个“既像 Rect（有 fill/stroke/尺寸），又像 Group（能包含 children）”的复合容器。

它最值得关注的点是 **重写机制（rewrite）**：

- `@rewrite(rect.__updateStrokeSpread)`、`@rewrite(rect.__render)` 等：直接复用 `Rect` 的逻辑
- 同时保留 `Group` 的渲染分支（children render）

因此读 `Box.ts` 能帮助你理解 Leafer UI 里一个重要设计：**通过“模块 + 重写”把能力组合成新组件，而不是靠深层继承堆逻辑**。

## 3. 基础图形类通常很薄：只声明 Tag + DataProcessor + RenderModule

以 `Rect` 为例（`packages/display/src/Rect.ts`）：

- `extends UI`
- `@dataProcessor(RectData)`：换成对应 Data（`packages/display-module/data/src/RectData.ts`）
- `@useModule(RectRender)`：挂载形状快速绘制实现（`packages/display-module/render/src/RectRender.ts`）

很多基础图形类的主体都很短，复杂逻辑通常下沉在：

- Data（属性解析、缓存标记、paint 计算）
- Render（通用绘制管线、effect/filter/paint）
- Hit（命中检测与拾取）

## 4. 推荐阅读路线（从 display 下钻）

1) `packages/display/src/Leafer.ts`：理解运行时 init 与控制器
2) `packages/display/src/UI.ts`：理解属性系统 + 模块注入 + 重写机制
3) `packages/display/src/Group.ts`、`packages/display/src/Box.ts`：理解 UI 树与容器
4) 任意基础图形（如 `Rect.ts`、`Text.ts`）+ 对应 Data/Render：看一个完整链路如何落地

