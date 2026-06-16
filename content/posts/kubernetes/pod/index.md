---
title: Pod
date: 2020-10-02T17:43:02+08:00
lastmod: 2020-10-02T17:43:02+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: image.png
images:
  - image.png
categories:
  - Kubernetes
tags:
  - Pod
# nolastmod: true
draft: fale
description: "在 Kubernetes（K8s）里，Pod 属于最基础且最小的可部署单元。它代表一组紧密关联的容器集合，这些容器会共享存储、网络以及运行环境。可以把 Pod 当作是运行特定应用程序的一个“逻辑主机”。"
---

# Pod 解析：

## 一、Pod 基础概念

### 1. 定义

在 Kubernetes（K8s）里，Pod 属于最基础且最小的可部署单元。它代表一组紧密关联的容器集合，这些容器会共享存储、网络以及运行环境。可以把 Pod 当作是运行特定应用程序的一个“逻辑主机”。

### 2. 设计理念

Pod 的设计是为了支持那些紧密耦合、相互协作的进程，这些进程可能会一起部署和管理。比如，一个主应用容器和一个用于日志收集的 Sidecar 容器就可以放在同一个 Pod 中。

### 3. 共享资源

- **网络**：Pod 内的所有容器共享同一个网络命名空间，拥有相同的 IP 地址和端口空间。这意味着容器之间可以通过 `localhost` 直接通信，极大地简化了应用内部的网络交互。
- **存储**：Pod 能够定义一个或多个存储卷（Volume），这些存储卷可被 Pod 内的所有容器访问，方便实现数据的持久化或者容器间的数据共享。

## 二、Pod 的生命周期

### 1. Pending（挂起）

- **状态说明**：当你创建一个 Pod 时，它首先会进入 Pending 状态。此时，Kubernetes API Server 已经接收到创建请求，但 Pod 还未被完全创建好。
- **可能原因**：
  - 调度器还没为 Pod 分配合适的节点。
  - 容器镜像正在下载，特别是大镜像时，下载时间可能较长。

### 2. ContainerCreating（容器创建中）

- **状态说明**：一旦 Pod 被调度到某个节点，kubelet 就会开始创建容器。
- **主要操作**：
  - 从镜像仓库下载容器镜像（如果本地不存在）。
  - 创建容器的网络和存储等资源。
  - 启动容器进程。

### 3. Running（运行中）

- **状态说明**：当 Pod 内的所有容器都成功创建并启动后，Pod 进入 Running 状态。不过，这并不意味着应用程序一定能正常工作，还需通过健康检查来确认。
- **相关操作**：此时，Pod 中的应用程序开始提供服务，并且可以接收外部流量。

### 4. Succeeded（成功）

- **状态说明**：如果 Pod 中的所有容器都正常终止（退出码为 0），Pod 就会进入 Succeeded 状态。
- **常见场景**：这种情况常见于一次性任务，比如批处理作业，任务完成后容器就会正常退出。

### 5. Failed（失败）

- **状态说明**：若 Pod 中的任何一个容器以非零退出码终止，或者在创建、运行过程中出现错误，Pod 就会进入 Failed 状态。
- **可能原因**：
  - 镜像下载失败。
  - 容器启动脚本出错。
  - 应用程序内部错误。

### 6. Unknown（未知）

- **状态说明**：当 kubelet 无法向 API Server 汇报 Pod 的状态时，Pod 会处于 Unknown 状态。
- **可能原因**：通常是由于节点与 API Server 之间的网络问题，或者 kubelet 进程崩溃导致。

### 7. 生命周期钩子（Lifecycle Hooks）

Kubernetes 允许为 Pod 中的容器定义生命周期钩子，这些钩子会在容器的特定阶段执行。

- **postStart**：容器启动后立即执行的操作，可用于初始化工作，例如创建必要的文件或目录。
- **preStop**：容器终止前执行的操作，可用于优雅关闭应用程序，比如保存数据、释放资源等。

