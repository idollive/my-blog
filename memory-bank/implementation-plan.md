# My-blog 实施计划

## 概述

本文档将 My-blog 拆解为一系列小而具体的步骤，每一步都包含验证测试。计划聚焦于**基础可用**——先让博客能写、能看、能部署，后续再叠加搜索、评论、归档等进阶功能。

每条指令面向 AI 开发者：精确、可操作、不包含代码。

## 架构关键决策

### 部署与工作流模型：本地管理 + 静态部署

```
┌──────────────────────────────────────────────────────┐
│  本地开发环境 (astro dev)                              │
│  ┌─────────────┐    ┌──────────────┐                 │
│  │ /admin 后台  │───▶│ MySQL/Prisma  │ 读写数据库       │
│  │ 创建/编辑文章 │    │ (PlanetScale) │                 │
│  └─────────────┘    └──────────────┘                 │
│         │                                            │
│         │ 同时写入                                    │
│         ▼                                            │
│  ┌─────────────────┐                                │
│  │ src/content/blog/│  *.mdx 文章文件                │
│  └─────────────────┘                                │
└──────────────────────────────────────────────────────┘
         │
         │ git commit & push
         ▼
┌──────────────────────────────────────────────────────┐
│  GitHub + Vercel                                     │
│  ┌──────────────┐    ┌──────────────┐               │
│  │ 源码 (含.mdx) │───▶│ astro build   │ 纯静态 HTML   │
│  └──────────────┘    └──────────────┘               │
│  不需要 DATABASE_URL     读取文件，不连数据库          │
└──────────────────────────────────────────────────────┘
```

**核心逻辑：**
- **后台只在本地使用** — `astro dev` 时通过 `/admin` 面板增删改文章，操作数据库的同时将 MDX 文件写入 `src/content/blog/`
- **数据库只存元数据**（title、slug、excerpt、status、category、tags 等），正文仅存 MDX 文件中。`Post.content` 字段不在数据库中——文件是正文的唯一数据源
- **前台页面全部从文件读取** — 通过 Astro 内容集合 API（`getCollection`），构建时不连数据库
- **部署即编译** — `git push` → Vercel 运行 `astro build` → 输出纯静态 HTML → CDN 分发。Vercel 不需要配置 `DATABASE_URL`（前台无运行时数据库查询）

### 数据存储边界

| 存储位置 | 存什么 | 不存什么 |
|----------|--------|----------|
| MySQL（PlanetScale） | Post 元数据（title/slug/excerpt/status/categoryId/viewCount）、Category、Tag、关联关系 | **正文内容**（正文仅存文件） |
| MDX 文件（`src/content/blog/`） | 文章正文 + frontmatter（标题/日期/摘要/分类/标签） | 阅读量等运行时统计 |
| `.env` / `src/config.ts` | 站点设置（标题/描述/Logo）、管理员密码哈希 | — |

> **为什么数据库不存正文？** 正文已经在 MDX 文件中，且 Astro 内容集合直接读取文件。数据库存正文是冗余，且会导致"文件和数据库哪个是数据源"的同步问题。一台机器上只有一套文件系统，单一数据源 = 零同步成本。搜索功能（后续阶段）可以通过 MySQL FULLTEXT 索引文件导入，不影响当前设计。

---

## 阶段 A：项目初始化

### 步骤 1：创建 Astro 项目

**目标：** 在本地得到一个可运行的 Astro 5 空项目。

**指令：**
1. 使用 `create astro` 官方 CLI 在当前目录创建项目
2. 选择以下选项：空项目模板、不安装示例代码、TypeScript 严格模式
3. 项目名称为 `my-blog`

**测试：** 运行 `astro dev`，在浏览器打开 `http://localhost:4321`，看到 Astro 默认欢迎页即通过。

---

### 步骤 2：安装核心依赖

**目标：** 安装设计文档指定的全部核心依赖，版本锁定在主版本。

