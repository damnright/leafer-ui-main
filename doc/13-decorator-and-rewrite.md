# 13. 机制：装饰器与重写（`@leafer-ui/decorator`、`rewrite/rewriteAble`、副作用注入）

Leafer UI 的源码有一个非常鲜明的风格：**大量使用装饰器来声明属性行为，并通过“模块注入 + 方法重写 + 原型副作用扩展”来实现组合式架构**。

本篇把这些机制拆开讲清楚，方便你读源码时不迷路。

## 1. `@leafer-ui/decorator`：UI 侧的少量“自定义装饰器”

目录：`packages/decorator/src/*`

它只提供几个非常聚焦的装饰器（入口在 `packages/decorator/src/index.ts`）：

### 1.1 `effectType(defaultValue?)`

文件：`packages/decorator/src/data.ts`

作用：把某个属性标记为“影响效果渲染”的属性，设置后：

- `this.__setAttr(key, value)`
- 如果 value 有值，标记 `(this as IUI).__.__useEffect = true`
- 并触发 `this.__layout.renderChange()`（让渲染器知道需要重绘）

典型用途：shadow/blur/filter 等 effect 类属性。

### 1.2 `resizeType(defaultValue?)`

设置后：

- 触发布局 boxChange
- 并调用 `(this as ICanvas).__updateSize()`（用于 Canvas 类在尺寸相关属性变化时更新内部尺寸）

### 1.3 `zoomLayerType()`

用于 `UI.zoomLayer` 这类“取值来自运行时环境”的属性：

- 如果是 app：返回 `app.tree.zoomLayer`
- 如果是 leafer：返回自身（或内部缓存）
- 否则返回 `this.leafer.zoomLayer`

这让“缩放层”的语义在 App/Leafer/UI 子节点之间保持一致。

### 1.4 `createAttr(defaultValue?)`

是一个通用的属性定义工具：用 `defineKey + createDescriptor` 直接为 UI 定义一个带默认值的属性描述符。

> 备注：更大量的属性装饰器（如 `@boundsType`、`@positionType`、`@maskType` 等）来自 `@leafer/core`，不在本仓库中。

## 2. `rewrite/rewriteAble`：可重写（插件化）是怎么落地的？

你在源码里会频繁看到类似注释：

- `// @leafer-in/flow will rewrite`
- `// @leafer-in/editor rewrite`
- `// @leafer-in/viewport will rewrite`

它们表达的不是“未来 TODO”，而是这个系统的设计点：**留出扩展包（`@leafer-in/*`）去覆盖/增强默认行为**。

### 2.1 `@rewriteAble()`：声明“此类允许被重写”

典型位置：

- `packages/display/src/UI.ts`
- `packages/display/src/Box.ts`

含义：允许后续通过重写机制替换类的部分方法/属性行为（具体实现由 `@leafer/core` 提供）。

### 2.2 `@rewrite(...)`：显式复用/替换别的实现

典型例子：`Box` 复用 `Rect` 的多段实现（见 `packages/display/src/Box.ts`）：

- `@rewrite(rect.__updateStrokeSpread)` → `Box.__updateStrokeSpread`
- `@rewrite(rect.__render)` → `Box.__renderRect`

这种写法的核心收益是：

- 避免继承层级越来越深（Rect 的能力可以被 Box 复用）
- 组合更清晰：Box 同时像 Rect（有 surface）又像 Group（有 children）

### 2.3 “重写点”通常怎么找？

你可以用两种方法快速定位：

1) 全局搜注释：`rg "@leafer-in/" packages`
2) 直接找 `@rewrite(`：定位哪些方法是“有意设计为可替换”的

## 3. 副作用注入：为什么 `import './xxx'` 很重要？

有些能力不是通过 `useModule(Object)` 混入，而是通过“导入模块即修改原型”的方式注入：

### 3.1 hit：原型扩展 Leaf/UI/Group/Canvas

入口：`packages/hit/src/index.ts`

它会：

- 给 `Platform.getSelector` 赋值（`selector.ts`）
- 给 `Group.prototype.pick` 赋值（`pick.ts`）
- 给 `Leaf.prototype.hit/__hitWorld/...` 赋值（`LeafHit.ts`）
- 给 `UI.prototype.__updateHitCanvas/__hit` 赋值（`UIHit.ts`）
- 给 `LeaferCanvasBase.prototype.hitFill/hitStroke/hitPixel` 赋值（`canvas.ts`）

因此 `@leafer-ui/hit` 必须被导入一次，相关能力才会生效。

### 3.2 mask：重写 `Group.prototype.__renderMask`

文件：`packages/partner/mask/src/index.ts`

它通过副作用扩展 Group 的渲染流程，使得 UI 树能按 `mask` 属性实现 clipping/alpha/grayscale 等遮罩模式。

而 `@leafer-ui/partner` 会显式 `import '@leafer-ui/mask'`，确保默认装配时 mask 能生效。

## 4. 读源码时的“心智模型”

建议你用下面三句话做总结（非常实用）：

1) **装饰器**决定“属性赋值如何影响 data/layout/render”
2) **useModule + ThisType** 决定“Bounds/Render 这类能力如何以 mixin 形式挂到 UI 上”
3) **rewrite + 副作用注入** 决定“默认实现如何被复用/替换/扩展”

