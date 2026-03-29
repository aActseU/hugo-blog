--- 
title: 为个人博客配置域名与 HTTPS 证书
description: "给 Nginx 配置域名，利用 certbot 申请证书"
date: 2026-03-29T16:30:00+09:00 
draft: false
tags: ["域名", "证书", "Certbot"]
---

之前在 AWS 上用 Hugo 和 Nginx 搭建了个人博客，但一直只能用 IP 地址访问，既不美观也不安全。这周末决定给博客申请一个专属域名，并配上 HTTPS 小锁头。

本来以为只是敲几行命令的事，没想到遇到了一个极其隐蔽的“端口劫持”大坑。在这里记录下完整的配置与排查过程，希望能帮到大家。

## 第一步：申请免费域名并绑定 IP

我使用的是 `nic.zle.ee` 提供的免费子域名服务。申请过程非常简单：
1. 在控制面板输入想要的子域名（例如 `newtonking.zle.ee`）。
2. 添加一条 **A 记录**，将“值”填入我 AWS 服务器的公网 IPv4 地址。

**注意一个限制：** 这个免费域名无法修改 NS (Name Server) 记录，这意味着不能把它托管给 Cloudflare。我们无法享受 CF 的免费 CDN 和一键 HTTPS，必须老老实实在自己的服务器上手动申请和配置 SSL 证书。

## 第二步：Nginx 基础配置

在申请证书前，需要先让 Nginx 认得这个新域名。

1. 打开 Nginx 的站点配置文件（例如 `/etc/nginx/sites-available/blog`），将 `server_name` 修改为刚申请的完整域名。
2. **清理干扰项：** 删除 Nginx 自带的默认配置软链接，防止它抢走 80 端口的流量：
   `sudo rm -f /etc/nginx/sites-enabled/default`
3. 养成好习惯，先测试再重载：
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   ```

## 第三步：申请 Let's Encrypt 证书

采用自动化神器 Certbot：
```bash
sudo apt install certbot python3-certbot-nginx
```

原本打算用 `sudo certbot --nginx -d 你的域名` 一键搞定，结果 Certbot 一直报错 `404`，提示无法验证域名所有权。与此同时，哪怕我已经在 AWS 的安全组（Security Group）里放行了 443 端口，浏览器访问域名依然是无限转圈，死活打不开。

### 核心排查：是谁动了我的 443 端口？

外部防火墙放行了，IP 也没错，内部 UFW 防火墙也是关闭状态，那问题一定出在服务器内部的网络路由上。

为了排除外部干扰，我直接在服务器终端“自己访问自己”进行测试：
```bash
curl --noproxy "*" -k -I https://127.0.0.1 -H "Host: 你的域名"
```

诡异的事情发生了：Nginx 并没有返回正常的网页，终端里竟然打印出了一个带有 `server: cloudflare` 和某海外节点标记的 `HTTP/2 403` 拦截页面！

**本地 Nginx 怎么可能伪装成 Cloudflare？** 这说明我的请求在极其底层的网络层面被劫持了。

马上用大招查一下到底是谁在监听 443 端口：
```bash
sudo ss -tulnp | grep 443
```

**真相大白：** 占用 443 端口的根本不是 Nginx，而是我之前在服务器上部署的一个**代理服务端程序**！
因为这个代理工具抢先霸占了 443 端口，导致：
1. Let's Encrypt 的验证流量被代理工具截获，验证文件自然找不到（报 404）。
2. 正常访客的 HTTPS 流量全进到了代理隧道里，Nginx 连一滴流量都接不到（浏览器无限转圈）。

### 解决方案：移交端口控制权

既然找出了幕后黑手，解决起来就很明确了——把 443 端口彻底还给 Nginx：

1. 打开代理程序的配置文件，将其监听端口（listen_port）从 `443` 修改为其他不常用的端口（例如 `8443`）。
2. 重启代理程序和 Nginx，完成端口的重新交接：
   ```bash
   sudo systemctl restart 代理程序
   sudo systemctl restart nginx
   ```
3. 前往 AWS 控制台，在安全组中放行新的 `8443` 端口，以保证原有的代理服务依然可用。

## 第四步：终极方案 Webroot 模式拿证书

排除了端口冲突后，为了避免 Certbot 的 Nginx 插件再次和现有的路由规则打架，我改用了业界公认最稳妥的 **Webroot 模式**。

这种模式的原理极其简单粗暴：不修改任何 Nginx 配置，直接把验证文件塞进博客的静态目录里让机构去读取。

先进行演习：
```bash
sudo certbot certonly --webroot -w /var/www/blog -d 你的域名 --dry-run
```
看到漂亮的 `The dry run was successful` 后，去掉 `--dry-run` 正式申请。

