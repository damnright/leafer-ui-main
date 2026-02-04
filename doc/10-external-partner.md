# 10. 模块：`@leafer-ui/external` + `@leafer-ui/partner`（可插拔能力体系）

这是 Leafer UI 源码里最“架构味”的一层：把一些能力（Paint、Effect、Filter、TextConvert…）做成可插拔模块，**UI 的渲染/边界计算依赖接口，但具体实现可以在运行时注入或被替换**。

## 1. `@leafer-ui/external`：能力占位符（壳）

文件：`packages/external/src/index.ts`

这里导出了一组对象/模块，供其他包依赖：

- `TextConvert` / `ColorConvert` / `UnitConvert`
- `PathArrow`
- `Paint` / `PaintImage` / `PaintGradient`
- `Effect`
- `Filter`
- `Export`
- `State`
- `Transition`

关键点在于：这些对象里很多是“空实现/壳”，例如：

- `Paint = {} as IPaintModule`
- `Effect = {} as IEffectModule`
- `TextConvert = {} as ITextConvertModule`

但同时它也包含少量“按需触发”的薄封装：

- `Filter.apply()` 内部会 `Plugin.need('filter')`
- `State.set()/setStyleName()` 内部会 `Plugin.need('state')`
- `Transition` 内置一个注册表：`register/get`

为什么要这么做？

- 上层（如 `UIRender/UIBounds/UIData`）只依赖 `external` 的接口，不需要关心实现来自哪里
- 在不同平台/不同裁剪包中，可以选择注入不同实现（甚至不注入）

## 2. `@leafer-ui/partner`：把真实实现注入到 external

文件：`packages/partner/partner/src/index.ts`

核心逻辑非常直白：

1) 从 `@leafer-ui/draw` 拿到 external 占位符对象：

- `import { Paint, PaintImage, PaintGradient, Effect, TextConvert, ColorConvert } from '@leafer-ui/draw'`

2) 引入各子模块的“真实实现”：

- `PaintModule`（`@leafer-ui/paint`）
- `PaintImageModule`（`@leafer-ui/image`）
- `PaintGradientModule`（`@leafer-ui/gradient`）
- `EffectModule`（`@leafer-ui/effect`）
- `TextConvertModule`（`@leafer-ui/text`）
- `ColorConvertModule`（`@leafer-ui/color`）
- 以及 `import '@leafer-ui/mask'`（mask 是通过原型副作用注入的，不是简单的对象实现）

3) 通过 `Object.assign(...)` 注入：

- `Object.assign(Paint, PaintModule)`
- `Object.assign(Effect, EffectModule)`
- ...

> 注：文件顶部注释 `leaferui's partner, allow replace` 说明这一层设计目标之一就是允许替换实现。

## 3. 为什么 UIRender/UIBounds 会依赖 external？

你在这些文件里能看到清晰的依赖关系：

- `packages/display-module/render/src/UIRender.ts`：
  - `import { Paint, Effect, Filter, ColorConvert } from '@leafer-ui/external'`
- `packages/display-module/bounds/src/UIBounds.ts`：
  - `import { Effect, Filter } from '@leafer-ui/external'`
- `packages/display-module/data/src/UIData.ts`：
  - `import { Paint, PaintImage, ColorConvert } from '@leafer-ui/external'`

这样做的好处是：

- UI 核心逻辑稳定：只依赖接口，不依赖实现细节
- 能力可裁剪：某些构建目标可以不打包某些实现，按需加载或用别的实现替换
- 生态可扩展：`@leafer-in/*` 插件可以重写/增强其中某些能力（源码里也保留了重写入口）

## 4. 每个 partner 子模块通常提供什么？

以几个典型模块为例（你可按需深入）：

- `packages/partner/paint/src/index.ts`：`PaintModule`
  - `fill/stroke/fills/strokes/shape/compute` 等核心绘制能力
- `packages/partner/image/src/index.ts`：`PaintImageModule`
  - image paint 的数据结构、pattern 生成、drawImage、回收等
- `packages/partner/gradient/src/index.ts`：`PaintGradientModule`
  - linear/radial/conic gradient 生成与变换
- `packages/partner/effect/src/index.ts`：`EffectModule`
  - shadow/innerShadow/blur/backgroundBlur 与 spread/transform 辅助
- `packages/partner/text/src/index.ts`：`TextConvertModule`
  - `getDrawData`（文本绘制数据转换）
- `packages/partner/color/src/index.ts`：`ColorConvertModule`
  - 颜色字符串转换
- `packages/partner/mask/src/index.ts`：通过副作用重写 `Group.prototype.__renderMask`

## 5. 如何理解“可替换/可定制”？

从源码设计角度，你可以这样做定制（概念层面）：

- 方案 A：在更早的时机对 external 占位符 `Object.assign(...)` 注入你自己的实现（覆盖 partner 的默认实现）
- 方案 B：实现一个类似 `@leafer-ui/partner` 的“装配器”，在你的构建目标中只注入你需要的模块
- 方案 C：利用 `Plugin.need('xxx')` 的机制，把某些能力延后到真正用到时再加载（需要更完整的构建/插件环境支持）

在当前这份仓库快照里，你主要能看到 A/B 的结构如何工作；C 通常需要配合更完整的 LeaferJS 集成仓库与构建体系。

