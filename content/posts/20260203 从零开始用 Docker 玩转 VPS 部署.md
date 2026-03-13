---
title: 从零开始用 Docker 玩转 VPS 部署
date: 2026-02-03T18:38:53+08:00
lastmod: 2026-02-03T18:38:41+08:00
keywords:
  - VPS
  - DOCKER
  - NPM
description: docker容器、NPM反向代理
tags:
  - VPS
  - DOCKER
categories:
  - 教程
draft: false
---


# 从零开始用 Docker 玩转 VPS 部署

如果你刚买了一台 VPS（虚拟专用服务器），正对着空白的命令行发呆，不知道该装点什么，那么这篇文章就是为你准备的。

今天，我们要聊聊目前最流行的服务部署方式：**Docker + Nginx Proxy Manager (NPM)**。

## 一、 为什么我们要用 Docker？

你可以把 VPS 想象成一间**毛坯房**，而你要安装的各种服务（比如云盘、博客、相册）就像是家具。

1. **不弄脏地板（更稳定）**：传统的安装方式就像直接往地板上钉钉子，装多了房子就乱了。Docker 则是把每个服务放进一个透明的“集装箱”，互不干扰，不想用了直接搬走，不留痕迹。
    
2. **只开一扇大门（更安全）**：通常每个服务都要开一个端口，就像给房子开了无数扇窗户。用了 Docker，我们只留一扇“正门”（80/443 端口），安全性大大提升。
    
3. **一键搬家（更便捷）**：Docker 把所有配置都写在一张“配方表” (`docker-compose.yml`) 里。下次换服务器，只要带着这张表和数据文件夹，几秒钟就能在新宿舍复原。
    

## 二、 基础：给你的 VPS 装上“引擎”

在开始之前，我们需要在 VPS（推荐使用 Ubuntu/Debian 系统）上安装 Docker 及其管理工具。

### 1. 一键安装环境

复制并粘贴以下命令，剩下的交给系统：

```
# 1. 更新系统并安装基础工具
sudo apt update && sudo apt install -y ca-certificates curl gnupg lsb-release

# 2. 信任并添加 Docker 官方软件源
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. 安装 Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

> **小贴士**：看到 `docker --version` 弹出版本号，说明你的“引擎”已经启动成功了！

## 三、 进阶：请一位“大内总管” (NPM)

在 VPS 上跑多个服务，最头疼的就是记不住端口号，还有那折磨人的 SSL 证书申请。

**Nginx Proxy Manager (NPM)** 就像是一位优雅的管家：

- **全图形化操作**：不需要写复杂的代码，点点鼠标就能配置域名。
    
- **SSL 自动续期**：免费的 HTTPS 证书，一键申请，再也不用担心浏览器报错。
    

### 部署 NPM

首先，我们建立一个“公共社区网络”，让所有容器都能互相说话：

```
docker network create net
```

然后在 `~/docker/npm` 目录下创建 `docker-compose.yml`，填入以下内容：

```
services:
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: npm
    restart: unless-stopped
    ports:
      - '80:80'    # HTTP 访问
      - '443:443'  # HTTPS 访问
      - '81:81'    # 管理面板（首次登录后建议防火墙关闭此端口或仅限自己访问）
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    networks:
      - net

networks:
  net:
    external: true
```

## 四、 实战：部署私有云盘 (Nextcloud)

有了管家，我们来请进第一位客人——**Nextcloud**。

### 1. 编写“配方表”

在 `~/docker/nextcloud` 文件夹下创建 `docker-compose.yml`：


```
services:
  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    networks:
      - net
    # 🔒 注意：我们不需要在这里写 ports，因为它躲在 NPM 后面，非常安全
    volumes:
      - ./data:/var/www/html
networks:
  net:
    external: true
```

运行 `docker compose up -d`，它就在后台悄悄跑起来了。

### 2. 让管家接客

现在，打开 NPM 的后台（`你的IP:81`）：

1. 点击 **Proxy Hosts** -> **Add Proxy Host**。
2. **Domain Names**: 输入你解析好的域名（如 `cloud.yourdomain.com`）。
3. **Forward Hostname**: 直接填 `nextcloud`（这就是 Docker 内部网络的魔力，不用记 IP）。
4. **Forward Port**: 填 `80`。
5. **SSL 选项卡**: 选择 `Request a new SSL Certificate`，勾选 `Force SSL`。

---

## 五、 结语

通过这种方式，你搭建的服务就像是一个整齐的**模块化工厂**：

- **新加服务**：再写一个 `docker-compose`，在 NPM 里点几下，几分钟搞定。
- **备份迁移**：直接拷贝整个 `docker` 文件夹，到哪都能原地复活。
- **私密性**：你的数据只属于你，没有广告，没有限速。

这样的服务有**相册备份、个人笔记、FreeRss订阅**等，它们都可以通过同样的方式完美运行在VPS上。

当你部署成功并用上之后，有没有感觉自己搭建的服务用起来成就感满满？
