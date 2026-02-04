# 08. 模块：`@leafer-ui/interaction*`（交互：监听/转换/派发）

交互相关代码分三层：

- `@leafer-ui/interaction`：平台无关的交互核心（`InteractionBase` + 拖拽/游标/路径检查等）
- `@leafer-ui/interaction-web`：Web 环境事件监听与转换（Pointer/Mouse/Touch/Wheel/Gesture）
- `@leafer-ui/interaction-miniapp`：小程序环境事件桥接（通过 `receiveEvent` 接收宿主事件）

平台入口（如 `@leafer-ui/web`）会选择对应的 `Interaction` 实现挂到 `Creator.interaction`，这样 `Leafer.init()` 时就能创建正确的交互控制器。

## 1. `InteractionBase`：交互状态机（核心）

文件：`packages/interaction/interaction/src/Interaction.ts`

它实现的是“UI 交互状态机”，包括：

- pointer 生命周期：`pointerDown` → `pointerMove` → `pointerUp` / `pointerCancel`
- hover/enter/leave/over/out 路径维护
- tap/double tap/long press/menu 等手势组合逻辑
- drag 逻辑（委托给 `Dragger`）
- keyboard 逻辑：`keyDown/keyUp`（可选启用）
- transform 相关接口（move/zoom/rotate/wheel/multiTouch...）—— 默认空实现，通常由插件（如 viewport）重写

它依赖的关键组件：

- `packages/interaction/interaction/src/Dragger.ts`：拖拽状态与拖拽事件派发
- `packages/interaction/interaction/src/emit.ts`：沿 path 派发事件
- `packages/interaction/interaction/src/InteractionHelper.ts`：路径可拖拽/是否含事件类型等判断
- `packages/event/*`：事件类型与事件名常量（PointerEvent/DropEvent/KeyEvent...）

### 1.1 一个最常见的调用链（以 pointerDown 为例）

`pointerDown()` 的核心步骤大致是：

1) 标准化按键（例如默认左键）
2) 更新 downData / hoverData（维护交互状态）
3) 计算当前 path（命中路径 + 默认路径）
4) 触发 `PointerEvent.BEFORE_DOWN` / `PointerEvent.DOWN`
5) 启动 tap/longPress 计时器
6) 初始化 dragger 的拖拽数据（注意它在 DOWN 事件之后）

理解这条链路有助于你定位“为什么某个事件没有触发/触发顺序不对”。

## 2. Web 实现：`@leafer-ui/interaction-web`

文件：`packages/interaction/interaction-web/src/Interaction.ts`

这一层解决的问题是：**如何把 DOM 事件变成 `InteractionBase` 能理解的事件数据**。

它做了几件重要的事：

- 对 canvas view 与 window 绑定监听：
  - view 上：`pointerdown/mousedown/touchstart/contextmenu/wheel/gesture*`
  - window 上：`pointermove/pointerup/.../keydown/keyup/scroll`
- 事件优先级：PointerEvent > TouchEvent > MouseEvent（按配置与实际触发动态切换）
- 坐标换算：`getLocal(e)` 把 client 坐标转到 canvas 内部坐标（可选 update client bounds）
- 触摸与多指：
  - 识别 `useMultiTouch`
  - 生成 keepTouch list 并调用 `multiTouch(...)`
- wheel/gesture：转换成 zoom/rotate 等高阶输入（供 transformer/viewport 使用）

你在排查 Web 端交互问题时，通常直接从这里下断点看“原生事件有没有被正确转换”。

## 3. 小程序实现：`@leafer-ui/interaction-miniapp`

文件：`packages/interaction/interaction-miniapp/src/Interaction.ts`

小程序环境通常没有完整的 DOM 事件模型，所以它采用“桥接接收”的方式：

- `Interaction.__listenEvents()` 会把 `this.receive` 绑定到 `config.eventer.receiveEvent`
- 外部（平台包）把宿主事件喂给 `receive(e)`，再按 `e.type` 分发到 `onTouchStart/Move/End/Cancel`

配合平台包 `@leafer-ui/miniapp`（`packages/platform/miniapp/src/index.ts`）你会看到：

- `Leafer.prototype.receiveEvent = function(event){ this.interaction.receive(event) }`
- 这样宿主只要调用 `leafer.receiveEvent(rawEvent)` 就能进入交互体系

## 4. 推荐调试断点

- 交互状态问题：`packages/interaction/interaction/src/Interaction.ts` 的 `pointerDown/pointerMove/pointerUp`
- Web 事件监听/转换问题：`packages/interaction/interaction-web/src/Interaction.ts` 的 `onPointerDown/onTouchStart/onWheel`
- 拖拽问题：`packages/interaction/interaction/src/Dragger.ts`
- 事件派发路径问题：`packages/interaction/interaction/src/emit.ts`

