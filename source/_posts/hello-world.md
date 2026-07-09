---
title: Hexo 使用指南
categories:
  - Blog
tags:
  - Hexo
  - Static Site
abbrlink: 64968
date: 2026-07-09 00:00:00
---

Hexo 是一个基于 Node.js 的静态博客框架，通过 Markdown 编写文章，利用主题系统将内容渲染为静态 HTML 页面，并部署到服务器或静态网站托管平台。

## 项目结构

一个典型 Hexo 项目结构如下：

```
blog/
├── _config.yml        # Hexo配置文件
├── package.json       # Node依赖配置
├── source/            # 博客源文件
│   ├── _posts/        # Markdown文章
│   └── pages/         # 独立页面
├── themes/            # 博客主题
├── public/            # 生成后的静态文件
└── scaffolds/         # 文章模板
```

## 常用指令

### 快速上手

日常使用 Hexo 的流程通常为：使用 `hexo new` 创建文章，修改 Markdown 内容，通过 `hexo s` 本地预览效果，使用 `hexo clean` 清理缓存，使用 `hexo g` 生成静态文件，最后通过 `hexo d` 部署到服务器。

Hexo 的核心流程可以概括为：`Markdown 内容 → Hexo 渲染 → 静态文件生成 → Web 服务器发布`。掌握文章管理、主题配置和部署流程后，即可长期维护一个个人技术博客。

### 创建

创建新的博客文章可以使用 `hexo new "文章标题"` 命令，生成的 Markdown 文件默认位于 `source/_posts/` 目录。文章通常包含 Front Matter 配置，用于设置标题、发布时间、分类和标签，例如 `title`、`date`、`categories` 和 `tags` 等字段。

### 启动服务器

启动 Hexo 本地服务器以预览博客效果，可以使用 `hexo server` 命令（简写：`hexo s`）。Hexo 会将 Markdown 文件渲染为静态 HTML/CSS/JavaScript 文件，执行 `hexo generate` 命令（简写：`hexo g`）后会生成 `public/` 目录，该目录可以直接部署到 Web 服务器。如果修改主题或配置后页面没有更新，可以执行 `hexo clean` 清理缓存，然后重新生成静态文件。

### 页面

Hexo 支持创建独立页面，例如使用 `hexo new page "about"` 创建关于页面，生成文件位于 `source/about/index.md`，访问路径通常为 `/about/`。除了文章页面外，常见页面还包括个人介绍、项目展示、友情链接和归档页面。

### 主题

Hexo 主题存放于 `themes/` 目录，可以通过修改 `_config.yml` 中的 `theme` 字段切换主题。主题自身的配置文件通常位于 `themes/主题名称/_config.yml`，不同主题提供不同的配置选项。如果需要修改字体、颜色、布局等样式，通常需要修改主题中的 CSS 文件，推荐通过自定义 CSS 覆盖主题样式，而不是直接修改主题源码，以避免主题升级导致修改丢失。

### 部署

Hexo 部署的本质是上传生成后的静态文件。常见方式包括 Git 部署和服务器部署，其中服务器部署通常使用 Nginx 提供静态文件访问。生成博客文件后，将 `public/` 目录上传到服务器指定目录，并通过 Nginx 配置域名访问即可。



