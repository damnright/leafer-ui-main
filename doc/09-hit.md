# 09. 模块：`@leafer-ui/hit`（命中检测与拾取）

`@leafer-ui/hit` 负责解决两类问题：

- hit：给定一个点/半径，判断某个 leaf 是否被点中
- pick：在 UI 树中找出“点中”的目标（常用于 hover/点击选中）

它的实现风格有一个重要特点：**通过导入副作用给核心类的 prototype 挂载/重写方法**（例如给 `Leaf.prototype`、`UI.prototype`、`Group.prototype`、`LeaferCanvasBase.prototype` 增加命中相关方法）。

## 1. 入口与关键文件

- 入口：`packages/hit/src/index.ts`
  - 导出 `HitCanvasManager`
  - 并 `import './xxx'` 触发副作用注册（selector/hit/pick/canvas 扩展）

关键实现文件：

- `packages/hit/src/selector.ts`：设置 `Platform.getSelector(...)`
- `packages/hit/src/pick.ts`：给 `Group.prototype.pick(...)` 注入实现
- `packages/hit/src/LeafHit.ts`：给 `Leaf.prototype.hit/__hitWorld/__hitFill/__hitStroke/__hitPixel...` 注入实现
- `packages/hit/src/UIHit.ts`：给 `UI.prototype.__updateHitCanvas/__hit...` 注入实现
- `packages/hit/src/canvas.ts`：给 `LeaferCanvasBase.prototype.hitFill/hitStroke/hitPixel` 注入实现
- `packages/hit/src/HitCanvasManager.ts`：命中画布缓存管理

## 2. `pick`：从 UI 树里找“点中的目标”

文件：`packages/hit/src/pick.ts`

这里把 `Group.prototype.pick` 实现为：

1) `this.updateLayout()`（确保布局与 bounds 是最新的）
2) `Platform.getSelector(this).getByPoint(...)`（交给 selector 做路径搜索）

这也是为什么 `hit` 包里要先在 `selector.ts` 里定义 `Platform.getSelector`：

- 有 leafer 实例时用 `leafer.selector`
- 否则用 `Creator.selector()` 创建一个平台级 selector（缓存到 `Platform.selector`）

## 3. `hit`：单个 leaf 的命中检测流程

核心入口在 `packages/hit/src/LeafHit.ts`：

### 3.1 `Leaf.prototype.hit(worldPoint, hitRadius?)`

关键步骤：

1) `updateLayout()`（确保 bounds 最新）
2) 先用 `this.__world` 做快速 bounds 过滤（不在 bounds 内直接 false）
3) 如果是分支节点（`isBranch`）：
   - 走 selector：`Platform.getSelector(this).hitPoint(...)`
4) 否则走 `__hitWorld(...)`（由叶子自身的 hit 逻辑决定）

### 3.2 `Leaf.prototype.__hitWorld(point, forceHitFill?)`

关键点：

- 支持 data.hitRadius（额外命中半径）
- 支持 data.hitBox（只按 boxBounds 命中）
- 如果 hitCanvas 需要更新（`layout.hitCanvasChanged` 或尚未创建）：
  - 调用 `this.__updateHitCanvas()`
- 最后进入 `this.__hit(innerPoint)`（UIHit 等会提供具体实现）

## 4. UI 的命中：path hit 与 pixel hit 的组合

文件：`packages/hit/src/UIHit.ts`

### 4.1 `UI.prototype.__updateHitCanvas()`

核心逻辑是：

- 根据 fill/stroke 的类型与 hit 配置决定是否需要 **pixel hit**
  - 例如 alpha 像素填充（png/webp/svg snapshot）且 `hitFill === 'pixel'`
- 从 `hitCanvasManager` 取一张命中画布：
  - pixel hit → `getPixelType(..., { willReadFrequently: true })`
  - path hit → `getPathType(...)`
- 若是 pixel hit：
  - 计算缩放，使 renderBounds 适配到 hitCanvas 的固定大小
  - `__renderShape(...)` 画到 hitCanvas 上
- 最后统一 `__drawHitPath(h)` 把路径画出来，并设置 strokeOptions（供 path hit 使用）

### 4.2 `UI.prototype.__hit(innerPoint, forceHitFill?)`

命中顺序（可从代码读到）：

1) 子 boxStyle 命中（如果存在）
2) pixel hit（如果启用）
3) fill path hit（根据 `hitFill` 策略）
4) stroke path hit（根据 `hitStroke` + strokeAlign）

这套顺序设计的目的是：在可用时用更精确的 pixel hit，否则回退到 path 命中。

## 5. `HitCanvasManager`：命中画布缓存

文件：`packages/hit/src/HitCanvasManager.ts`

它继承自 `@leafer/core` 的 `CanvasManager`，并额外区分两类缓存：

- `pathList`：用于 path 命中的画布
- `pixelList`：用于 pixel 命中的画布（通常更重，因为要 read pixel）

当两类画布的总量超过 `maxTotal` 时会自动 clear，避免无限增长。

## 6. 性能与坑位提示

- pixel hit 使用 `getImageData`（见 `packages/hit/src/canvas.ts` 的 `hitPixel`），属于相对昂贵操作；因此需要：
  - 尽可能复用 hitCanvas（HitCanvasManager 缓存）
  - 在 Web 端启用 `willReadFrequently`（代码已做）
- path hit 更快，但对复杂透明纹理不如 pixel hit 精确；两者的切换由 `hitFill/hitStroke` 等配置决定。

