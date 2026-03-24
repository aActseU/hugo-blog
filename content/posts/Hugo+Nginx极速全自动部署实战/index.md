--- 
title: 避坑指南：AWS 小机 & 纯 IP 的 Hugo + Nginx 极速全自动部署实战
date: 2026-03-24T17:30:00+09:00 
draft: false
---

在 2026 年的今天，拥有一个属于自己的博客依然是记录思考、分享技术的最酷方式。对于我们这些手握 AWS EC2 t2.micro 或 Lightsail 这种“小机器”的技术人来说，传统的 Jenkins 或 GitLab CI 部署方案简直就是“内存杀手”。

最近，我通过一次精彩的探索，成功搭建了一套完美的、基于 **Hugo + Nginx + Webhook** 的极速全自动部署系统。它不依赖复杂的 CI/CD 工具，不占用不必要的内存，完美适应无域名环境。更重要的是，在搭建过程中，我把所有可能遇到的坑（权限、路径、版本代沟）都踩了一遍。

本教程就是基于这次实战的探索，为你整理的“避坑版”指南。

## 0. 为什么选择这套架构？

* **轻量化**：对于小机器，内存就是生命。Nginx 伺服静态文件效率无敌，Hugo 编译速度极快。
* **全自动**：`git push` 到 GitHub 后，剩下的活儿（拉取、编译、发布、权限修复）全由服务器自己包揽。
* **零成本（软件）**：全部使用开源免费软件。

---

## 1. 准备工作

### AWS 安全组配置
在开始安装之前，必须去 AWS 控制台修改这台实例的**安全组（Security Group）**，放行以下端口的入站规则：

* **TCP 80**：HTTP 访问（必选）
* **TCP 9000**：允许 GitHub 的 Webhook 信号接入（必选）

### 域名解析（可选）
如果你有域名，可以添加 A 记录指向你的 IP。如果没有域名，**本教程完全支持使用纯 IP 访问**。

---

## 2. 软件安装：一定要避开“版本坑”

登录 AWS 服务器，开始安装核心组件。

```bash
sudo apt update
sudo apt install nginx git webhook -y
```

### 避坑第一点：手动安装最新版 Hugo

**⚠️ 警告：** Ubuntu 官方自带软件库（`apt`）里的 Hugo 版本通常太老（例如 `v0.123`），不兼容许多现代化新主题（通常要求 `v0.146.0`+）。直接执行 `apt install hugo` 大概率会导致编译失败。

我们要去 Hugo GitHub Releases 下载最新版本进行手动安装：

1.  **下载最新版安装包**（这里推荐大家使用的 `v0.146.0` 扩展版）：
    ```bash
    wget https://github.com/gohugoio/hugo/releases/download/v0.146.0/hugo_extended_0.146.0_linux-amd64.deb
    ```
2.  **强制安装**：
    ```bash
    sudo dpkg -i hugo_extended_0.146.0_linux-amd64.deb
    ```
3.  **验证版本**：
    ```bash
    hugo version
    # 确保输出显示 v0.146.0
    ```

---

## 3. 极简 Nginx 配置（纯 IP 版）

对于 Hugo 静态博客，我们无需复杂的 Nginx 配置。最重要的是把静态文件的输出目录设为 Nginx 的标准托管目录，比如 `/var/www/blog`。

**1. 创建网站目录并处理好权限**：
```bash
sudo mkdir -p /var/www/blog
sudo chown -R ubuntu:www-data /var/www/blog
```

**2. 创建配置文件**：
```bash
# 注意拼写，是 sites-available
sudo nano /etc/nginx/sites-available/blog
```
写入以下内容（把域名或 IP 替换成你自己的）：
```nginx
server {
    listen 80;
    # 填入你的公网 IP
    server_name 13.212.22.21; 

    # 指定 Hugo 生成的 public 文件所在目录
    root /var/www/blog;
    index index.html index.xml;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

**3. 启用配置并重启 Nginx**：
```bash
# 创建软链接启用站点
sudo ln -s /etc/nginx/sites-available/blog /etc/nginx/sites-enabled/
# 检查语法
sudo nginx -t
# 重载应用
sudo systemctl reload nginx
```

---

## 4. 自动化流水线的“心脏”：Webhook 与脚本

这一步我们将把 GitHub 推送信号转化为服务器的编译动作。为了避免用户权限错误（403），我们优化了目录结构：**源码放在用户的 home 目录，静态文件编译输出到 Nginx 目录下。**

### 第一步：编写增强版部署脚本 (`deploy.sh`)

这个脚本不仅处理 Git 拉取和 Hugo 编译，**最重要的是，它还会自动处理主题子模块（Submodule）和修复 Nginx 读取文件的权限。**

**1. 在 home 目录新建脚本**：
```bash
nano /home/ubuntu/deploy.sh
```
写入以下内容（注意我们特意把 Hugo 的路径指向了手动安装后的地方）：
```bash
#!/bin/bash
echo ">>> Pulling latest GitHub changes..."
git -C /home/ubuntu/hugo-site reset --hard
git -C /home/ubuntu/hugo-site pull origin main