**指令：**
1. 安装运行时依赖：`@astrojs/mdx`（^5.x）、`@astrojs/tailwind`（^5.x）、`@prisma/client`（^6.x）、`tailwindcss`（^4.x）、`@tailwindcss/typography`（^4.x）、`pinyin-pro`（^3.x，用于中文标题转 slug）
2. 安装开发依赖：`prisma`（^6.x）
3. 安装后运行 `npx astro add mdx` 和 `npx astro add tailwind` 让 Astro 自动生成配置文件

**测试：** 检查 `package.json` 中 dependencies 包含上述 6 个运行时包（`@astrojs/mdx`、`@astrojs/tailwind`、`@prisma/client`、`tailwindcss`、`@tailwindcss/typography`、`pinyin-pro`），devDependencies 包含 `prisma`。运行 `npx astro check` 不报错。

---

### 步骤 3：初始化 Prisma

**目标：** Prisma CLI 可用，生成基础配置文件。

**指令：**
1. 运行 `npx prisma init` 生成 `prisma/schema.prisma` 和 `.env` 文件
2. 将 `.env` 中的 `DATABASE_URL` 占位符替换为 PlanetScale 连接字符串格式（实际值后续步骤填写）
3. 在 `.env` 中添加 `ADMIN_PASSWORD_HASH` 键，初始值填写占位文本（实际哈希值在步骤 19 生成并替换）

**测试：** 项目根目录存在 `prisma/schema.prisma` 和 `.env` 两个文件。`.env` 中包含 `DATABASE_URL` 和 `ADMIN_PASSWORD_HASH` 两个键。

---

### 步骤 4：初始化 Git 仓库

**目标：** 建立版本控制，连接远程仓库。

**指令：**
1. 初始化 Git 仓库
2. 创建 `.gitignore` 文件，忽略 `node_modules/`、`.env`、`dist/`、`.astro/`
3. 暂存所有文件并创建初始提交，提交信息为 "init: scaffold Astro project with Prisma"
4. 在 GitHub 上创建同名远程仓库（`my-blog`），将本地仓库推送到远程

**测试：** 运行 `git log` 显示一条提交记录。运行 `git remote -v` 显示正确的 GitHub 远程地址。

---

## 阶段 B：数据库

### 步骤 5：创建 PlanetScale 数据库

**目标：** PlanetScale 上存在一个可用数据库，获取连接字符串。

**指令：**
1. 登录 PlanetScale 控制台（或使用 `pscale` CLI）
2. 创建新数据库，命名为 `my-blog`
3. 选择区域为亚太（ap-southeast 或 ap-northeast，视用户位置而定）
4. 创建后进入数据库设置，生成连接字符串
5. 将连接字符串写入 `.env` 的 `DATABASE_URL`

**测试：** 在 PlanetScale 控制台中数据库状态显示为 "ready"。复制连接字符串后，在终端运行 `npx prisma db pull` 不报错（此时数据库为空，应成功连接但无表）。

---

### 步骤 6：编写 Prisma Schema

**目标：** 定义完整的数据库模型，匹配设计文档中的数据模型。

**指令：**
1. 在 `prisma/schema.prisma` 中将 provider 改为 `mysql`
2. 定义 `Post` 模型，字段为：id（自增整数主键）、title（字符串）、slug（字符串，唯一索引）、excerpt（可选字符串）、coverImage（可选字符串）、status（字符串，默认 "draft"）、viewCount（整数，默认 0）、createdAt（DateTime，默认 now()）、updatedAt（DateTime，自动更新）
   - **注意：Post 模型不含 content 字段。** 文章正文仅存储在 MDX 文件中，数据库只存元数据。
3. 定义 `Category` 模型，字段为：id（自增整数主键）、name（字符串）、slug（字符串，唯一索引）
4. 定义 `Tag` 模型，字段为：id（自增整数主键）、name（字符串）、slug（字符串，唯一索引）
5. 定义 `PostTag` 中间表，字段为：postId + tagId 联合主键，分别建立到 Post 和 Tag 的外键关系
6. 给 Post 模型添加 categoryId 字段（可选整数），建立到 Category 的外键关系

