# 12. 模块：`@leafer-ui/interface`（类型定义与模块接口约定）

`@leafer-ui/interface` 是整个 UI 栈的“协议层”：它不提供运行时代码逻辑，主要用于：

- 统一 UI 对象（Rect/Text/Image/Group/Leafer/App…）的 TS 接口与数据结构命名
- 统一各能力模块（Render/Bounds/Hit/Paint/Effect/Filter…）的接口形状
- 作为 `@leafer-ui/external`（能力占位符）与 `@leafer-ui/partner`（能力注入）的“类型契约”

> 你在读源码时，看到 `implements IUI`、`as IPaintModule`、`IUIRenderModule` 之类的类型，几乎都来自这里。

## 1. 入口与组织方式

- 入口：`packages/interface/src/index.ts`
- 主要内容分三块：
  1) UI 对象接口：`packages/interface/src/IUI.ts`
  2) 能力模块接口：`packages/interface/src/module/*`
  3) 细分类型：`packages/interface/src/type/*`、`ICommonAttr.ts`、`IAnimation.ts`、`IScroller.ts`、`editor/*` 等

同时它还会：

- `export * from '@leafer/interface'`：把更底层的引擎通用接口（canvas、bounds、事件基础类型等）一起导出，方便上层只依赖一个入口。

## 2. UI 对象接口的命名规律（读源码时很省脑子）

在 `IUI.ts` 中，一个 UI 对象通常对应三类类型：

- `IXXX`：运行时实例接口（例如 `IRect`、`IText`、`IGroup`）
- `IXXXData`：DataProcessor 对应的数据接口（例如 `IRectData`、`ITextData`）
- `IXXXInputData`：用户输入/序列化输入接口（例如 `IRectInputData`、`ITextInputData`）

并且常见结构是：

- `interface IRect extends IUI { __: IRectData }`
- `export interface IRectData extends ... , IUIData {}`
- `export interface IRectInputData extends ... , IUIBaseInputData {}`

这样你在实现类里看到：

- `export class Rect<TInputData = IRectInputData> extends UI<TInputData> implements IRect`
- `@dataProcessor(RectData) declare public __: IRectData`

就能非常直接把“类/数据/输入类型”对应起来。

## 3. 模块接口：为什么大量使用 `ThisType<T>`？

你会在 `packages/interface/src/module/*` 看到这种写法：

- `export type IUIRenderModule = IUIRender & ThisType<IUI>`
- `export type IRectRenderModule = IRectRender & ThisType<IRect>`
- `export type IUIBoundsModule = IUIBounds & ThisType<IUI>`

原因是：模块对象通常是“被 mixin 到 UI 实例上运行的”，模块内部大量使用 `this` 指向 UI 实例。

借助 `ThisType<T>`，TS 能在模块对象的方法体里把 `this` 当成 `T` 来做类型检查（例如 `this.__layout`、`this.__`、`this.__drawRenderPath(...)`）。

这与 `@leafer/core` 的 `useModule(...)` 机制相配套。

## 4. 与 `external/partner` 的契约关系（非常重要）

`@leafer-ui/external` 中的占位符对象，会被显式断言成接口类型：

- `Paint = {} as IPaintModule`
- `Effect = {} as IEffectModule`
- `Filter = ... as IFilterModule`
- `TextConvert = {} as ITextConvertModule`

这些接口都来自 `@leafer-ui/interface` 的 `module/` 目录，例如：

- `module/IPaint.ts`：`IPaintModule` / `IPaintImageModule` / `IPaintGradientModule`
- `module/IEffect.ts`：`IEffectModule`
- `module/IFilter.ts`：`IFilterModule`

然后 `@leafer-ui/partner` 会把实现注入进去（`Object.assign(Paint, PaintModule)`），确保运行时满足接口约定。

因此当你新增/修改某个能力模块时，最佳实践是：

1) 先在 `@leafer-ui/interface` 里明确接口（或补齐字段）
2) 再在实现包（`packages/partner/*` 等）补齐实现
3) 最后确认 `@leafer-ui/partner` 的装配逻辑覆盖到了该能力

## 5. 阅读建议：什么时候该打开 `interface`？

常见三种场景：

1) 看到一个字段/方法不知道是什么（例如 `__drawAfterFill`、`__updateRenderSpread`）
   - 先去 `packages/interface/src/module/*` 找对应的模块接口
2) 想知道某个 UI 对象有哪些输入字段（例如 Text 的 `fontFamily/lineHeight/...`）
   - 去 `packages/interface/src/IUI.ts` 找 `ITextAttrData/ITextInputData`
3) 想做扩展/插件（例如实现一个新的 filter processor）
   - 去 `packages/interface/src/module/IFilter.ts` 看 `register/apply/getSpread` 的约定

