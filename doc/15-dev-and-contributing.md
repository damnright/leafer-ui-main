# 15. 本地联调与贡献指南

这篇文档回答两个问题：

1) 在“当前仓库快照”下，我能怎么使用源码/怎么联调？  
2) 如果我要给 `leafer-ui` 提交修改，建议怎么做？

## 1. 先讲清楚：当前仓库快照的“可运行性”

从根目录 `package.json` 与仓库结构看：

- 根包 `leafer-ui` 的 `main/exports/types` 指向 `dist/` 与 `types/` 等发布产物，但当前工作区不存在这些目录。
- 仓库内也没有常见的 workspace / 构建脚本配置（例如 `pnpm-workspace.yaml`、`lerna.json`、`turbo.json`、`tsconfig.json` 等）。
- 绝大多数源码位于 `packages/**/src`，并使用 `@leafer-ui/*` 的包名互相导入。

因此：**这份快照更偏“源码阅读/结构学习/局部修改”用途，而不是一个开箱即跑的构建工程**。

如果你的目标是“能跑 demo、能一键 build、能跨包联调”，通常需要在官方“主集成仓库”里进行（README 中有生态仓库索引）。

## 2. 使用方式 A（推荐）：作为 npm 包使用（业务接入最快）

如果你只想在业务项目里用 Leafer UI：

- 安装：`npm i leafer-ui`（或安装全量包 `leafer`）
- Web 使用：直接 `import { Leafer, Rect } from 'leafer-ui'`（默认走 `@leafer-ui/web` 平台装配）

这种方式的优点：

- 不需要关心本仓库的构建与跨包依赖
- API/行为以发布包为准，稳定

缺点：

- 你不容易直接改源码并实时生效（需要走 patch/link/workspace）

## 3. 使用方式 B：联调/改源码（推荐在主集成仓库中做）

如果你要做的事属于以下任一项：

- 改 `packages/display-module/render` 的渲染逻辑
- 改 `packages/interaction/*` 的交互分发逻辑
- 改 `packages/hit/*` 的命中检测逻辑
- 改 `external/partner` 的能力注入与实现

那么更推荐的联调方式是：在官方“主集成仓库”里改（它通常包含 workspace + 构建 + 示例/调试入口）。

原因：

- 本仓库快照缺少构建产物与构建配置
- scoped 包互相导入（`@leafer-ui/*`）需要 workspace/打包工具配合解析

## 4. 使用方式 C：在你的业务工程里“打补丁”调试（小改动可用）

当你只改一两处逻辑，但不想搭完整 workspace，可以考虑：

- 在业务工程中对安装的 `leafer-ui` 进行 patch（例如使用 `patch-package` 或包管理器的 patch 功能）
- 先把改动在本仓库中实现，然后把对应文件的修改“同步/应用”到业务工程的 node_modules 补丁中

适用场景：

- 修一个小 bug
- 验证一个渲染/事件边界条件

不适用场景：

- 需要跨多个 `@leafer-ui/*` 包协同改动
- 需要重新产出 `types/`、`dist/`、`lib/` 等发布物

## 5. 贡献流程与提交规范

仓库内已有贡献者文档：

- 行为准则：`contributor/CODE_OF_CONDUCT.md`
- 提交规范：`contributor/COMMIT_CONVENTION.md`

其中提交信息规范采用 conventional commits 风格（示例）：

- `feat(drag): add 'hover' option`
- `fix(editor): handle events on select`
- `perf(core): improve ...`

以及贡献流程（简版）：

1) fork 仓库
2) 建分支：`git checkout -b feat/xxxx`
3) 提交：`git commit -am 'feat(type): add xxxxx'`
4) push 分支并提交 PR
5) 首次提交需要同意 CLA（文档中给出链接）

## 6. 改代码前的“定位建议”（省时间）

当你遇到一个行为问题（例如“点击不触发、阴影边界不对、渐变渲染异常”），建议先用下面的定位路径：

1) 交互/事件是否进来（Web）：
   - `packages/interaction/interaction-web/src/Interaction.ts`
2) path 是否正确（pick/hit）：
   - `packages/interaction/interaction/src/Interaction.ts` 的 `findPath`
   - `packages/hit/src/*`
3) 是否走到绘制管线：
   - `packages/display-module/render/src/UIRender.ts`
4) 数据是否正确解析：
   - `packages/display-module/data/src/UIData.ts`
5) external 能力是否已注入：
   - `packages/partner/partner/src/index.ts`

配合 `doc/14-end-to-end-trace.md` 的断点清单，会更快。

