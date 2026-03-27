---
title: Hextra 主题美化踩坑实录 2
date: 2026-03-25T14:00:00+08:00
draft: false
categories: ["技术折腾"]
tags: ["Hugo", "Hextra", "AWS", "Webhook", "自动化部署"]
---

最近给在 AWS 云端部署的 Hugo 博客换上了基于 Tailwind CSS 的现代化主题：Hextra。在享受“本地 Push、云端自动发布”丝滑体验的同时，也踩了几个前端配置和后台自动化脚本的坑。特此复盘记录，以备后查。这次主要是 Webhook 部署“样式裸奔”的问题。

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
```
配置加上后，引擎就能精准捕捉到公式定界符，数学公式成功复活。

## 3. Webhook 自动化部署的终极痛点：样式“裸奔”

这是本次折腾最费脑筋的一个坑。

**症状表现**：每次 Git Push 之后，立刻访问云服务器 IP，会发现网页样式全部丢失（俗称“裸奔”）。但如果我通过 SSH 连上服务器，手动执行一次 `deploy.sh`，再访问网站就一切正常。

一开始以为是部署的“时间差”导致的——原来的脚本简单粗暴地使用了 `rm -rf` 清空网站目录再让 Hugo 重新生成，导致浏览器在真空期抓不到 CSS 文件。于是我将部署脚本升级成了 `rsync` 平滑替换：

```bash
# 放弃 rm -rf，使用 rsync 实现无缝同步，消除真空期
rsync -av --delete public/ /var/www/blog/
```

**真正的原因**：换了 `rsync` 后依然裸奔，最终锁定了真凶——**Webhook 后台运行的非交互式环境极度精简，缺少环境变量和缓存目录权限**。

当通过 Webhook 触发部署时，执行脚本的用户通常没有 `$HOME` 环境变量，也没有向默认缓存目录（如 `~/.cache/hugo_cache`）写入临时文件的权限。对于 Hextra 这种重度依赖 Tailwind CSS 实时编译的主题，Hugo 在没有缓存写入权限时，会默默跳过 CSS 生成步骤，只输出 HTML。

**终极解法**：在部署脚本中强制注入环境变量，并使用 `--cacheDir` 指定一个 Webhook 绝对有权限读写的本地缓存目录。

最终优化后的 `deploy.sh` 核心片段如下：

```bash
#!/bin/bash
# 强制为 Webhook 注入环境变量
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export HOME=/home/ubuntu 

# ... (拉取代码等步骤) ...

# 强制指定当前项目下的缓存目录，确保 Webhook 有权限读写，并开启 minify
/usr/local/bin/hugo --gc --minify --cacheDir /home/ubuntu/hugo-site/hugo_cache -d public

# 平滑同步到 Nginx 目录
rsync -av --delete public/ /var/www/blog/
chmod -R 755 /var/www/blog
```

通过这次调整，本地撰写 $\rightarrow$ GitHub 托管 $\rightarrow$ Webhook 自动无缝编译部署的闭环终于彻底打通，~~累死我了~~