**测试：** 运行 `npx prisma format` 不报错。运行 `npx prisma validate` 输出 schema 有效。

---

### 步骤 7：推送 Schema 到数据库

**目标：** PlanetScale 数据库中的表结构与 Prisma Schema 一致。

**指令：**
1. 运行 `npx prisma db push` 将 schema 推送到 PlanetScale
2. 推送完成后，在 PlanetScale 控制台查看数据库分支，确认表已创建

**测试：** 在 PlanetScale 控制台的 "Branches" 页面中看到 4 张表（Post、Category、Tag、PostTag）。运行 `npx prisma studio`，浏览器打开后可浏览空表。

---

### 步骤 8：生成 Prisma Client 并验证连接

**目标：** Prisma Client 可从 Astro 代码中导入并查询数据库。

**指令：**
1. 运行 `npx prisma generate` 生成类型安全的客户端
2. 在 `src/lib/` 目录下创建 `prisma.ts` 文件，导出一个全局单例 PrismaClient 实例
3. 在 `src/pages/index.astro` 的前置脚本中导入该客户端，调用 `prisma.post.count()`，将返回值打印到控制台（`console.log`）

**测试：** 运行 `astro dev`，打开 `http://localhost:4321`，在终端输出中看到 `0`（表示成功连接数据库并查询到 0 条 Post 记录）。无报错即通过。验证后移除该临时代码。

---

## 阶段 C：内容集合与基础布局

### 步骤 9：配置 Astro 内容集合

**目标：** Astro 知道 `src/content/blog/` 下的 MDX 文件属于 `blog` 集合，并进行类型校验。

**指令：**
1. 创建 `src/content/blog/` 目录
2. 创建 `src/content/config.ts` 文件
3. 在其中使用 `defineCollection` 定义名为 `blog` 的集合，类型为 `mdx`（从 `astro:content` 导入）
4. 为 `blog` 集合定义 frontmatter schema，字段为：title（必填字符串）、slug（必填字符串）、excerpt（可选字符串）、coverImage（可选字符串）、status（可选字符串，默认 "draft"）、category（可选字符串）、tags（可选字符串数组）、createdAt（必填日期）
5. 导出 collections 对象

**测试：** 运行 `astro dev`，终端不报 schema 相关错误。在 `src/content/blog/` 中放入一个 `.mdx` 文件后，执行 `astro check` 不报类型错误。

---

### 步骤 10：创建第一篇示例文章

**目标：** 有一篇真实的 MDX 文章可用于后续开发和测试。

**指令：**
1. 在 `src/content/blog/` 下创建文件 `hello-world.mdx`
2. Frontmatter 填写：title 为 "你好，世界"、slug 为 "hello-world"、excerpt 为 "这是我的第一篇博客文章"、status 为 "published"、category 为 "技术"、tags 为 ["intro"]、createdAt 为当天日期
3. 正文写 3 段 Markdown 内容，至少包含一个二级标题、一个有序列表、一段代码块
4. 在 `src/content/blog/` 下再创建 `draft-post.mdx`，status 设为 "draft"，正文任意

**测试：** 在 `src/pages/index.astro` 的前置脚本中调用 `getCollection('blog')`，打印返回的数组长度。运行 `astro dev`，终端输出 `2`。验证后移除临时代码。

---

### 步骤 11：创建基础布局组件

**目标：** 所有页面共享同一个 HTML 骨架（头部导航 + 内容区 + 页脚）。

