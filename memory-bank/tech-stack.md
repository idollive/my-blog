# My-blog 技术栈推荐

## 核心原则：简单 + 健壮

> 个人博客的核心需求是**写文章**和**被阅读**，技术栈应服务于内容，而非喧宾夺主。

---

## 推荐方案

| 层级 | 选择 | 为什么 |
|------|------|--------|
| 框架 | **Astro 5** | 专为内容站点设计，零 JS 默认输出，内置 MDX 支持 |
| 样式 | **Tailwind CSS 4** | 原子化 CSS，开发效率高，生产体积小 |
| 数据库 | **MySQL 8** + Prisma | 稳定可靠，Prisma 消除手写 SQL |
| 托管 | **PlanetScale** | Serverless MySQL，免运维，免费 5GB |
| 部署 | **Vercel** | 推送即部署，自动 SSL，全球 CDN |
| 认证 | **账号密码** | 不引入第三方依赖，单用户场景无需 OAuth |
| 评论 | **Giscus** | 基于 GitHub Discussions，零自建成本 |

---

## 为什么不用 Next.js？

| 对比维度 | Astro | Next.js |
|----------|-------|---------|
| 学习曲线 | 低，类 HTML 模板语法 | 高，RSC/Server Action/路由规则复杂 |
| 默认 JS 体积 | 0 KB | ~87 KB（最小 Runtime） |
| MDX 集成 | 一等公民，开箱即用 | 需插件配置 |
| 构建产物 | 纯静态 HTML | Hybrid，默认 SSR |
| 适用场景 | 内容站、博客、文档 | 全栈应用、SaaS、后台系统 |

**结论：博客不需要 React Server Components、Streaming、Suspense 边界。用 Astro 更简单，产出更快。**

---

## 细节说明

### Astro — 用 .astro 写模板，需要交互时再引入框架

```astro
---
// 服务端逻辑，运行时 0 JS
import { getCollection } from 'astro:content'
const posts = await getCollection('blog')
---
<ul>
  {posts.map(p => <li><a href={`/posts/${p.slug}`}>{p.data.title}</a></li>)}
</ul>
```

需要交互（如搜索框、评论区）时，用 Astro Islands 嵌入 React/Vue/Svelte 组件，互不干扰。

### MySQL + Prisma — 类型安全的数据库操作

```prisma
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  slug      String   @unique
  content   String   @db.Text
  status    String   @default("draft")
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

Prisma 自动生成类型，`prisma.post.findMany()` 有完整的智能提示。

### PlanetScale — 不用管数据库运维

- 自动备份、自动扩缩容
- 分支式 Schema 变更（类似 Git，安全迁移）
- 免费 5GB 存储，个人博客绰绰有余

---

## 最小依赖清单

```json
{
  "dependencies": {
    "@astrojs/mdx": "^5.0.0",
    "@astrojs/tailwind": "^5.0.0",
    "@prisma/client": "^6.0.0",
    "@tailwindcss/typography": "^4.0.0",
    "pinyin-pro": "^3.0.0",
    "tailwindcss": "^4.0.0"
  },
  "devDependencies": {
    "astro": "^5.0.0",
    "prisma": "^6.0.0"
  }
}
```

**共 8 个核心依赖**（含 2 个辅助包：typography 排版插件 + pinyin-pro 中文转 slug），项目干净可控。

---

## 与当前设计文档的改动点

| 设计文档原方案 | 推荐方案 | 理由 |
|---------------|---------|------|
| Next.js 14 | Astro 5 | 更简单、更快、更适合博客 |
| App Router | 文件路由（Astro 原生） | 无需理解 Server/Client Component |
| Vercel + 全栈 | Vercel + 纯静态 | 构建时生成 HTML，零服务端运行时 |

数据库（MySQL + Prisma + PlanetScale）保持一致，无需调整。

---

## 总结

> **Astro + Tailwind + MySQL/Prisma + PlanetScale + Vercel**

这套组合的核心逻辑：

1. **Astro** 负责把 Markdown 变成 HTML — 它的唯一职责
2. **Tailwind** 负责样式 — 不写 CSS 文件
3. **Prisma** 负责数据 — 不写 SQL
4. **PlanetScale** 负责数据库 — 不碰服务器
5. **Vercel** 负责部署 — 不碰 Nginx

每个环节都只做一件事，组合起来简单、可维护、出问题一眼定位。
