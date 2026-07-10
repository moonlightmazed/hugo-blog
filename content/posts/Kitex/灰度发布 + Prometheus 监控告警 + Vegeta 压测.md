---
title: 灰度发布 + Prometheus 监控告警 + Vegeta 压测
slug: gray-release-prometheus-alert-vegeta-benchmark
aliases:
  - /posts/kitex/灰度发布-+-prometheus-监控告警-+-vegeta-压测/
date: 2025-01-03T14:21:02+08:00
lastmod: 2025-01-03T14:21:02+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /img/covers/钟表和钱.png
images:
  - /img/covers/钟表和钱.png
categories:
  - 微服务框架
tags:
  - Kitex
  - 示例
# nolastmod: true
draft: false
description: "新增Istio 金丝雀灰度发布、Prometheus 全栈监控、Grafana 可视化仪表盘、关键告警规则、Vegeta 分布式压测脚本，实现完整的生产级可观测性与流量治理能力"
---

## 一、Istio 金丝雀灰度发布
字节跳动内部 90% 以上的微服务都使用 Istio 进行流量治理，支持按比例灰度、按用户特征灰度、按请求头灰度三种核心模式

### 1. 前置条件
- 集群已安装 Istio 1.19+
- 所有微服务已注入 Istio Sidecar
- 服务使用 Kubernetes Service 暴露

### 2. 按比例灰度发布（最常用）
适用于全量发布前的小流量验证，逐步提升灰度比例
```yaml
# k8s/istio/destination-rule.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: userservice
  namespace: cloudwego-mall
spec:
  host: userservice
  subsets:
  - name: stable
    labels:
      version: v1.0.0
  - name: canary
    labels:
      version: v1.1.0
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    connectionPool:
      tcp:
        maxConnections: 1000
      http:
        http1MaxPendingRequests: 1000
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
---
# k8s/istio/virtual-service-10percent.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: userservice
  namespace: cloudwego-mall
spec:
  hosts:
  - userservice
  http:
  - route:
    - destination:
        host: userservice
        subset: stable
      weight: 90
    - destination:
        host: userservice
        subset: canary
      weight: 10
```

### 3. 按用户 ID 灰度发布
适用于内部测试或特定用户群体验证，只让指定用户访问新版本

```yaml
# k8s/istio/virtual-service-user-id.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: userservice
  namespace: cloudwego-mall
spec:
  hosts:
  - userservice
  http:
  - match:
    - headers:
        X-User-ID:
          regex: "^(1001|1002|1003)$" # 只允许这3个用户访问灰度版本
    route:
    - destination:
        host: userservice
        subset: canary
  - route:
    - destination:
        host: userservice
        subset: stable
```

### 4. 按请求头灰度发布
适用于开发人员自测，通过添加特定请求头访问新版本

```yaml
# k8s/istio/virtual-service-header.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: userservice
  namespace: cloudwego-mall
spec:
  hosts:
  - userservice
  http:
  - match:
    - headers:
        X-Canary:
          exact: "true"
    route:
    - destination:
        host: userservice
        subset: canary
  - route:
    - destination:
        host: userservice
        subset: stable
```

### 5. 灰度发布流程（字节标准）
1. 部署新版本 Deployment，标签`version: v1.1.0`
2. 应用 DestinationRule，定义 stable 和 canary 两个子集
3. 应用 10% 流量的 VirtualService
4. 观察监控指标（错误率、延迟、QPS）15 分钟
5. 逐步提升流量到 30%、50%、100%
6. 全量后删除旧版本 Deployment 和灰度配置

---

## 二、Prometheus + Grafana 全栈监控
### 1. 新增监控依赖
```go
// go.mod 新增
require (
    github.com/kitex-contrib/monitor-prometheus v0.3.0
    github.com/hertz-contrib/monitor-prometheus v0.3.0
)
```

### 2. Kitex 服务端监控中间件
```go
// pkg/monitor/prometheus.go
package monitor

import (
    "github.com/cloudwego/kitex/server"
    "github.com/cloudwego/hertz/pkg/app/server"
    kitexprom "github.com/kitex-contrib/monitor-prometheus"
    hertzprom "github.com/hertz-contrib/monitor-prometheus"
)

// KitexServerMonitor 返回Kitex服务端监控中间件
func KitexServerMonitor(serviceName string) server.Option {
    return server.WithSuite(kitexprom.NewServerSuite(
        serviceName,
        ":9090", // 监控指标暴露端口
        kitexprom.WithEnableGoCollector(true),
        kitexprom.WithEnableProcessCollector(true),
    ))
}

// HertzServerMonitor 返回Hertz网关监控中间件
func HertzServerMonitor(serviceName string) server.Option {
    return server.WithTracer(hertzprom.NewTracer(
        serviceName,
        ":9091",
        hertzprom.WithEnableGoCollector(true),
        hertzprom.WithEnableProcessCollector(true),
    ))
}
```