**指令：**
1. 创建 `src/layouts/BaseLayout.astro` 文件
2. 该组件接收一个 `title` prop（字符串）
3. HTML 结构包含：`<html>`、`<head>`（含 meta charset、viewport、title 使用传入的 prop、link 引入全局 CSS）、`<body>`（包含 `<header>` 占位导航栏、`<main>` slot 用于插入页面内容、`<footer>` 占位页脚）
4. 在 `<header>` 中放置站点名称 "My-blog" 和三个导航链接：首页（/）、归档（/archive）、关于（/about）
5. 创建 `src/styles/globals.css`，在 BaseLayout 中通过 link 标签引入
6. 在 `astro.config.mjs` 中确认 Tailwind 集成已配置

**测试：** 创建临时页面 `src/pages/test-layout.astro`，使用 BaseLayout 包裹一段文字。运行 `astro dev`，打开 `http://localhost:4321/test-layout`，看到完整的 HTML 文档结构、导航栏中显示站点名和三个链接、页脚占位文字。验证后删除临时页面。

---

### 步骤 12：配置 Tailwind 基础样式

**目标：** 全局 CSS 中引入 Tailwind 指令，排版样式生效。

**指令：**
1. 在 `src/styles/globals.css` 中添加 `@import "tailwindcss"` 和 `@plugin "@tailwindcss/typography"` 指令（Tailwind v4 的插件语法）
2. 在 CSS 中定义少量 CSS 自定义属性（design tokens）：正文字体、标题字体、主色调、背景色、文字色各一组默认值

**测试：** 在某个临时测试页面中使用 Tailwind 工具类（如 `text-2xl font-bold text-blue-600`），运行 `astro dev` 在浏览器确认样式生效。使用 `prose` 类包裹一段 Markdown 渲染内容，确认排版样式生效。验证后删除临时页面。

---

## 阶段 D：前台页面

### 步骤 13：首页 — 已发布文章列表

**目标：** 访问 `/` 看到已发布文章的标题列表，草稿不显示。

**指令：**
1. 编辑 `src/pages/index.astro`
2. 在前置脚本中调用 `getCollection('blog')` 获取全部文章
3. 过滤出 `status === 'published'` 的文章
4. 按 `createdAt` 降序排列
5. 模板中遍历文章数组，每篇文章渲染：标题（链接到 `/posts/[slug]`）、发布日期、摘要、分类标签
6. 使用 BaseLayout 包裹，title 为站点名
7. 处理空状态：当已发布文章数为 0 时，显示 "暂无文章" 提示

**测试：** 运行 `astro dev`，访问 `http://localhost:4321`。页面上显示 1 篇文章（hello-world），不显示 draft-post。标题可点击。文章按日期排序。

---

### 步骤 14：首页 — 分页

**目标：** 当文章超过一定数量时，首页自动分页。

**指令：**
1. 在首页前置脚本中，从 URL 查询参数读取当前页码（默认第 1 页）
2. 使用 Astro 的 `paginate` 函数或手动切片，每页显示 10 篇文章
3. 模板底部渲染分页导航：上一页 / 下一页链接，当前页码 / 总页数显示
4. 第 1 页时不显示 "上一页"；最后一页时不显示 "下一页"

**测试：** 用当前 1 篇文章验证分页不崩溃。创建 12 篇临时文章（通过复制 MDX 文件并修改 slug 和 title），确认第 1 页显示 10 篇、第 2 页显示 2 篇，分页链接正确。验证后删除临时文章，保留原始的 2 篇。

---

### 步骤 15：文章详情页

**目标：** 访问 `/posts/[slug]` 看到完整文章内容。

**指令：**
1. 创建文件 `src/pages/posts/[slug].astro`
2. 使用 `getStaticPaths` 从 `getCollection('blog')` 获取所有文章的 slug，返回路径参数
3. 在前置脚本中通过 `getEntryBySlug('blog', slug)` 获取当前文章
4. 如果文章不存在，返回 404 页面
5. 使用 `render()` 方法渲染 MDX 内容到 HTML
6. 集成代码语法高亮：安装并配置 `rehype-pretty-code` 或类似 Shiki 集成，确保 Markdown 代码块在渲染时自动高亮。在 Astro 配置中注册 rehype 插件
7. 从文章标题层级（h2、h3）自动生成目录（TOC）：在渲染后的 HTML 中提取二级和三级标题，在文章内容右侧（桌面端）或顶部（移动端）渲染为锚点链接列表
8. 模板中显示：文章标题、发布日期、分类、标签、封面图（如有）、目录（TOC）、渲染后的文章正文（包裹在 `prose` 容器中，代码块应有语法高亮）
9. 使用 BaseLayout 包裹，title 为文章标题

