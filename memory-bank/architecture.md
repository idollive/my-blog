# My-blog 架构文档

## 部署模型：本地管理 + 静态部署

```
本地 (astro dev)                     远程 (Vercel)
┌─────────────────────┐            ┌──────────────────────┐
│ /admin 后台          │            │ 纯静态 HTML           │
│   ├─ 文章 CRUD       │  git push  │   ├─ /               │
│   ├─ 写 MDX 文件      │ ────────▶ │   ├─ /posts/[slug]   │
│   └─ 写 MySQL 元数据  │            │   ├─ /archive        │
│                      │            │   └─ /about          │
│ MySQL (PlanetScale)  │            │                      │
└─────────────────────┘            └──────────────────────┘
```

- 后台仅在本地 `astro dev` 时使用
- `git push` 触发 Vercel `astro build`，生成纯静态 HTML
- Vercel 不需要数据库连接——前台页面全部从 MDX 文件读取（Astro 内容集合 API）

## 数据存储策略

| 数据 | 存储位置 | 用途 |
|------|----------|------|
| 文章正文 | `src/content/blog/*.mdx` | 唯一数据源，Astro 内容集合读取 |
| 文章元数据 | MySQL `Post` 表 | title/slug/excerpt/status/categoryId/viewCount |
| 分类 | MySQL `Category` 表 | id/name/slug |
| 标签 | MySQL `Tag` 表 | id/name/slug |
| 文章-标签关联 | MySQL `PostTag` 表 | postId + tagId 联合主键 |
| 站点设置 | `src/config.ts` | SITE_TITLE/SITE_DESCRIPTION/SITE_LOGO |
| 管理员密码 | `.env` `ADMIN_PASSWORD_HASH` | salt:hash 格式（crypto.scryptSync） |

**关键约束：** `Post` 表**不含 content 字段**。正文仅存 MDX 文件，数据库只存元数据。

## 前台页面数据流

```
src/content/blog/*.mdx  →  getCollection('blog')  →  页面渲染  →  静态 HTML
（构建时）                   （Astro 内容集合 API）         （.astro 组件）
```

不涉及数据库查询。所有页面的数据来自 MDX 文件的 frontmatter + 正文。

## 后台数据流

```
/admin 表单 POST  →  Prisma 写入 MySQL（元数据）
                  →  fs.writeFile 写入 MDX 文件（正文 + frontmatter）
```

管理员操作同时写入两个存储：MySQL 记录元数据，MDX 文件记录正文。

## 认证

- 单用户密码认证（Node.js `crypto.scryptSync`）
- `.env` 中 `ADMIN_PASSWORD_HASH` = `salt:hash`（hex 编码）
- 登录成功后设 httpOnly cookie，7 天有效期
- Astro 中间件检查 `/admin/*` 路径的 cookie

## 技术栈

| 层级 | 选择 |
|------|------|
| 框架 | Astro 5 |
| 样式 | Tailwind CSS 4 + @tailwindcss/typography |
| 内容格式 | MDX（含 rehype-pretty-code 语法高亮） |
| 数据库 | MySQL 8 + Prisma ORM |
| 数据库托管 | PlanetScale |
| 部署 | Vercel（纯静态模式） |
| 认证 | 单用户密码（crypto.scryptSync） |

## 路由清单

```
/                   首页（已发布文章列表 + 分页）
/posts/[slug]       文章详情（正文 + TOC + 语法高亮）
/archive            按年月归档
/about              关于页面
/admin/login        后台登录
/admin/posts        后台文章列表
/admin/posts/new    新建文章
/admin/posts/[id]/edit  编辑文章
/admin/posts/[id]/toggle-status  切换文章状态 API
/admin/posts/[id]/update         更新文章 API
/admin/posts/[id]/delete         删除文章 API
/admin/settings     站点设置
/admin/logout       退出登录
```

## 当前项目文件布局（步骤 2 之后）

### 根目录配置文件

| 文件 | 作用 |
|------|------|
| `astro.config.mjs` | Astro 核心配置：注册 `@astrojs/mdx` 集成 + `@tailwindcss/vite` Vite 插件。Tailwind 插件通过 `@type {any}` 类型断言绕过 Vite 版本差异导致的 TS 类型不兼容 |
| `package.json` | 项目元数据、scripts（dev/build/preview）、7 个运行时依赖 + 3 个开发依赖 |
| `tsconfig.json` | 继承 Astro 严格模式 TypeScript 配置（`astro/tsconfigs/strict`） |
| `.gitignore` | 忽略 `node_modules/`、`dist/`、`.astro/`、`.env` |
| `.env` | （待步骤 3 创建）环境变量：`DATABASE_URL`、`ADMIN_PASSWORD_HASH` |

### `src/` 源代码目录

| 路径 | 状态 | 作用 |
|------|------|------|
| `src/pages/` | ✅ 已有 | Astro 文件路由页面。当前仅含 `index.astro`（脚手架默认页） |
| `src/pages/index.astro` | ✅ 已有 | 首页，后续改造为已发布文章列表 |
| `src/styles/global.css` | ✅ 已有 | Tailwind CSS 入口文件，含 `@import "tailwindcss"` 指令。后续添加 `@plugin "@tailwindcss/typography"` 和 design tokens |
| `src/layouts/` | 🔜 步骤 11 | 布局组件（BaseLayout、AdminLayout） |
| `src/components/` | 🔜 后续 | 可复用 UI 组件（React/Vue Islands 按需） |
| `src/lib/` | 🔜 步骤 8 | 工具库（`prisma.ts` 数据库客户端单例、`utils.ts`） |
| `src/content/` | 🔜 步骤 9 | Astro 内容集合（`blog/` 目录存放 MDX 文章） |
| `src/middleware.ts` | 🔜 步骤 19 | Astro 中间件，处理 `/admin/*` 认证 |

### 其他目录

| 路径 | 状态 | 作用 |
|------|------|------|
| `public/` | ✅ 已有 | 静态资源（favicon.ico、favicon.svg），构建时原样复制到 `dist/` |
| `prisma/` | 🔜 步骤 3 | Prisma schema（`schema.prisma`）+ 迁移文件 |
| `.vscode/` | ✅ 已有 | VS Code 配置（推荐扩展 + 调试启动项） |
| `memory-bank/` | ✅ 已有 | 项目设计文档（设计文档、技术栈、架构、实施计划、进度） |

### 文档文件

| 文件 | 来源 | 作用 |
|------|------|------|
| `CLAUDE.md` | 脚手架生成 | Astro 项目 AI 开发指引（命令、技术栈、目录结构） |
| `AGENTS.md` | 脚手架生成 | 自动生成，指向 `CLAUDE.md` |
| `README.md` | 脚手架生成 | 项目 README，待自定义 |
| `memory-bank/CLAUDE.md` | 手动创建 | 项目整体设计文档，面向 AI 开发者 |

### `astro.config.mjs` 配置说明

```js
// @ts-check
import { defineConfig } from 'astro/config';
import tailwindcss from '@tailwindcss/vite';   // Tailwind CSS 4 的 Vite 插件
import mdx from '@astrojs/mdx';                // MDX 内容支持

export default defineConfig({
  integrations: [mdx()],                       // 注册 MDX 集成
  vite: {
    plugins: [/** @type {any} */ (tailwindcss())]  // Tailwind 4 Vite 插件
    // 注意：@type {any} 是因为 @tailwindcss/vite 引用的 Vite 类型
    // 与 Astro 内置 Vite 版本不同，TS 检查时报类型不兼容。
    // 运行时无影响——两个 Vite 版本的 Plugin 接口实际兼容。
  }
});
```
