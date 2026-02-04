# 06. 模块：`@leafer-ui/display-module`（Data / Bounds / Render）

`@leafer-ui/display-module` 是 UI 显示对象的“能力模块层”。它把 UI 的核心能力拆成三个方向：

- Data：属性与输入数据如何存储、解析、派生（决定“数据怎么变、需要重算什么”）
- Bounds：边界如何计算（决定“哪里需要绘制/命中/裁剪”）
- Render：如何绘制（决定“最终像素如何画出来”）

在 `UI` 基类里（`packages/display/src/UI.ts`），你能看到：

- `@useModule(UIBounds)`
- `@useModule(UIRender)`
- `@dataProcessor(UIData)`

这三句基本就把“UI 的一半灵魂”装配完成了。

## 1. 包结构与入口

- `packages/display-module/display-module/src/index.ts`（`@leafer-ui/display-module` 的入口）
  - `export * from '@leafer-ui/data'`
  - `export * from '@leafer-ui/bounds'`
  - `export * from '@leafer-ui/render'`

也就是说，它只是一个聚合包，真正实现分别在：

- `packages/display-module/data` → `@leafer-ui/data`
- `packages/display-module/bounds` → `@leafer-ui/bounds`
- `packages/display-module/render` → `@leafer-ui/render`

## 2. Data：`UIData` 是属性系统的“计算中枢”

文件：`packages/display-module/data/src/UIData.ts`

你可以把 `UIData` 理解为：

- 所有 UI 属性的“真实存储体”
- 负责把用户输入（字符串/对象/数组/路径等）归一化
- 负责维护大量派生标记位（例如 `__complex`、`__isFills`、`__needComputePaint` 等）
- 触发布局/渲染变更（例如 width/height 变更、fill/stroke 变更）

典型逻辑（从代码可见）：

- `setFill/setStroke`：
  - 字符串：用 `ColorConvert.hasTransparent(...)` 标记透明度
  - 对象/数组：进入 `__setPaint(...)`，把“paint 输入”记录到 `__input`，并标记需要 compute
- `setPath`：
  - 字符串/命令对象：走 `PathConvert.parse/objectToCanvasData` 转成统一数据结构
- `__computePaint()`：
  - 调用 `Paint.compute('fill'|'stroke', leaf)`（注意：`Paint` 来自 `@leafer-ui/external`，最终实现由 partner 注入）

配套的还有各图形的 Data 类（`RectData/TextData/ImageData/...`），通常只扩展/定制自身需要的属性。

## 3. Bounds：`UIBounds` 负责“渲染扩展边界”

文件：`packages/display-module/bounds/src/UIBounds.ts`

`UIBounds` 提供的方法主要用于计算“扩张量（spread）”，影响：

- renderBounds：绘制时需要额外预留的像素（阴影、模糊、滤镜、描边、箭头等）
- hitBounds：命中检测可能用到的边界

你会看到它依赖 `@leafer-ui/external` 的能力：

- `Effect.getShadowRenderSpread(...)`
- `Effect.getInnerShadowSpread(...)`
- `Filter.getSpread(...)`

这再次体现了可插拔设计：**Bounds 逻辑依赖接口能力，但实现可被替换/按需加载**。

## 4. Render：`UIRender` 是通用绘制管线

文件：`packages/display-module/render/src/UIRender.ts`

`UIRender` 的职责是把 Data 上的状态转换成真正的 canvas 绘制调用：

- `__updateChange()`：根据 fill/stroke/effect 状态维护渲染标记位
  - 是否单/多 paint、是否复杂形状、是否可走 fast shadow 等
- `__draw(...)`：主绘制入口（复杂路径 vs 快速路径）
- `__drawShape(...)`：仅绘制形状（常用于快照、命中、滤镜等）

通用渲染管线会调用外部能力模块：

- `Paint.fill/stroke/fills/strokes/shape/compute`
- `Effect.shadow/innerShadow/...`
- `Filter.apply(...)`
- `ColorConvert.string(...)`（例如 fast shadow 需要转色值）

这些都来自 `@leafer-ui/external`，并由 `@leafer-ui/partner` 注入真实实现。

## 5. 形状特化渲染：以 `RectRender` 为例

文件：`packages/display-module/render/src/RectRender.ts`

`RectRender` 只实现一个 `__drawFast`，用于 Rect 的快速绘制：

- 直接 `fillRect` / `strokeRect`
- 按 `strokeAlign` 处理 inside/center/outside
- 与 `UIRender` 的通用逻辑互补：当形状足够简单时可以走更快的路径

在 `Rect` 类（`packages/display/src/Rect.ts`）上通过 `@useModule(RectRender)` 挂载。

## 6. 阅读建议：如何把它们串起来？

建议你按下面顺序阅读并对照调用关系：

1) `packages/display/src/UI.ts`：看 `@useModule` / `@dataProcessor` 如何装配
2) `packages/display-module/data/src/UIData.ts`：看属性写入后如何打标记、触发重算
3) `packages/display-module/render/src/UIRender.ts`：看绘制管线如何消费这些标记位
4) `packages/display-module/bounds/src/UIBounds.ts`：看 renderBounds 的 spread 如何计算

这样你就能形成一个稳定的心智模型：**UI 属性 → Data 归一化/标记 → Bounds/Render 消费 → 渲染结果**。

