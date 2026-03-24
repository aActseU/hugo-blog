--- 
title: 🚀 AWS + Hugo 自动化部署探索笔记：从本地到云端
date: 2026-03-24T17:30:00+09:00 
draft: false
---

## 📋 核心逻辑简述
要实现“本地写稿 $\rightarrow$ GitHub 托管 $\rightarrow$ AWS 自动编译部署”，关键在于**源码**。Hugo 本质上是一个把 Markdown 转换成 HTML 的翻译官。我们现在的任务是在本地搭建好这个“翻译工厂”，然后把图纸扔到 GitHub 上。

---

## 第一阶段：本地环境搭建（避坑指南）

### 1. Hugo 软件安装与环境变量
1. 以`Windows`为例，在[https://github.com/gohugoio/hugo/releases]下载适用于本系统的压缩包`hugo_extended_0.xxx.x_windows-amd64.zip`
2. 把它放到一个文件夹（如 `D:\hugo\bin`），并把这个路径添加到系统的 **环境变量 (Path)** 中。
	>  添加**目录**到环境变量中（即`D:\hugo\bin`）而不是文件（`D:\hugo\bin\hugo.exe`）
	
3. 打开终端（CMD 或 PowerShell），输入 `hugo version`。看到版本号，说明你成功了

### 2. 初始化站点
```bash
hugo new site myblog
cd myblog
git init
```

---

## 第二阶段：解决“网络与连接”大山

### 1. Git 代理配置
在拉取主题（Submodule）时，如果遇到 `SSL_ERROR_SYSCALL` 或连接超时：
* **解决方案**：利用本地的 Clash 代理（假设端口为 7890）。
    ```bash
    git config --global http.proxy http://127.0.0.1:7890
    git config --global https.proxy https://127.0.0.1:7890
    ```
* **经验**：Git 不会自动识别系统代理，必须手动“指路”。

### 2. 添加主题（子模块方式）
Hugo 默认是没样式的，我们必须选一个主题。以极简的 `Ananke` 为例：

```bash
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
```

---

## 第三阶段：致命的“编码陷阱” ⚠️

这是新手最容易卡住的地方。

### 1. 最好不要用 PowerShell `echo` 追加
* **问题**：使用 `echo "theme = 'ananke'" >> hugo.toml` 会导致文件变成 **UTF-16** 编码。
* **报错表现**：`unmarshal failed: toml: invalid character at start of key: ÿ`（那个 `ÿ` 就是编码乱码的标志）。

### 2. 记事本的正确打开方式
* **手动修正**：使用记事本打开 `hugo.toml`，“另存为”时编码必须选择 **UTF-8**（不要带 BOM）。
* **终极建议**：推荐使用 **VS Code**，它能自动处理编码并高亮 TOML 语法错误。

---

## 第四阶段：源码上云（GitHub 推送）

在 GitHub 创建一个空的 Repository（例如 `my-blog`），然后在本地执行：

```bash
git add .
git commit -m "feat: 初建博客并添加主题"
git branch -M main
git remote add origin https://github.com/你的用户名/my-blog.git
git push -u origin main
```

---

## 🏁 总结与后续
至此，你已经解决了“GitHub 仓库没有源码”的根本问题。你的“货”已经拉到了 GitHub 仓库，接下来 AWS 上的 Webhook 才有东西可以搬运。