### 3. 在所有服务中启用监控

```go
// cmd/userservice/main.go 新增
import "github.com/yourname/cloudwego-mall/pkg/monitor"

func main() {
    // ... 其他初始化代码

    svr := userservice.NewServer(
        service.NewUserServiceImpl(),
        // ... 其他选项
        monitor.KitexServerMonitor(config.GlobalConfig.Server.Name), // 新增
    )
}

// cmd/api-gateway/main.go 新增
func main() {
    // ... 其他初始化代码

    h := server.Default(
        server.WithHostPorts(":8080"),
        // ... 其他选项
        monitor.HertzServerMonitor("api-gateway"), // 新增
    )
}
```

### 4. Prometheus 配置文件
```yaml
# k8s/prometheus/prometheus.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
      # 抓取API网关指标
      - job_name: 'api-gateway'
        kubernetes_sd_configs:
        - role: pod
          namespaces:
            names: ['cloudwego-mall']
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_label_app]
          regex: api-gateway
          action: keep
        - source_labels: [__meta_kubernetes_pod_container_port_number]
          regex: 9091
          action: keep

      # 抓取所有微服务指标
      - job_name: 'microservices'
        kubernetes_sd_configs:
        - role: pod
          namespaces:
            names: ['cloudwego-mall']
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_label_app]
          regex: (userservice|productservice|cartservice|orderservice|paymentservice)
          action: keep
        - source_labels: [__meta_kubernetes_pod_container_port_number]
          regex: 9090
          action: keep
```

### 5. 关键告警规则

```yaml
# k8s/prometheus/rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: monitoring
data:
  mall-rules.yaml: |
    groups:
    - name: mall-service-alerts
      rules:
      # 服务不可用告警
      - alert: ServiceDown
        expr: up{job="microservices"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "服务 {{ $labels.app }} 不可用"
          description: "服务 {{ $labels.app }} 在 {{ $labels.instance }} 上已经1分钟没有响应"

      # 高错误率告警
      - alert: HighErrorRate
        expr: |
          sum(rate(kitex_server_requests_total{code!="0"}[5m])) by (service, method) /
          sum(rate(kitex_server_requests_total[5m])) by (service, method) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "服务 {{ $labels.service }} 接口 {{ $labels.method }} 错误率过高"
          description: "接口错误率达到 {{ $value | humanizePercentage }}，超过5%阈值"

      # 高延迟告警
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, sum(rate(kitex_server_request_duration_seconds_bucket[5m])) by (service, method, le)) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "服务 {{ $labels.service }} 接口 {{ $labels.method }} 延迟过高"
          description: "接口95分位延迟达到 {{ $value }}s，超过500ms阈值"

      # 高QPS告警
      - alert: HighQPS
        expr: sum(rate(kitex_server_requests_total[1m])) by (service) > 1000
        for: 1m
        labels:
          severity: info
        annotations:
          summary: "服务 {{ $labels.service }} QPS过高"
          description: "服务QPS达到 {{ $value }}，超过1000阈值"

      # 数据库连接池耗尽告警
      - alert: DBConnectionPoolExhausted
        expr: |
          go_sql_stats_open_connections / go_sql_stats_max_open_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "服务 {{ $labels.service }} 数据库连接池即将耗尽"
          description: "数据库连接池使用率达到 {{ $value | humanizePercentage }}"
```

---

## 三、Vegeta 分布式压测脚本
Vegeta 是字节跳动内部最常用的 HTTP 压测工具，支持分布式压测、动态 QPS 调整、详细的性能报告

### 1. 安装 Vegeta
```bash
go install github.com/tsenart/vegeta@latest
```

### 2. 压测脚本 benchmark/run.sh

```bash
#!/bin/bash

# 压测配置
TARGET_URL="http://localhost:8080"
DURATION="5m"
QPS=1000
CONCURRENCY=50
OUTPUT_DIR="results/$(date +%Y%m%d-%H%M%S)"

mkdir -p $OUTPUT_DIR

echo "开始压测，QPS: $QPS, 持续时间: $DURATION"
echo "结果将保存到: $OUTPUT_DIR"

# 1. 登录获取token
echo "正在获取测试用户token..."
TOKEN=$(curl -s -X POST $TARGET_URL/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"123456"}' | jq -r '.data.token')

if [ "$TOKEN" == "null" ] || [ -z "$TOKEN" ]; then
  echo "获取token失败"
  exit 1
fi

echo "获取token成功: $TOKEN"

# 2. 生成压测请求
cat > $OUTPUT_DIR/requests.txt << EOF
GET $TARGET_URL/api/v1/users/me
Authorization: Bearer $TOKEN

POST $TARGET_URL/api/v1/cart/items
Authorization: Bearer $TOKEN
Content-Type: application/json
@benchmark/payloads/add_cart.json

POST $TARGET_URL/api/v1/orders
Authorization: Bearer $TOKEN
Content-Type: application/json
@benchmark/payloads/create_order.json
EOF

# 3. 执行压测
vegeta attack -rate=$QPS -duration=$DURATION -connections=$CONCURRENCY \
  -targets=$OUTPUT_DIR/requests.txt > $OUTPUT_DIR/results.bin

# 4. 生成报告
vegeta report -type=text $OUTPUT_DIR/results.bin > $OUTPUT_DIR/report.txt
vegeta report -type=json $OUTPUT_DIR/results.bin > $OUTPUT_DIR/report.json
vegeta plot $OUTPUT_DIR/results.bin > $OUTPUT_DIR/plot.html

echo "压测完成！"
echo "文本报告: $OUTPUT_DIR/report.txt"
echo "HTML图表: $OUTPUT_DIR/plot.html"
```