**测试：** 访问 `/posts/hello-world`，看到完整文章内容，包括标题、日期、文章目录（从正文标题自动生成）、带语法高亮的代码块、Markdown 渲染后的正文。访问 `/posts/non-existent`，看到 404 页面。访问 `/posts/draft-post`，内容正常显示（注：前台不区分草稿/已发布是预期的——详情页的访问控制后续由构建时过滤处理）。

---

### 步骤 16：关于页面

**目标：** 访问 `/about` 看到一个静态介绍页。

**指令：**
1. 创建文件 `src/pages/about.astro`
2. 使用 BaseLayout 包裹，title 为 "关于"
3. 写一段占位文字：个人介绍、技术方向、联系方式占位
4. 样式使用 Tailwind 排版类（`prose`），居中单栏布局

**测试：** 访问 `http://localhost:4321/about`，看到关于页面内容，导航栏高亮 "关于" 链接。

---

### 步骤 17：归档页面

**目标：** 访问 `/archive` 看到文章按年月分组的列表。

**指令：**
1. 创建文件 `src/pages/archive.astro`
2. 获取所有已发布文章，按 `createdAt` 降序排列
3. 将文章按年份分组（使用 `createdAt.getFullYear()`）
4. 模板中渲染：年份标题 → 该年文章按月份分组 → 每月文章列表
5. 每篇文章显示：标题（链接到详情页）、发布日期（MM-DD 格式）
6. 空状态处理：无已发布文章时显示 "暂无归档"

**测试：** 访问 `/archive`，看到文章按年月分组显示。hello-world 出现在正确月份下。

---

### 步骤 18：构建静态站点

**目标：** `astro build` 生成纯静态 HTML 文件。

**指令：**
1. 运行 `npx astro build`
2. 检查 `dist/` 目录输出
3. 确认每个路由都生成了对应的 HTML 文件
4. 确认 `dist/` 中没有服务端代码或 JS bundle（除 Astro 必要的少量客户端脚本）

**测试：** 运行 `npx astro build` 不报错。`dist/` 目录下存在 `index.html`、`about/index.html`、`archive/index.html`、`posts/hello-world/index.html`。用浏览器直接打开 `dist/index.html`（通过 `file://` 协议）能正常渲染页面。用 `npx serve dist` 在本地预览，所有链接跳转正常。

---

## 阶段 E：管理后台

### 步骤 19：创建简单的认证中间件

**目标：** `/admin/*` 路径受密码保护，未登录用户被重定向。

**指令：**
1. 创建 `src/middleware.ts`（或 Astro 中间件文件）
2. 在中间件中检查请求路径是否以 `/admin` 开头
3. 检查请求 cookie 中是否存在 `auth_token`
4. 如果不存在，返回 302 重定向到 `/admin/login`
5. 认证逻辑：使用 Node.js 内置 `crypto.scryptSync` 对用户输入的密码进行哈希，与 `.env` 中的 `ADMIN_PASSWORD_HASH` 对比。`ADMIN_PASSWORD_HASH` 的格式为 `salt:hash`（hex 编码），由 `crypto.randomBytes(16)` 生成 salt，`crypto.scryptSync(password, salt, 64)` 生成哈希
6. 创建 `src/pages/admin/login.astro`，包含一个简单的密码输入表单，POST 到自身
7. 在登录处理逻辑中，使用 `crypto.scryptSync` 验证明文密码是否与 `.env` 中的哈希值匹配
8. 登录成功后设置 httpOnly cookie（值为随机生成的 session token，非密码哈希），有效期 7 天，重定向到 `/admin`
9. 提供一个脚本（如 `scripts/hash-password.mjs`），用于生成 `ADMIN_PASSWORD_HASH` 值：读取命令行参数中的明文密码，输出 salt:hash 格式的字符串，写入 `.env`

