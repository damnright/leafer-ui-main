# Leafer UI 源码阅读与使用指南（WIP）

这份文档用于帮助你快速熟悉本仓库（`leafer-ui`）的源码组织方式，并指导你在项目中“使用/接入”源码进行阅读、调试与二次开发。

## 文档结构（按步骤逐步补全）

- `doc/01-overview.md`：总览（项目定位、分层、阅读入口、如何使用源码）
- `doc/02-directory-structure.md`：目录结构（仓库树形结构、packages 清单、每个包的职责速览）
- `doc/03-platform-web.md`：模块：`@leafer-ui/web`（Web 运行时装配与副作用初始化）
- `doc/04-core-and-app.md`：模块：`@leafer-ui/draw` / `@leafer-ui/core` / `@leafer-ui/app`（聚合层与 App 分层）
- `doc/05-display.md`：模块：`@leafer-ui/display`（UI 显示对象：UI/Leafer/Group/Box/基础图形）
- `doc/06-display-module.md`：模块：`@leafer-ui/display-module`（Data/Bounds/Render 模块与绘制管线）
- `doc/07-event.md`：模块：`@leafer-ui/event`（UI 事件模型与事件类型）
- `doc/08-interaction.md`：模块：`@leafer-ui/interaction*`（交互分发：Web/小程序桥接 + InteractionBase）
- `doc/09-hit.md`：模块：`@leafer-ui/hit`（命中检测与拾取：hit/pick/HitCanvasManager）
- `doc/10-external-partner.md`：模块：`@leafer-ui/external` + `@leafer-ui/partner`（可插拔能力：占位符与注入）
- `doc/11-platform-others.md`：模块：`@leafer-ui/node` / `@leafer-ui/worker` / `@leafer-ui/miniapp` / `@leafer-ui/canvaskit`（跨端适配要点）
- `doc/12-interface.md`：模块：`@leafer-ui/interface`（类型定义与模块接口约定）
- `doc/13-decorator-and-rewrite.md`：机制：装饰器与重写（`@leafer-ui/decorator`、`rewrite/rewriteAble`、副作用注入）
- `doc/14-end-to-end-trace.md`：全链路：从 import 到渲染/交互/命中（建议配合断点阅读）
- `doc/15-dev-and-contributing.md`：本地联调与贡献（当前仓库快照的可运行性说明、推荐联调方式、提交规范）
- `doc/16-debug-faq.md`：调试清单与常见问题（平台、装配、渲染、事件、命中）
- 后续：会继续补齐“按模块讲解”的文档（例如：`display`、`event`、`interaction`、`hit`、`external/partner`、`platform/*` 等）

## 重要说明（先读）

- 本仓库根目录的 `package.json` 指向发布产物（如 `dist/`、`types/`），但当前工作区内未包含这些构建产物目录；源码主要位于 `packages/**/src`。
- `packages/**` 内的源码大量使用形如 `@leafer-ui/*`、`@leafer/*` 的包名进行导入。这意味着“单独把本仓库当成一个可直接 `npm i && npm run dev` 的工程”通常行不通；更常见的做法是：
  - 直接安装已发布的 npm 包使用（面向业务接入）
  - 在官方的主集成仓库中联调/开发（面向贡献与深度调试）
