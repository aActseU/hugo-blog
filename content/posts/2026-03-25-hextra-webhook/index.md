---
title: Hextra 主题美化与 Webhook 部署“样式裸奔”踩坑实录"
date: 2026-03-25T10:00:00+08:00
draft: false
categories: ["技术折腾"]
tags: ["Hugo", "Hextra", "AWS", "Webhook", "自动化部署"]
---

最近给在 AWS 云端部署的 Hugo 博客换上了基于 Tailwind CSS 的现代化主题：Hextra。在享受“本地 Push、云端自动发布”丝滑体验的同时，也踩了几个前端配置和后台自动化脚本的坑。特此复盘记录，以备后查。

## 1. Hextra 列表页美化：告别单薄的纯文本流

Hextra 默认的列表页走的是极简风，文章之间分界不够明显。为了提升视觉质感，我做了一些微调：

* **卡片化布局**：在 `assets/css/custom.css` 中注入自定义样式，给 `article` 标签加上了 `padding`、边框、圆角以及悬浮阴影效果，让每篇文章变成一张独立的卡片。
* **文案汉化**：通过新建 `i18n/zh-cn.yaml`，把默认的英文 `readMore` 覆盖成了更符合中文阅读习惯的“阅读全文 ➔”。
* **显示最后修改时间**：在 `hugo.toml` 中开启了 `enableGitInfo = true` 和 `displayUpdated = true`，让页面显得更有厚度，也提升了技术博客的严谨性。

## 2. 解决原生 KaTeX 公式不渲染的问题

Hextra 虽然在主题层面原生支持 KaTeX，但我发现文章里的行内公式（比如 `$\rightarrow$`）并没有被正确渲染成漂亮的箭头，而是以纯文本形式“躺”在页面上。

查阅官方文档后发现，这是因为 Hugo 底层的 Markdown 解析器（Goldmark）默认把这些符号当成了普通文本。需要在 `hugo.toml` 中开启公式透传（passthrough）扩展：

```toml
[markup.goldmark.extensions.passthrough]
  enable = true
  [markup.goldmark.extensions.passthrough.delimiters]
    block = [['\[', '\]'], ['$$', '$$']]
    inline = [['\(', '\)'], ['$', '$']]