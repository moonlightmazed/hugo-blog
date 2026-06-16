---
title: Harbor
date: 2024-01-13T18:48:49+08:00
lastmod: 2024-01-13T18:48:49+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: image.png
images:
  - image.png
categories:
  - CICD
tags:
  - Harbor
# nolastmod: true
description: "Harbor 是由 VMware 开源的一款云原生制品仓库，Harbor 的核心功能是存储和管理 Artifact。"
draft: false
---

## 更换国内源

```bash
# 阿里
sed -e 's|^#mirrorlist=|mirrorlist=|g' \
    -e 's|^baseurl=https://mirrors.aliyun.com/rockylinux|#baseurl=http://dl.rockylinux.org/$contentdir|g' \
    -i.bak \
    /etc/yum.repos.d/rocky*.repo

# 中科大
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.ustc.edu.cn/rocky|g' \
    -i.bak \
    /etc/yum.repos.d/rocky-extras.repo \
    /etc/yum.repos.d/rocky.repo

# 生成缓存
dnf makecache
```

## Docker 安装

### 更新系统与安装依赖

```bash
sudo dnf update -y
sudo dnf install -y yum-utils device-mapper-persistent-data lvm2
```

### 配置国内 Docker 源

```bash
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo sed -i 's/download.docker.com/mirrors.aliyun.com\/docker-ce/' /etc/yum.repos.d/docker-ce.repo
```

### 安装 Docker

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io
```

### 安装完成后启动服务并设置开机自启

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

### 配置国内镜像加速器

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
   "https://192e82cca03a4a8aa64fea319af4dd55.mirror.swr.myhuaweicloud.com",
    "https://yp8sluaa.mirror.aliyuncs.com",
    "https://registry.docker-cn.com",
    "https://mirror.ccs.tencentyun.com",
    "https://registry.hub.docker.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://registry.docker-cn.com"
  ]
}
EOF
```

## 证书

### 创建证书目录

```bash
sudo mkdir -p /etc/harbor/ssl
cd /etc/harbor/ssl
```

### 生成自签名证书

```bash
# 生成私钥
sudo openssl genrsa -out harbor.local.com.key 2048

# 生成证书签名请求（注意替换IP/域名）
sudo openssl req -new -key harbor.local.com.key -out harbor.local.com.csr -subj "/CN=harbor.local.com"

# 生成自签名证书（有效期10年）
openssl req -x509 -nodes -days 3650 \
  -newkey rsa:2048 \
  -keyout harbor.local.com.key \
  -out harbor.local.com.crt \
  -subj "/CN=harbor.local.com" \
  -addext "subjectAltName=IP:192.168.66.66,DNS:harbor.local.com"

#设置权限
sudo chmod 600 harbor.local.com.key
```

### 将证书加入系统

```bash
# 将Harbor的证书复制到系统CA信任目录
sudo mkdir -p /etc/pki/ca-trust/source/anchors
sudo cp /etc/harbor/ssl/harbor.local.com.crt /etc/pki/ca-trust/source/anchors/

# 更新系统证书信任库
sudo update-ca-trust extract
```

### 验证证书信任状态

```bash
# 检查证书是否已加入信任库
sudo trust list | grep harbor.local.com
# 应显示类似：harbor.local.com: CN=harbor.local.com
```

### docker 证书信任

```bash
# 创建Docker证书目录
sudo mkdir -p /etc/docker/certs.d/harbor.local.com

# 复制证书到Docker目录
sudo cp /etc/harbor/ssl/harbor.local.com.crt /etc/docker/certs.d/harbor.local.com/ca.crt

# 重启Docker服务
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 使用 openssl 验证证书链

```bash
openssl s_client -connect harbor.local.com:443 -showcerts
# 检查输出中是否包含Verify return code: 0 (ok)
```

### 使用 Let's Encrypt 免费证书（生产环境推荐）

```bash
# 安装Certbot
sudo yum install -y certbot

# 生成证书
sudo certbot certonly --standalone -d harbor.local.com
```

## 部署 harbor

### 下载镜像

```bash
wget https://github.com/amy5200/harbor-arm64/releases/download/v2.11.0/harbor-v2.11.0-aarch64.tar.gz
sudo tar -zxvf harbor-v2.11.0-aarch64.tar.gz -C /usr/local
cd /usr/local/harbor
```

### 配置文件

```yaml
hostname: harbor.local.com # 域名
http:
  port: 80 # 非必要可注释
https:
  port: 443
  certificate: /etc/harbor/ssl/harbor.local.com.crt
  private_key: /etc/harbor/ssl/harbor.local.com.key
harbor_admin_password: Harbor12345 # 管理员密码
data_volume: /data/harbor # 存储路径
```

### 启动

```bash

# 预处理配置
sudo ./prepare

# 启动安装
sudo ./install.sh
```

### 登陆

```bash
# 验证
curl https://harbor.local.com

# Docker客户端登陆
docker login harbor.local.com -u admin
```