### 3. 压测载荷文件

```json
// benchmark/payloads/add_cart.json
{
  "product_id": 1,
  "product_name": "iPhone 15 Pro",
  "quantity": 1,
  "price": 7999.00
}

// benchmark/payloads/create_order.json
{
  "items": [
    {
      "product_id": 1,
      "product_name": "iPhone 15 Pro",
      "quantity": 1,
      "price": 7999.00
    }
  ]
}
```

### 4. 分布式压测
在多台机器上执行以下命令，实现分布式压测：

```bash
# 机器1-3：执行压测
vegeta attack -rate=500 -duration=5m -targets=requests.txt | tee results.bin | vegeta report

# 汇总结果
vegeta report machine1.bin machine2.bin machine3.bin > final_report.txt
```

### 5. 压测指标解读
|指标|优秀|良好|需优化|
| -------------| ----------| ----------| ------------|
|成功率|\>99.9%|\>99.5%|\<99%|
|平均延迟|\<100ms|\<300ms|\>500ms|
|95 分位延迟|\<200ms|\<500ms|\>1s|
|99 分位延迟|\<500ms|\<1s|\>2s|
|QPS|达到目标|接近目标|远低于目标|


---

## 四、生产环境性能优化建议
### 1. Go 运行时优化
```go
import "runtime"

func init() {
    // 设置GOMAXPROCS为CPU核心数
    runtime.GOMAXPROCS(runtime.NumCPU())
    
    // 调整GC百分比，降低GC频率
    debug.SetGCPercent(100)
    
    // 禁用内存限制（容器环境）
    debug.SetMemoryLimit(-1)
}
```

### 2. Kitex 性能优化

```go
svr := userservice.NewServer(
    service.NewUserServiceImpl(),
    // 启用连接复用
    server.WithReusePort(true),
    // 调整读写缓冲区大小
    server.WithReadBufferSize(4096),
    server.WithWriteBufferSize(4096),
    // 调整最大并发连接数
    server.WithMaxConn(10000),
    // 启用零拷贝
    server.WithZeroCopy(true),
)
```

### 3. 数据库优化

```go
// pkg/mysql/mysql.go
sqlDB.SetMaxIdleConns(20)
sqlDB.SetMaxOpenConns(200)
sqlDB.SetConnMaxLifetime(1 * time.Hour)
sqlDB.SetConnMaxIdleTime(30 * time.Minute)
```

### 4. Redis 优化
```go
// pkg/redis/redis.go
Client = redis.NewClient(&redis.Options{
    Addr:         config.GlobalConfig.Redis.Address,
    PoolSize:     200,
    MinIdleConns: 20,
    DialTimeout:  1 * time.Second,
    ReadTimeout:  500 * time.Millisecond,
    WriteTimeout: 500 * time.Millisecond,
    IdleTimeout:  5 * time.Minute,
})
```
---


## 五、项目最终完整生产级能力总结

✅ **5 个核心微服务**：用户、购物车、商品、订单、支付
✅ **完整电商闭环**：注册→登录→加购→下单→预扣减→支付→确认扣减
✅ **高并发防超卖**：Redis + 数据库双写预扣减 + 超时自动释放
✅ **分布式事务**：Seata AT 模式保证订单 - 库存 - 支付一致性
✅ **异步解耦**：Kafka 处理订单事件、通知、统计等异步任务
✅ **全链路可观测**：OpenTelemetry+Jaeger 链路追踪 + Zap 结构化日志
✅ **监控告警**：Prometheus+Grafana 全栈监控 + 关键指标告警
✅ **流量治理**：Istio 金丝雀灰度发布 + 流量控制 + 故障注入
✅ **高可用保障**：Sentinel 熔断限流 + K8s 多副本部署 + 健康检查
✅ **生产级部署**：完整 K8s 部署清单 + 配置管理 + 资源限制
✅ **性能压测**：Vegeta 分布式压测脚本 + 性能优化指南