# 强制拉取并更新主题子模块代码
git -C /home/ubuntu/hugo-site submodule update --init --recursive

echo ">>> Building Hugo site..."
cd /home/ubuntu/hugo-site
# 清空旧的构建文件
rm -rf /var/www/blog/*
# 执行构建，并使用 -d 指定 Nginx 的托管目录
/usr/local/bin/hugo -d /var/www/blog

echo ">>> Fixing permissions for Nginx..."
# 赋予 Nginx 读取文件的权限
chmod -R 755 /var/www/blog

echo ">>> Deploy completed!"
```
给脚本执行权限：
```bash
chmod +x /home/ubuntu/deploy.sh
```

### 第二步：编写 Webhook 安全配置文件 (`hooks.json`)

为了防止互联网上的恶意扫描，我们需要加上 Secret 鉴权验证。

1.  **生成一个高强度的、随机的密钥**：
    ```bash
    openssl rand -hex 20
    # 把它复制下来，两边都要用到
    ```
2.  **编辑 hooks.json**：
    ```bash
    nano /home/ubuntu/hooks.json
    ```
    填入以下安全配置，把 `你的超强秘钥_openssl生成的` 替换成上面的随机字符串：
    ```json
    [
      {
        "id": "deploy-blog",
        "execute-command": "/home/ubuntu/deploy.sh",
        "command-working-directory": "/home/ubuntu",
        "response-message": "Deploy triggered",
        "trigger-rule": {
          "match": {
            "type": "payload-hash-sha256",
            "secret": "你的超强秘钥_openssl生成的",
            "parameter": {
              "source": "header",
              "name": "X-Hub-Signature-256"
            }
          }
        }
      }
    ]
    ```

### 第三步：配置 Webhook 为系统服务

```bash
sudo nano /etc/systemd/system/webhook.service
```
填入以下内容：
```ini
[Unit]
Description=Webhook Listener for Hugo Auto Deployment
After=network.target

[Service]
ExecStart=/usr/bin/webhook -hooks /home/ubuntu/hooks.json -port 9000 -verbose
Restart=always
User=ubuntu
WorkingDirectory=/home/ubuntu

[Install]
WantedBy=multi-user.target
```
启动服务：
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now webhook
```

---

## 5. GitHub 端的最后设置

1.  进入博客仓库 -> **Settings** -> **Webhooks** -> **Add webhook**
2.  **Payload URL**: `http://你的公网IP:9000/hooks/deploy-blog`
3.  **Content type**: 选择 `application/json`
4.  **Secret**: 填入你刚才在 `hooks.json` 中设置的那串随机密钥。
5.  **SSL verification**: 选择红色标明的 **Disable (not recommended)**（因为我们没有 HTTPS，不禁用会导致 GitHub 报错）。
6.  **Which events...**: 保持默认的 **Just the push event.** 即可。

---

## 6. 避坑全集 & 故障排查

如果点击 GitHub 上的 Re-deliver 后网页依然显示 **403 Forbidden**，请按照以下顺序在服务器端进行排查：

### 方法 0：查看 Webhook 的实时日志
```bash
sudo journalctl -u webhook -f
```
盯着屏幕，再次发送 Ping 请求，看看错误出在哪里。

### 坑一：Hugo 构建失败 (No such file)
* **现象**：`journalctl` 报错 `deploy.sh: line 14: /usr/bin/hugo: No such file or directory`。
* **原因**：脚本里写错了 Hugo 的路径。
* **解决**：运行 `which hugo` 确认实际路径（通常是 `/usr/local/bin/hugo`），并修改脚本。

### 坑二：Hugo 主题丢失 (render of "home" failed)
* **现象**：`deploy.sh` 脚本虽然执行完，但报错主题路径找不到或者 layout 文件找不到。
* **原因**：没有拉取 Git 主题子模块。
* **解决**：确保你的 `deploy.sh` 包含了 `git submodule update` 这一步。

### 坑三：Nginx 权限问题 (403)
* **现象**：编译成功，文件夹有文件，但网页依然 403。
* **原因**：Hugo 生成的文件 Nginx 读取不到。
* **解决**：确保你的 `deploy.sh` 包含了 `chmod -R 755 /var/www/blog` 这一步。

### 坑四：Obsidian 图片全部丢失
* **现象**：网页能打开，但从 Obsidian 复制过来的图片全是空白框。
* **原因**：Obsidian 的双链语法不兼容 Hugo，且 Hugo 找不到图片文件。
* **解决**：推荐长期方案，在 Hugo 中使用“页面捆绑包” (Page Bundles)。在 `content/posts/` 下新建一个专门放文章的文件夹，把图片和文章（重命名为 `index.md`）放在一起。然后在 Markdown 里直接引用图片名，比如 `![自建节点截图](proxy.png)`。