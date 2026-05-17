---
title: 白嫖 GitHub 基础设施搭建博客：Quartz + Obsidian + 自动部署
date: 2026-05-18
tags:
  - quartz
  - obsidian
  - github
  - blog
description: 用 Quartz + Obsidian + GitHub Pages 搭建零成本博客，Git 子模块分离内容与框架，Push 自动部署，Giscus 免费评论区。How to build a free blog with Quartz, Obsidian, and GitHub infrastructure.
---

## 🎯 目标：零成本、低摩擦、能活下来的博客

我之前搭过 N 次博客。Halo、WordPress、Hexo 全试过，每次都是写完「Hello World」就弃坑。问题出在哪？

- **VPS 要钱**，吃灰有罪恶感
- **写作流程重**，打开电脑 → 进后台 → 写 → 发布，链路太长
- **数据不可控**，数据库一丢全没了

所以这次的设计目标是：**手机打开就能写，写完自动发，全部白嫖 GitHub，零运维。**

---

## 🏗️ 整体架构

```
┌─────────────────────────────────────────────────┐
│                   Obsidian                       │
│           (手机上编辑 Markdown)                    │
└────────────────┬────────────────────────────────┘
                 │ git push
                 ▼
┌─────────────────────────────────────────────────┐
│     blog-content (GitHub 仓库 / Obsidian 存储库)   │
│     sorubedo/blog-content                        │
│     · 纯 Markdown 内容                            │
│     · 不含任何构建配置                              │
└────────────────┬────────────────────────────────┘
                 │ repository_dispatch webhook
                 ▼
┌─────────────────────────────────────────────────┐
│     blog (GitHub 仓库 / Quartz 站点)               │
│     sorubedo/blog                                │
│     · content/ → blog-content (git submodule)    │
│     · Quartz 框架 + 配置                           │
└────────────────┬────────────────────────────────┘
                 │ GitHub Actions
                 ▼
┌─────────────────────────────────────────────────┐
│            GitHub Pages                          │
│       sorubedo.github.io/blog                    │
│       · 免费 HTTPS · 全球 CDN                      │
└─────────────────────────────────────────────────┘
```

两个仓库分工明确：一个只管内容，一个只管构建。互不污染。

---

## 📝 第一步：Quartz 初始化

