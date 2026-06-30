# My-blog 设计文档

## 项目意图

个人技术博客。核心任务：**写 MDX 文章 → 发布 → 读者访问**。不做复杂功能，优先保证写作体验和阅读性能。

## 技术栈

```
Astro 5          — 框架，构建纯静态 HTML，零 JS 默认输出
Tailwind CSS 4   — 样式
MDX              — 文章格式（Markdown + 组件）
Prisma + MySQL   — 数据库（PlanetScale 托管）
Vercel           — 部署，推送即上线
```

选择 Astro 而非 Next.js 的理由：博客是内容站点，不需要 SSR/Server Components。Astro 默认输出零 JS 的 HTML，需要交互时再嵌入 React 组件（Islands 模式）。

## 页面结构

```
/                 文章列表，分页，按分类/标签筛选
/posts/[slug]     文章详情，MDX 渲染 + 代码高亮 + 目录
/archive          按年月归档
/about            个人介绍
/search           全文搜索（MySQL FULLTEXT）
```

## 后台（/admin）

```
/admin/posts              文章列表，CRUD，草稿/发布切换
/admin/posts/new          新建文章（MDX 编辑器）
/admin/posts/[id]/edit    编辑文章
/admin/settings           站点标题、描述、Logo
```

认证：单用户账号密码，NextAuth.js 或直接 Astro 中间件处理。

## 数据模型

```
Post
  id, title, slug(unique), content(MDX), excerpt, coverImage
  status(draft|published), viewCount, createdAt, updatedAt
  categoryId → Category
  tags → PostTag → Tag

Category: id, name, slug
Tag: id, name, slug
```

## 目录结构

```
src/
├── content/blog/          *.mdx 文章放这里（Astro 内容集合）
├── pages/
│   ├── index.astro        首页
│   ├── posts/[slug].astro 文章详情
│   ├── archive.astro      归档
│   ├── about.astro        关于
│   └── admin/             后台页面
├── components/            可复用组件（React Islands 按需）
├── lib/
│   ├── prisma.ts          数据库客户端
│   └── utils.ts
└── styles/globals.css
prisma/schema.prisma
```

## 关键约定

1. **文章即文件** — MDX 放在 `src/content/blog/`，用 Astro 内容集合 API 读取，不存数据库
2. **数据库只管元数据** — 文章的 title/slug/category/tags 等结构化字段存 MySQL，方便筛选和搜索
3. **静态优先** — 所有前台页面构建时生成 HTML，零服务端计算
4. **交互按需加载** — 搜索框、评论区等才用 JS，不污染静态页面

## 不做的事情

- ❌ 用户系统 / 多用户
- ❌ 评论自建（用 Giscus，基于 GitHub Discussions）
- ❌ 统计分析自建（用 Umami 或 Simple Analytics）
- ❌ 主题切换（初版只做一种颜色方案）
- ❌ i18n
- ❌ 复杂动画