**测试：** 运行 `node scripts/hash-password.mjs "yourpassword"` 生成哈希值并更新 `.env`。未登录状态下访问 `http://localhost:4321/admin`，自动重定向到 `/admin/login`。输入正确密码后跳转到 `/admin`。清除 cookie 后再访问 `/admin`，再次被重定向。输入错误密码留在登录页并显示错误提示。验证 `.env` 中的不是明文密码。

---

### 步骤 20：管理后台布局

**目标：** 后台页面共享管理布局，与前台视觉区分。

**指令：**
1. 创建 `src/layouts/AdminLayout.astro`
2. 包含侧边栏导航：文章管理、新建文章、站点设置
3. 顶部栏显示当前登录用户名（固定为 "Admin"）和退出按钮
4. 退出按钮链接到 `/admin/logout`，清除 auth_token cookie 并重定向到首页
5. 后台使用更紧凑的无衬线字体样式，背景为浅灰色，侧边栏深色

**测试：** 登录后访问 `/admin`，看到侧边栏和顶部栏。点击 "退出" 后 cookie 被清除，跳转到首页。

---

### 步骤 21：后台文章列表

**目标：** `/admin/posts` 以表格形式列出全部文章（含草稿），可快速切换文章状态。

**指令：**
1. 创建 `src/pages/admin/posts/index.astro`
2. 从数据库查询全部文章（使用 Prisma），按更新时间降序排列，包含关联的 Category 数据
3. 表格列：标题、状态（草稿/已发布徽章）、分类、更新日期、操作（编辑按钮、切换状态按钮）
4. "切换状态" 按钮提交 POST 表单到 `/admin/posts/[id]/toggle-status`
5. 创建对应的 API 端点 `/admin/posts/[id]/toggle-status`：接收 POST 请求，验证认证状态，切换 Post 记录的 status 字段，重定向回文章列表
6. 文章标题链接到编辑页

**测试：** 访问 `/admin/posts`，表格显示 2 篇文章（hello-world 和 draft-post）。hello-world 显示 "已发布" 徽章，draft-post 显示 "草稿" 徽章。点击 draft-post 的切换按钮，该文章状态变为 "已发布"，首页现在显示 2 篇文章。再次切换回草稿状态，首页恢复显示 1 篇。

---

### 步骤 22：新建文章页

**目标：** 通过 Web 表单创建新文章，MDX 文件写入 `src/content/blog/`，元数据写入数据库。

**指令：**
1. 创建 `src/pages/admin/posts/new.astro`
2. 表单字段：标题（文本输入）、slug（文本输入，自动从标题生成——使用 `pinyin-pro` 库将中文标题转为拼音 slug，也可手动修改）、分类（下拉选择，从数据库 Category 表读取）、标签（多选或逗号分隔输入）、摘要（文本域）、状态（草稿/已发布 单选）、正文（大文本域，用于输入 Markdown 内容）
3. 表单 POST 到自身或 `/admin/posts/create` API 端点
4. 后端处理逻辑（在 API 端点或页面 script 中）：
   - 验证认证状态
   - 校验必填字段（title、slug、content）不为空
   - 校验 slug 在 Post 表中不重复
   - 将 MDX 正文写入 `src/content/blog/[slug].mdx`（包含 frontmatter 和正文）
   - 在数据库 Post 表中创建对应记录（仅元数据：title、slug、excerpt、status、categoryId 等，**不存正文**）
   - 处理分类：如果输入的分类不在 Category 表中，自动创建
   - 处理标签：如果标签不在 Tag 表中，自动创建；创建对应的 PostTag 关联
