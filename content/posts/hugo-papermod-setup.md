---
title: "Hugo + PaperMod 博客搭建记录"
date: 2026-06-26
draft: false
categories:
  - 技术
tags:
  - Hugo
  - 博客搭建
  - GitHub Pages
summary: "记录使用 Hugo 和 PaperMod 主题搭建个人博客的完整过程。"
ShowToc: true
---

## 为什么选择 Hugo

在众多静态站点生成器中，Hugo 以其极快的构建速度和灵活的架构脱颖而出。作为一个 Go 语言编写的工具，它不需要复杂的依赖环境，安装简单，构建速度通常在毫秒级别。

## 为什么选择 PaperMod

PaperMod 是目前 Hugo 生态中最受欢迎的主题之一，它提供了：

- 极简的设计风格，专注内容阅读
- 内置暗色模式自动切换
- 全文搜索支持
- 优秀的移动端适配

## 搭建步骤

### 1. 安装 Hugo

```bash
brew install hugo
```

### 2. 创建站点

```bash
hugo new site my-blog
cd my-blog
```

### 3. 安装主题

```bash
git clone --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

### 4. 配置与部署

编辑 `hugo.yaml`，配置导航、社交链接等参数，然后通过 GitHub Actions 自动部署到 GitHub Pages。

## 总结

整个过程非常顺畅，Hugo + PaperMod 的组合非常适合技术博客使用。