[Quartz](https://quartz.jzhao.xyz/) 是一个基于 Node.js 的静态站点生成器，专门为 Obsidian 风格的 Markdown 设计。开箱支持 wikilink、Callouts、知识图谱、标签系统。

```bash
git clone https://github.com/jackyzha0/quartz.git
cd quartz
npm i
npx quartz create
```

核心文件：

| 文件 | 作用 |
|:--|:--|
| `quartz.config.ts` | 插件配置：Transformer、Filter、Emitter |
| `quartz.layout.ts` | 页面布局：侧边栏、正文区、评论区 |
| `content/` | 文章目录（后面会替换成子模块） |

默认 `content/` 里有示例文章，删掉就行。

## 🔗 第二步：Git 子模块分离内容与框架

这是整个架构最关键的一步。把 `content/` 替换为独立仓库的子模块，实现**内容与框架分离**：

```bash
# 移除默认 content 目录
rm -rf content

# 添加子模块
git submodule add git@github.com:sorubedo/blog-content.git content

# 提交
git add .gitmodules content
git commit -m "Move content to submodule"
```

好处：

- **内容仓库**（`blog-content`）就是 Obsidian Vault 本身，手机端 Obsidian 直接打开这个目录，写完提交
- **框架仓库**（`blog`）只关心 Quartz 版本和配置，不碰文章
- 改主题、升级 Quartz 不会污染文章 commit 历史

> [!note]
> Obsidian 的 `.obsidian/` 配置目录也放在 `blog-content` 里，在 `quartz.config.ts` 中加到 `ignorePatterns` 即可避免被构建。

---

## 🤖 第三步：双触发自动部署

### 内容仓库的触发器

`blog-content/.github/workflows/trigger-rebuild.yml`：

```yaml
name: Trigger blog rebuild
on:
  push:
    branches: [main]

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.DISPATCH_TOKEN }}
          script: |
            await github.rest.repos.createDispatchEvent({
              owner: 'sorubedo',
              repo: 'blog',
              event_type: 'content-updated',
            })
```

每次写完文章 push，就发一个 `repository_dispatch` 事件通知 blog 仓库重建。

> [!warning] 别忘了设置 Token
> `DISPATCH_TOKEN` 需要在 `blog-content` 仓库的 **Settings → Secrets and variables → Actions** 中创建。去 [GitHub Settings → Developer settings → Personal access tokens](https://github.com/settings/tokens) 生成一个 **classic PAT**，勾选 `repo` 范围，把值填进去即可。

### 框架仓库的部署

`blog/.github/workflows/deploy.yml` 的关键步骤：

```yaml
on:
  push:
    branches: [v4]
  repository_dispatch:
    types: [content-updated]   # ← 接收内容仓库的触发

jobs:
  build:
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive  # ← 拉取最新内容
      - name: Update content submodule
        run: |
          git submodule update --init --remote --recursive
          cd content && git checkout main && git pull origin main
      - run: npm ci
      - run: npx quartz build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: public

  deploy:
    needs: build
    steps:
      - uses: actions/deploy-pages@v4
```

两种触发方式都会先拉取最新的 content 子模块，再构建部署。

> [!note] 为什么用两个触发条件？
> `push` 处理框架本身的改动（改配置、升级 Quartz），`repository_dispatch` 处理内容更新。无论哪种变更，站点都会自动重建。

---

## 💬 第四步：Giscus 评论区

[Giscus](https://giscus.app/) 是一个基于 GitHub Discussions 的评论系统。**完全免费，无广告，数据归你所有。**

### 配置步骤

1. **仓库设置**：blog 仓库必须 Public，开启 Discussions 功能（Settings → Features → Discussions）
2. **安装 Giscus App**：去 [github.com/apps/giscus](https://github.com/apps/giscus) 授权
3. **获取配置**：打开 [giscus.app](https://giscus.app)，填入仓库名，选择 Discussion 分类（推荐 Announcements），页面会自动生成你的专属配置参数（以下是示例值，请替换为你自己的）：

```
repo: 你的用户名/你的仓库名
repoId: MDEwOlJlcG9zaXRvcnkzODcyMTMyMDg
category: Announcements
categoryId: DIC_kwDOFxRnmM4B-Xg6
```

4. **修改 `quartz.layout.ts`**，在 `afterBody` 中加入：

```typescript
afterBody: [
  Component.Comments({
    provider: "giscus",
    options: {
      // 替换为 giscus.app 生成的实际值
      repo: "你的用户名/你的仓库名",
      repoId: "MDEwOlJlcG9zaXRvcnkzODcyMTMyMDg",
      category: "Announcements",
      categoryId: "DIC_kwDOFxRnmM4B-Xg6",
      lang: "zh-CN",
      mapping: "pathname",
      strict: true,
      reactionsEnabled: true,
      inputPosition: "bottom",
    },
  }),
],
```

5. **单篇文章控制**：在 frontmatter 中加 `comments: false` 可关闭特定文章的评论区。

---

## 🆓 白嫖清单

| 服务 | 用途 | 费用 |
|:--|:--|:--|
| **GitHub Pages** | 静态站点托管，HTTPS + 全球 CDN | 免费 |
| **GitHub Actions** | CI/CD 自动构建部署 | 公开仓库免费 |
| **GitHub Discussions** | Giscus 评论区后端 | 免费 |

全部跑在 GitHub 上，没有 VPS、没有数据库、没有服务器账单。即使哪天 GitHub 挂了（不太可能），你的 Markdown 文件全在本地，换一个 SSG 重新生成就行。

### 手机端同步：obsidian-git 插件

手机上 Obsidian 怎么把文章推到 GitHub？用社区插件 [obsidian-git](https://github.com/Vinzent03/obsidian-git)。

虽然开发者表示安卓端不在官方支持范围内，但实际使用下来，做**备份、拉取、提交推送**完全没问题，写完后打开菜单点一下就同步上去了，配合前面的 `repository_dispatch` 触发器，全程不用碰 Git 命令。

> [!note]
> 不建议在安卓上用它做复杂的合并/变基操作。日常就是：打开 Obsidian → 写 → 插件推送 → 博客自动更新。简单场景够用了。

---

---

## ✨ 最终效果

我的写作流程现在是这样的：

1. 手机打开 Obsidian → 写文章
2. 提交推送到 `blog-content`
3. GitHub Actions 自动触发 `blog` 仓库构建
4. 一分钟内站点更新，评论区自动就绪

零运维、零成本、零维护负担。这可能是我第一个不会弃坑的博客。

> [!quote]
> 白嫖 GitHub 不算本事，白嫖完还能稳定产出内容才算。
> —— 一个写满五篇文章后感到被掏空的博主 🫡
