# Leafer UI 源码阅读与使用指南（WIP）

这份文档用于帮助你快速熟悉本仓库（`leafer-ui`）的源码组织方式，并指导你在项目中“使用/接入”源码进行阅读、调试与二次开发。

## 文档结构（按步骤逐步补全）

- `doc/01-overview.md`：总览（项目定位、分层、阅读入口、如何使用源码）
- `doc/02-directory-structure.md`：目录结构（仓库树形结构、packages 清单、每个包的职责速览）
- 后续：会继续补齐“按模块讲解”的文档（例如：`display`、`event`、`interaction`、`hit`、`platform/*` 等）

## 重要说明（先读）

- 本仓库根目录的 `package.json` 指向发布产物（如 `dist/`、`types/`），但当前工作区内未包含这些构建产物目录；源码主要位于 `packages/**/src`。
- `packages/**` 内的源码大量使用形如 `@leafer-ui/*`、`@leafer/*` 的包名进行导入。这意味着“单独把本仓库当成一个可直接 `npm i && npm run dev` 的工程”通常行不通；更常见的做法是：
  - 直接安装已发布的 npm 包使用（面向业务接入）
  - 在官方的主集成仓库中联调/开发（面向贡献与深度调试）