5. 创建成功后重定向到 `/admin/posts`，显示成功提示
6. 创建失败时回到表单页，保留已输入内容，显示错误信息

**测试：** 访问 `/admin/posts/new`，填写标题 "测试文章"、slug "test-post"、正文 "## 这是一篇测试"，选择状态 "已发布"，提交。检查 `src/content/blog/` 下生成了 `test-post.mdx`。数据库中 Post 表新增一条记录。访问 `/posts/test-post` 能看到文章内容。首页列表中出现该文章。

---

### 步骤 23：编辑文章页

**目标：** 修改已有文章的标题、内容、分类、标签和状态。

**指令：**
1. 创建 `src/pages/admin/posts/[id]/edit.astro`
2. 通过路由参数中的 id 从数据库查询文章，同时查出关联的分类和标签
3. 表单字段与新建页相同，但各字段预填当前值
4. 表单 POST 到 `/admin/posts/[id]/update` API 端点
5. 后端逻辑：
   - 验证认证状态
   - 更新数据库中 Post 记录（仅元数据字段：title、slug、excerpt、status、categoryId，**不包含 content**）
   - 更新 `src/content/blog/[slug].mdx` 文件内容（正文的唯一数据源）。如果 slug 变化，需要同时删除旧 MDX 文件、创建新 MDX 文件
   - 同步标签关联（移除旧关联，创建新关联）
6. 更新成功后重定向到编辑页自身，显示 "已保存" 提示
7. 在编辑页底部放一个删除按钮（红色，需二次确认），POST 到 `/admin/posts/[id]/delete` API 端点。删除逻辑：验证认证 → 从数据库删除 Post 记录 → 删除对应的 `src/content/blog/[slug].mdx` 文件 → 重定向到 `/admin/posts`

**测试：** 编辑 hello-world 文章，修改标题为 "你好，世界（已更新）"，保存。访问 `/posts/hello-world`，看到新标题。修改 slug 为 "hello-world-updated"，保存。确认 `src/content/blog/hello-world.mdx` 不再存在，`hello-world-updated.mdx` 已创建。访问 `/posts/hello-world` 返回 404，`/posts/hello-world-updated` 正常显示。删除测试文章，确认文件和数据库记录均被移除。

---

### 步骤 24：站点设置页

**目标：** 可在后台修改站点标题、描述等全局信息。

**指令：**
1. 创建 `src/pages/admin/settings.astro`
2. 表单字段：站点标题、站点描述、Logo URL（或上传占位）
3. 设置数据存储在 `.env` 或一个新创建的 `src/config.ts` 文件中（简单方案：写入 `src/config.ts` 作为导出常量）
4. 首次实现使用文件存储：`src/config.ts` 导出 `SITE_TITLE`、`SITE_DESCRIPTION`、`SITE_LOGO` 三个常量
5. 后台读取这些值预填表单，保存时通过后端接口覆写 `src/config.ts` 文件内容
6. 修改 BaseLayout 和 index.astro 中的站点标题引用，改为从 `src/config.ts` 导入

**测试：** 在后台设置页修改站点标题为 "我的技术博客"，保存。访问首页，导航栏和浏览器标签页标题显示 "我的技术博客"。再次进入设置页，表单预填修改后的值。

---

## 阶段 F：部署

### 步骤 25：Vercel 项目创建与关联

**目标：** Vercel 项目与 GitHub 仓库关联，推送代码自动触发部署。

**指令：**
1. 登录 Vercel 控制台（vercel.com）
2. 点击 "New Project"，导入 GitHub 上的 `my-blog` 仓库
3. 框架自动检测为 Astro（或手动选择）
4. 构建命令设置为 `astro build`
5. 输出目录设置为 `dist`
6. 暂不部署（等下一步配置完成）
   - **注意：** Vercel 不需要配置 `DATABASE_URL` 或 `ADMIN_PASSWORD_HASH` 环境变量。Vercel 只负责前台页面的纯静态构建（从 MDX 文件读取），不涉及数据库操作。后台管理仅在本地 `astro dev` 时使用。

