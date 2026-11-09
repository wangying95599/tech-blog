# tech-blog

Hugo 静态博客，部署在 GitHub Pages：<https://wangying95599.github.io/tech-blog/>

## 写新文章

在 `content/posts/` 下新建 Markdown 文件，frontmatter 固定四字段：

```markdown
---
title: "文章标题"
date: 2026-11-09
tags: ["标签1", "标签2"]
author: wangying95599
---
```

文件名建议 `YYYY-MM-DD-slug.md`，会直接成为 URL。

## 本地预览

```bash
hugo server -D
```

需要 Hugo extended ≥ 0.158。

## 发布

提 PR 到 `main`，合并后 GitHub Actions 自动构建并部署到 Pages。