以下是包含生命周期钩子的 Pod 示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-hook-pod
spec:
  containers:
    - name: my-container
      image: nginx:1.14.2
      lifecycle:
        postStart:
          exec:
            command:
              ["/bin/sh", "-c", "echo 'Container started' > /tmp/start.log"]
        preStop:
          exec:
            command: ["/usr/sbin/nginx", "-s", "quit"]
```

### 8. 健康检查（Probes）

为了确保 Pod 中的应用程序正常运行，Kubernetes 提供了健康检查机制，主要有两种类型的探针：

- **存活探针（Liveness Probe）**：用于检测容器是否存活。如果存活探针失败，Kubernetes 会重启容器。
- **就绪探针（Readiness Probe）**：用于检测容器是否准备好接收流量。如果就绪探针失败，Kubernetes 会将该 Pod 从服务的负载均衡中移除。

以下是包含健康检查的 Pod 示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-pod
spec:
  containers:
    - name: my-container
      image: nginx:1.14.2
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 15
        periodSeconds: 5
      readinessProbe:
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
```

## 三、Pod 的调度

### 1. 调度器的作用

Kubernetes 中的调度器（Scheduler）负责将 Pod 分配到合适的节点上运行。调度器会综合考虑节点的资源状况、Pod 的资源请求以及各种调度策略来做出决策。

### 2. 调度过程

#### 预选阶段（Predicate）

调度器会根据一系列规则过滤掉不满足条件的节点，主要规则如下：

- **资源检查**：检查节点的可用 CPU、内存等资源是否满足 Pod 的请求。如果节点资源不足，该节点会被排除。
- **节点选择器（NodeSelector）**：如果 Pod 定义了 `nodeSelector`，调度器会只考虑带有相应标签的节点。
- **节点亲和性与反亲和性（Node Affinity and Anti - Affinity）**：可以定义 Pod 与节点之间的亲和或反亲和关系。例如，要求 Pod 必须调度到特定区域的节点上，或者避免调度到某些节点上。
- **污点与容忍度（Taints and Tolerations）**：节点可以设置污点，防止某些 Pod 被调度到该节点上；而 Pod 可以设置容忍度，表示它可以容忍哪些污点。

#### 优选阶段（Priority）

在通过预选的节点中，调度器会对每个节点进行打分，选择得分最高的节点。打分的依据有很多，例如：

- **节点负载情况**：优先选择负载较低的节点，以保证 Pod 能稳定运行。
- **与其他 Pod 的亲和性**：为了提高性能或实现高可用，可能会优先选择与其他相关 Pod 在同一节点或不同节点的节点。

### 3. 调度策略示例

#### 节点选择器

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-selector-pod
spec:
  containers:
    - name: my-container
      image: nginx:1.14.2
  nodeSelector:
    disktype: ssd
```

上述示例中，Pod 会被调度到带有 `disktype: ssd` 标签的节点上。

#### 节点亲和性

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-pod
spec:
  containers:
    - name: my-container
      image: nginx:1.14.2
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: region
                operator: In
                values:
                  - us-west
```

此示例中，Pod 必须被调度到带有 `region: us-west` 标签的节点上。

#### 污点与容忍度

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: taint-toleration-pod
spec:
  containers:
    - name: my-container
      image: nginx:1.14.2
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "special-user"
      effect: "NoSchedule"
```

该 Pod 可以容忍带有 `dedicated: special-user` 污点且效果为 `NoSchedule` 的节点。

### 4. 调度失败处理

如果调度器无法为 Pod 找到合适的节点，Pod 会一直处于 Pending 状态。可以通过查看 `kubectl describe pod <pod-name>` 的输出信息来定位调度失败的原因，常见原因包括节点资源不足、节点标签不匹配等。

## 四、总结

Pod 作为 Kubernetes 中最核心的概念之一，其生命周期和调度机制对于有效管理和运行容器化应用程序起着至关重要的作用。通过合理配置 Pod 的生命周期钩子、健康检查和调度策略，能够显著提高应用程序的可用性和性能。同时，深入理解 Pod 的相关知识，有助于更好地应对 Kubernetes 集群中的各种场景和问题。
