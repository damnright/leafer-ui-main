# 02. 目录结构：仓库怎么组织？每个目录是什么？

## 1. 根目录（你最常用的入口）

```
leafer-ui-main/
  README.md
  package.json
  src/
    index.ts
  packages/
  contributor/
```

- `README.md`：项目介绍、文档入口、生态仓库索引
- `package.json`：根包 `leafer-ui` 的发布配置（注意：这里会引用 `dist/`、`types/` 等发布产物）
- `src/index.ts`：根入口，直接 re-export `@leafer-ui/web`
- `packages/`：核心源码区（本仓库最重要的目录）
- `contributor/`：贡献相关文档与少量示例代码

## 2. `packages/` 总览（按“领域分组”组织）

`packages/` 下不是“一个 packages = 一个 npm 包”的扁平结构，而是“领域文件夹 / 包文件夹”混合：有些目录本身就是包（如 `packages/app`），有些目录下包含多个包（如 `packages/platform/*`）。

### 2.1 UI 核心与基础能力

- `packages/app` → `@leafer-ui/app`：应用层封装（导出 `App`）
- `packages/core/core` → `@leafer-ui/core`：UI 核心聚合（draw + app + interaction + event + hit）
- `packages/core/draw` → `@leafer-ui/draw`：绘制层聚合（@leafer/core + display + display-module + decorator + external）
- `packages/decorator` → `@leafer-ui/decorator`：属性/效果等装饰器与元数据工具
- `packages/interface` → `@leafer-ui/interface`：UI 层接口与类型声明集合（大量 TS 类型导出）

### 2.2 显示对象与显示模块（Data / Bounds / Render）

- `packages/display` → `@leafer-ui/display`：UI 显示对象（Leafer/Group/Rect/Text/Image…）
- `packages/display-module/data` → `@leafer-ui/data`：UIData 与各图形 Data 类型
- `packages/display-module/bounds` → `@leafer-ui/bounds`：UIBounds（边界计算）
- `packages/display-module/render` → `@leafer-ui/render`：UIRender/RectRender（渲染相关）
- `packages/display-module/display-module` → `@leafer-ui/display-module`：聚合导出 data/bounds/render

### 2.3 事件、交互、命中检测

- `packages/event` → `@leafer-ui/event`：UI 事件类型与工具（Pointer/Touch/Drag/Zoom/Key…）
- `packages/interaction/interaction` → `@leafer-ui/interaction`：跨平台交互基础（Cursor/Dragger…）
- `packages/interaction/interaction-web` → `@leafer-ui/interaction-web`：Web 端交互实现（导出 `Interaction`）
- `packages/interaction/interaction-miniapp` → `@leafer-ui/interaction-miniapp`：小程序交互实现
- `packages/hit` → `@leafer-ui/hit`：命中检测与拾取（HitCanvasManager、pick、各 UI hit 规则注册）

### 2.4 可插拔能力（Partner / External）

- `packages/external` → `@leafer-ui/external`：能力模块“占位符”（运行时由 partner 注入真实实现）
- `packages/partner/partner` → `@leafer-ui/partner`：把各能力模块注入到占位符中（`Object.assign(...)`）
- `packages/partner/*`：
  - `color` → `@leafer-ui/color`
  - `gradient` → `@leafer-ui/gradient`
  - `effect` → `@leafer-ui/effect`
  - `image` → `@leafer-ui/image`
  - `paint` → `@leafer-ui/paint`
  - `text` → `@leafer-ui/text`
  - `mask` → `@leafer-ui/mask`（按需引入以启用相关能力）

### 2.5 平台适配（Platform）

`packages/platform/*` 下每个子目录通常对应一个 npm 包，用于装配 `Platform` 与 `Creator`，并选择对应平台的交互实现：

- `packages/platform/web` → `@leafer-ui/web`：Web 端运行时入口（根包 `leafer-ui` 默认指向它）
- `packages/platform/worker` → `@leafer-ui/worker`：Worker 运行时入口
- `packages/platform/node` → `@leafer-ui/node`：Node 运行时入口（并额外导出 `@leafer-in/export`）
- `packages/platform/miniapp` → `@leafer-ui/miniapp`：小程序运行时入口（尝试绑定 `wx` 并桥接事件）
- `packages/platform/canvaskit` → `@leafer-ui/canvaskit`：CanvasKit 相关（当前工作区 `src/index.ts` 为空，通常依赖构建产物）

## 3. `contributor/`（贡献相关）

- `contributor/CODE_OF_CONDUCT.md`：行为准则
- `contributor/COMMIT_CONVENTION.md`：提交信息规范
- `contributor/code/`：少量贡献者示例代码（如 `Checkerboard.ts`）

