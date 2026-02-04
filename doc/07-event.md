# 07. 模块：`@leafer-ui/event`（UI 事件模型）

`@leafer-ui/event` 主要定义两件事：

1) UI 侧的事件类型（Pointer/Touch/Drag/Zoom/Rotate/Key…）  
2) 与事件相关的工具（键盘按键状态、鼠标按钮判断、拖拽边界辅助等）

它本身不负责“监听 DOM 事件”；监听与分发由 `@leafer-ui/interaction-*` 完成。你可以把 event 当成“事件数据结构 + 事件名规范”。

## 1. 入口与文件分布

- 入口：`packages/event/src/index.ts`
- 核心基类：
  - `packages/event/src/UIEvent.ts`
- 常用事件类型：
  - `packages/event/src/PointerEvent.ts`
  - `packages/event/src/TouchEvent.ts`
  - `packages/event/src/DragEvent.ts`
  - `packages/event/src/ZoomEvent.ts`
  - `packages/event/src/RotateEvent.ts`
  - `packages/event/src/SwipeEvent.ts`
  - `packages/event/src/KeyEvent.ts`
- 工具与辅助：
  - `packages/event/src/Keyboard.ts`：键盘按键按下/抬起状态
  - `packages/event/src/PointerButton.ts`：鼠标/指针按钮判断（left/right/middle）
  - `packages/event/src/DragBoundsHelper.ts`：拖拽边界辅助

## 2. `UIEvent`：事件的共同字段与坐标换算

文件：`packages/event/src/UIEvent.ts`

`UIEvent` 继承自 `@leafer/core` 的 `Event`，并补充 UI 场景常用字段：

- 坐标：`x/y`（通常是当前事件的点）
- 传播路径：`path`（从目标到根的 leaf 列表）
- 修饰键：`altKey/ctrlKey/shiftKey/metaKey` + `spaceKey`（来自 `Keyboard`）
- 鼠标按钮：`left/right/middle/buttons`（来自 `PointerButton`）
- 事件目标：`target/current` + `bubbles`

它还提供了一组常用换算方法（把事件点转换到不同坐标系）：

- `getBoxPoint(relative?)`
- `getInnerPoint(relative?)`
- `getLocalPoint(relative?)`
- `getPagePoint()`

这些方法会转调到 `current`（当前冒泡节点）的 UI 几何换算能力，因此事件对象本身保持轻量。

## 3. 事件名规范：以 `PointerEvent` 为例

文件：`packages/event/src/PointerEvent.ts`

`PointerEvent` 的特点是：

- 用 `static` 常量统一定义事件名（例如 `pointer.down`、`pointer.move`、`tap`、`double_click` 等）
- 用装饰器 `@registerUIEvent()` 注册到引擎的事件体系（来自 `@leafer/core`）

这套事件名规范会被 `InteractionBase` 直接使用（例如 `this.emit(PointerEvent.DOWN, data)`），从而保证上层订阅事件时稳定一致。

## 4. 它与 `interaction` 的协作方式

你可以这样理解：

- `@leafer-ui/event`：定义“事件是什么样子、叫什么名字”
- `@leafer-ui/interaction-*`：负责“把平台原始事件转成这些事件对象，并按路径派发”

具体派发位置在：

- `packages/interaction/interaction/src/Interaction.ts`（`InteractionBase`）

它会在 pointerDown/Move/Up 等入口里构造/更新事件数据，然后用 `emit(...)` 以 `path` 为基础分发。

## 5. 推荐阅读路线

如果你想快速弄清“事件对象里有什么、事件名怎么组织”：

1) `packages/event/src/UIEvent.ts`
2) `packages/event/src/PointerEvent.ts`
3) `packages/interaction/interaction/src/Interaction.ts`：看 emit 的调用点与事件生命周期