**测试：** Vercel 项目详情页显示正确的构建命令（`astro build`）和输出目录（`dist`）。

---

### 步骤 26：配置 Astro 适配器

**目标：** Astro 构建产物与 Vercel 平台兼容。

**指令：**
1. 安装 `@astrojs/vercel` 适配器
2. 在 `astro.config.mjs` 中配置该适配器，模式设为 `static`（纯静态输出）
3. 确认 `site` 字段填写为 Vercel 分配的域名（或自定义域名）

**测试：** 运行 `npx astro build`，输出目录 `dist/` 中包含 Vercel 需要的配置文件（如 `.vercel/output/config.json` 或 `vercel.json`）。

---

### 步骤 27：首次部署

**目标：** 网站可通过公网 URL 访问。

**指令：**
1. 确保本地 `astro build` 通过
2. 将全部更改提交并推送到 GitHub 主分支
3. 在 Vercel 中手动触发首次部署（或等待自动触发）
4. 部署完成后，在 Vercel 的 Domains 设置中查看分配的 URL（格式为 `*.vercel.app`）
5. 访问该 URL，确认首页正常加载
6. 访问 `/about`、`/archive`、`/posts/hello-world` 确认全部可用

**测试：** 通过 Vercel 分配的 URL 访问网站，所有前台页面正常显示且加载速度正常（静态 HTML 无加载延迟）。使用移动端浏览器访问，确认响应式布局正常。

---

### 步骤 28：绑定自定义域名（可选）

**目标：** 使用自己的域名访问博客。

**指令：**
1. 在域名注册商（如 Cloudflare、Namecheap）的 DNS 设置中添加记录
2. 类型为 CNAME，指向 Vercel 分配的域名（`cname.vercel-dns.com`）
3. 或使用 A 记录指向 Vercel 的 IP 地址
4. 在 Vercel 项目的 Domains 设置中添加自定义域名
5. 等待 DNS 传播（通常几分钟到几小时）
6. 确认 Vercel 自动为自定义域名配置 SSL 证书

**测试：** 通过自定义域名访问网站，浏览器地址栏显示锁形图标（SSL 已生效）。HTTP 请求自动重定向到 HTTPS。`curl -I` 检查返回 200 且包含 HSTS 头（如配置）。

---

## 后续阶段（基础版不做）

以下功能在基础版完成后按需实施：

- **全文搜索**（/search）：MySQL FULLTEXT 索引 + 搜索接口 + 前端搜索框（需将 MDX 正文内容导入数据库）
- **评论系统**：集成 Giscus（GitHub Discussions）
- **统计**：集成 Umami 或 Simple Analytics
- **分类/标签管理页面**：后台独立管理分类和标签（编辑名称、删除、合并），当前 v1 仅支持创建文章时自动创建
- **标签/分类筛选**：首页增加按标签或分类过滤的能力
- **RSS/Atom Feed**：生成订阅源
- **站点地图**：sitemap.xml 自动生成
- **SEO Meta 标签**：Open Graph、Twitter Card 等
- **图片优化**：Astro Image 集成
- **加载动画**：页面过渡效果
- **MDX 编辑器增强**：WYSIWYG 或实时预览面板

---

## 测试原则

贯穿所有步骤的测试方法：

1. **每步必测** — 绝不跳过步骤末尾的验证测试
2. **优先手动验证** — 打开浏览器确认页面渲染正确，不依赖自动化测试框架
3. **回归检查** — 完成新步骤后，回到前几步确认之前正常的功能仍然正常
4. **提交粒度** — 每一步通过测试后立即提交，提交信息格式为 `feat: [步骤号] 简短描述`
5. **失败回退** — 任一步骤的测试不通过时，修复后再继续，不带着已知问题前进
