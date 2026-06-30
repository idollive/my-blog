# My-blog 实施进度

## 步骤 1：创建 Astro 项目 ✅

**完成时间：** 2026-06-30

**执行内容：**
- 使用 `create-astro@5.2.0` CLI 以 `minimal` 模板在当前目录搭建项目
- 由于当前目录非空（含 `memory-bank/`），CLI 自动在子目录 `wandering-wavelength/` 创建，随后将所有文件移动到根目录，删除空子目录
- 初始脚手架安装了 Astro 7，降级到 `astro@^5.18.2` 以匹配技术栈文档的 ^5.x 约束
- 修正 `package.json` 中的 `name` 字段从 `wandering-wavelength` 到 `my-blog`

**最终状态：**

| 项 | 值 |
|----|-----|
| Astro 版本 | `^5.18.2` |
| TypeScript | 严格模式 (`astro/tsconfigs/strict`) |
| 项目名 | `my-blog` |
| 模板 | `minimal`（仅 `src/pages/index.astro`） |

**验证结果：** `astro dev` 在 `http://localhost:4321` 正常显示 Astro 默认欢迎页。

**备注：**
- `CLAUDE.md` 和 `AGENTS.md` 是脚手架自动生成的，与 `memory-bank/CLAUDE.md` 的链接同名——它们各自服务于不同目的（前者是 Astro 项目的 AI 指引，后者是项目整体设计文档）
- `.vscode/extensions.json` 和 `.vscode/launch.json` 由脚手架生成，建议保留以便 VS Code 用户获得 Astro 语法支持

---

## 步骤 2：安装核心依赖 ✅

**完成时间：** 2026-06-30

**执行内容：**
- 安装 6 个运行时依赖 + 1 个开发依赖
- 配置 `astro.config.mjs`：添加 `@astrojs/mdx` 集成 + `@tailwindcss/vite` 插件
- 创建 `src/styles/global.css`（Tailwind CSS 入口，含 `@import "tailwindcss"`）

**最终 `package.json` 依赖：**

| 依赖 | 安装版本 | 用途 |
|------|----------|------|
| `@astrojs/mdx` | ^4.3.14 | MDX 内容支持 |
| `@tailwindcss/vite` | ^4.3.2 | Tailwind CSS 4 Vite 插件 |
| `@prisma/client` | ^6.19.3 | 数据库 ORM 客户端 |
| `tailwindcss` | ^4.3.2 | 原子化 CSS 框架 |
| `@tailwindcss/typography` | ^0.5.20 | 排版插件（prose 类） |
| `pinyin-pro` | ^3.28.1 | 中文标题转拼音 slug |
| `prisma` (dev) | ^6.19.3 | Prisma CLI + 迁移工具 |
| `@astrojs/check` (dev) | — | Astro 类型检查 |
| `typescript` (dev) | — | TypeScript 编译器 |

**与计划的偏差：**

| 偏差 | 原因 |
|------|------|
| `@astrojs/mdx@^4.3` 代替 `^5.x` | v5 要求 `astro@^6.0.0`，与 Astro 5.18.2 不兼容 |
| `@tailwindcss/vite` 代替 `@astrojs/tailwind` | `@astrojs/tailwind` 已弃用，Tailwind 4 官方推荐 Vite 插件（`astro add tailwind` 自动配置） |
| `@tailwindcss/typography@^0.5` 代替 `^4.x` | npm 上最高版本为 0.5.20，4.0.0 不存在 |

**验证结果：** `npx astro check` 输出 0 错误、0 警告。`npx astro dev` 启动成功。

**备注：**
- `@astrojs/mdx` 未通过 `astro add mdx` 安装（该命令试图安装 v7，与 Astro 5 不兼容），改为手动 `npm install` 后在 `astro.config.mjs` 中手动添加 `integrations: [mdx()]`
- `@tailwindcss/vite` 与 Astro 内置 Vite 版本不同导致 TS 类型错误，已在 `astro.config.mjs` 中使用 `@type {any}` 类型断言绕过，运行时无影响

---

## 步骤 3：初始化 Prisma ✅

**完成时间：** 2026-06-30

**执行内容：**
- 运行 `npx prisma init` 生成 `prisma/schema.prisma`、`prisma.config.ts` 和 `.env`
- 将 `prisma/schema.prisma` 中的 provider 从 `postgresql` 改为 `mysql`
- 将 `.env` 中的 `DATABASE_URL` 替换为 MySQL 连接字符串占位格式
- 在 `.env` 中添加 `ADMIN_PASSWORD_HASH` 键（占位值）

**与计划的偏差：**

| 偏差 | 原因 |
|------|------|
| 新增 `prisma.config.ts` 文件 | Prisma 6.19.3 引入的新配置文件格式，分离了 datasource URL 和 migration 路径配置 |
| `generator client` 输出到 `src/generated/prisma` | Prisma 6 默认将客户端生成到项目源码目录而非 `node_modules`，后续步骤导入路径需相应调整 |

**最终状态：**

| 文件 | 关键内容 |
|------|----------|
| `prisma/schema.prisma` | `provider = "mysql"`，`generator client` → `src/generated/prisma` |
| `prisma.config.ts` | datasource URL 从 `.env` 读取，engine: "classic" |
| `.env` | `DATABASE_URL` (MySQL 占位符) + `ADMIN_PASSWORD_HASH` (占位符) |

**验证结果：** `npx prisma format` 格式化成功，`npx prisma validate` 输出 "The schema at prisma\schema.prisma is valid 🚀"。

**备注：**
- 实际的 PlanetScale 连接字符串将在步骤 5 填写
- `ADMIN_PASSWORD_HASH` 的实际值将在步骤 19 生成
- `prisma.config.ts` 要求 `dotenv` 依赖（`import "dotenv/config"`），当前由 Prisma CLI 内置提供，后续若独立运行可能需要显式安装

---

## 步骤 4：初始化 Git 仓库 ✅

**完成时间：** 2026-06-30

**执行内容：**
- Git 仓库已在步骤 1 初始化，`.gitignore` 已覆盖 `node_modules/`、`.env`、`dist/`、`.astro/`
- 添加远程仓库 `origin` → `https://github.com/idollive/my-blog.git`
- 将全部 4 条提交推送到 GitHub

**最终状态：**

| 项 | 值 |
|----|-----|
| 远程仓库 | `https://github.com/idollive/my-blog.git` |
| 分支 | `main` |

| 提交数 | 4（初始 + 步骤 1~3） |
| `.gitignore` | 已覆盖 `node_modules/`、`.env`、`dist/`、`.astro/`、`/src/generated/prisma` |

**与计划的偏差：**

| 偏差 | 原因 |
|------|------|
| 无需重新 `git init` | 步骤 1 的 `create-astro` CLI 已自动初始化仓库并创建初始提交 |
| `gh` CLI 不可用 | 未安装 GitHub CLI，手动在 GitHub 网页创建仓库后添加 remote |

**验证结果：** `git log --oneline` 显示 4 条提交。`git remote -v` 显示 `origin → https://github.com/idollive/my-blog.git (fetch/push)`。

**备注：**
- 工作区有未提交变更（`memory-bank/CLAUDE.md` + `.claude/skills/`），建议在步骤 5 前提交
