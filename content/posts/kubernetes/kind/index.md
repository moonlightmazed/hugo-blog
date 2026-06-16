---
title: Kind
date: 2020-10-03T14:11:02+08:00
lastmod: 2020-10-03T14:11:02+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: image.png
images:
  - image.png
categories:
  - Kubernetes
tags:
  - Kind
# nolastmod: true
draft: false
description: "kind是k8s官网提供给学习者用来创建k8s的一种方式，适用于学习环境下，资源不充足的情况。"
---

## Kind 的 yaml 配置文件

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = ["https://ustc-edu-cn.mirror.aliyuncs.com", "https://ccr.ccs.tencentyun.com", "https://docker.m.daocloud.io"]
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
        protocol: TCP
      - containerPort: 30081
        hostPort: 30081
        protocol: TCP
      - containerPort: 30082
        hostPort: 30082
        protocol: TCP
      - containerPort: 30083
        hostPort: 30083
        protocol: TCP
      - containerPort: 30084
        hostPort: 30084
        protocol: TCP
      - containerPort: 30085
        hostPort: 30085
        protocol: TCP
      - containerPort: 30086
        hostPort: 30086
        protocol: TCP
      - containerPort: 30087
        hostPort: 30087
        protocol: TCP
      - containerPort: 30088
        hostPort: 30088
        protocol: TCP
      - containerPort: 30089
        hostPort: 30089
        protocol: TCP
      - containerPort: 30090
        hostPort: 30090
        protocol: TCP
      - containerPort: 31272
        hostPort: 31272
        protocol: TCP
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

## 安装 nginx 测试集群

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.22
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080 # 可以指定一个端口号，如果不指定，会自动分配一个
```
