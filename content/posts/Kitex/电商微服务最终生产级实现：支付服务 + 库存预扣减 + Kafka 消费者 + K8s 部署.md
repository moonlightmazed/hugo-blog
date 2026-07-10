---
title: 电商微服务最终生产级实现：支付服务 + 库存预扣减 + Kafka 消费者 + K8s 部署
date: 2025-01-02T14:21:02+08:00
lastmod: 2025-01-02T14:21:02+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /img/covers/自行车.png
images:
  - /img/covers/自行车.png
categories:
  - 微服务框架
tags:
  - Kitex
  - 示例
# nolastmod: true
draft: false
description: "新增支付宝 / 微信支付模拟、库存预扣减 + 超时自动释放、Kafka 异步消费者、K8s 完整部署清单，实现高可用、高并发、最终一致性的完整电商闭环"
---

## 一、支付服务 (PaymentService) 完整实现

### 1. 支付 IDL idl/payment.thrift
```thrift
namespace go payment

struct CreatePaymentRequest {
    1: required i64 order_id
    2: required double amount
    3: required i32 pay_method // 1:支付宝 2:微信
}

struct CreatePaymentResponse {
    1: required i64 payment_id
    2: required string pay_url // 支付链接
}

struct GetPaymentStatusRequest {
    1: required i64 payment_id
}

struct GetPaymentStatusResponse {
    1: required i32 status // 1:待支付 2:支付成功 3:支付失败 4:已退款
}

service PaymentService {
    CreatePaymentResponse CreatePayment(1: CreatePaymentRequest req)
    GetPaymentStatusResponse GetPaymentStatus(1: GetPaymentStatusRequest req)
}
```

### 2. 数据库模型 internal/paymentservice/model/payment.go
```go
package model

import (
    "time"
    "gorm.io/gorm"
)

// Payment 支付记录表
type Payment struct {
    ID            uint64         `gorm:"primaryKey;autoIncrement"`
    OrderID       uint64         `gorm:"uniqueIndex;not null"`
    Amount        float64        `gorm:"type:decimal(10,2);not null"`
    PayMethod     int32          `gorm:"not null"` // 1:支付宝 2:微信
    Status        int32          `gorm:"not null;default:1"` // 1:待支付
    TransactionID string         `gorm:"type:varchar(64)"` // 第三方交易号
    CreatedAt     time.Time
    UpdatedAt     time.Time
    DeletedAt     gorm.DeletedAt `gorm:"index"`
}

func (Payment) TableName() string { return "payments" }
```

### 3. DAO 层 internal/paymentservice/dao/payment_dao.go
```go
package dao

import (
    "context"
    "gorm.io/gorm"
    "github.com/yourname/cloudwego-mall/internal/paymentservice/model"
    "github.com/yourname/cloudwego-mall/pkg/mysql"
    "github.com/yourname/cloudwego-mall/pkg/errors"
)

type PaymentDAO struct {
    db *gorm.DB
}

func NewPaymentDAO() *PaymentDAO {
    return &PaymentDAO{db: mysql.GetDB()}
}

// Create 创建支付记录
func (d *PaymentDAO) Create(ctx context.Context, payment *model.Payment) error {
    if err := d.db.WithContext(ctx).Create(payment).Error; err != nil {
        return errors.New(errors.CodeInternalError, "创建支付记录失败")
    }
    return nil
}

// GetByID 根据ID查询支付记录
func (d *PaymentDAO) GetByID(ctx context.Context, id uint64) (*model.Payment, error) {
    var payment model.Payment
    if err := d.db.WithContext(ctx).First(&payment, id).Error; err != nil {
        if err == gorm.ErrRecordNotFound {
            return nil, errors.New(errors.CodePaymentNotFound, "支付记录不存在")
        }
        return nil, errors.New(errors.CodeInternalError, "查询支付记录失败")
    }
    return &payment, nil
}

// UpdateStatus 更新支付状态
func (d *PaymentDAO) UpdateStatus(ctx context.Context, id uint64, status int32, transactionID string) error {
    updates := map[string]interface{}{
        "status": status,
    }
    if transactionID != "" {
        updates["transaction_id"] = transactionID
    }

    result := d.db.WithContext(ctx).Model(&model.Payment{}).Where("id = ?", id).Updates(updates)
    if result.Error != nil {
        return errors.New(errors.CodeInternalError, "更新支付状态失败")
    }
    return nil
}
```

### 4. Service 层（含模拟支付）internal/paymentservice/service/payment_service.go
```go
package service

import (
    "context"
    "fmt"
    "github.com/yourname/cloudwego-mall/api/kitex_gen/payment"
    "github.com/yourname/cloudwego-mall/internal/paymentservice/dao"
    "github.com/yourname/cloudwego-mall/internal/paymentservice/model"
    "github.com/yourname/cloudwego-mall/pkg/errors"
    "github.com/yourname/cloudwego-mall/pkg/logger"
    "go.uber.org/zap"
)

type PaymentServiceImpl struct {
    paymentDAO *dao.PaymentDAO
}

func NewPaymentServiceImpl() *PaymentServiceImpl {
    return &PaymentServiceImpl{paymentDAO: dao.NewPaymentDAO()}
}

// CreatePayment 创建支付订单
func (s *PaymentServiceImpl) CreatePayment(ctx context.Context, req *payment.CreatePaymentRequest) (*payment.CreatePaymentResponse, error) {
    logger.Info("CreatePayment request", zap.Int64("order_id", req.OrderId), zap.Float64("amount", req.Amount))

    // 创建支付记录
    paymentModel := &model.Payment{
        OrderID:   uint64(req.OrderId),
        Amount:    req.Amount,
        PayMethod: req.PayMethod,
        Status:    1, // 待支付
    }

    if err := s.paymentDAO.Create(ctx, paymentModel); err != nil {
        return nil, err
    }

    // 生成模拟支付链接（实际项目中调用支付宝/微信统一下单接口）
    payURL := fmt.Sprintf("https://pay.example.com?payment_id=%d&amount=%.2f", paymentModel.ID, req.Amount)

    return &payment.CreatePaymentResponse{
        PaymentId: int64(paymentModel.ID),
        PayUrl:    payURL,
    }, nil
}

// GetPaymentStatus 查询支付状态
func (s *PaymentServiceImpl) GetPaymentStatus(ctx context.Context, req *payment.GetPaymentStatusRequest) (*payment.GetPaymentStatusResponse, error) {
    paymentModel, err := s.paymentDAO.GetByID(ctx, uint64(req.PaymentId))
    if err != nil {
        return nil, err
    }

    return &payment.GetPaymentStatusResponse{
        Status: paymentModel.Status,
    }, nil
}

// 模拟支付回调（实际项目中由第三方支付平台调用）
func (s *PaymentServiceImpl) MockPaymentCallback(ctx context.Context, paymentID uint64, success bool) error {
    var status int32
    var transactionID string

    if success {
        status = 2 // 支付成功
        transactionID = fmt.Sprintf("TXN%d", paymentID)
    } else {
        status = 3 // 支付失败
    }

    return s.paymentDAO.UpdateStatus(ctx, paymentID, status, transactionID)
}
```

### 5. 支付服务启动文件 cmd/paymentservice/main.go
```go
package main

import (
    "context"
    "net"
    "github.com/cloudwego/kitex/pkg/rpcinfo"
    "github.com/cloudwego/kitex/server"
    "github.com/yourname/cloudwego-mall/api/kitex_gen/payment/paymentservice"
    "github.com/yourname/cloudwego-mall/internal/paymentservice/service"
    "github.com/yourname/cloudwego-mall/pkg/config"
    "github.com/yourname/cloudwego-mall/pkg/logger"
    "github.com/yourname/cloudwego-mall/pkg/discovery"
    "github.com/yourname/cloudwego-mall/pkg/otel"
    "github.com/yourname/cloudwego-mall/pkg/sentinel"
    "github.com/yourname/cloudwego-mall/pkg/mysql"
    "go.uber.org/zap"
)

func main() {
    if err := config.Init("configs/paymentservice.yaml"); err != nil {
        panic(err)
    }

    logger.Init(config.GlobalConfig.Server.Name)
    mysql.Init()

    otelProvider := otel.InitProvider(
        config.GlobalConfig.Server.Name,
        config.GlobalConfig.Jaeger.Endpoint,
    )
    defer otelProvider.Shutdown(context.Background())

    r, err := discovery.NewEtcdRegistry(config.GlobalConfig.Etcd.Endpoints)
    if err != nil {
        logger.Fatal("etcd registry init failed", zap.Error(err))
    }

    svr := paymentservice.NewServer(
        service.NewPaymentServiceImpl(),
        server.WithServiceAddr(&net.TCPAddr{IP: net.ParseIP("0.0.0.0"), Port: 8085}),
        server.WithRegistry(r),
        server.WithServerBasicInfo(&rpcinfo.EndpointBasicInfo{
            ServiceName: config.GlobalConfig.Server.Name,
        }),
        otel.KitexServerOption(config.GlobalConfig.Server.Name),
        sentinel.KitexServerMiddleware(),
    )

    logger.Info("paymentservice started on :8085")
    if err := svr.Run(); err != nil {
        logger.Fatal("server run failed", zap.Error(err))
    }
}
```
---

## 二、库存预扣减 + 超时自动释放（高并发防超卖）
### 1. 修改商品服务 DAO 层 internal/productservice/dao/product_dao.go
```go
// 新增：预扣减库存（Redis+数据库双写）
func (d *ProductDAO) PreDeductStock(ctx context.Context, productID uint64, quantity int64) error {
    // 1. 先查Redis库存
    stockKey := fmt.Sprintf("stock:%d", productID)
    currentStock, err := redis.Client.DecrBy(ctx, stockKey, quantity).Result()
    if err != nil {
        return errors.New(errors.CodeInternalError, "扣减Redis库存失败")
    }

    // 2. Redis库存不足，回滚并返回错误
    if currentStock < 0 {
        redis.Client.IncrBy(ctx, stockKey, quantity)
        return errors.New(errors.CodeStockNotEnough, "库存不足")
    }

    // 3. 数据库预扣减（使用乐观锁）
    result := d.db.WithContext(ctx).Model(&model.Stock{}).
        Where("product_id = ? AND stock >= ?", productID, quantity).
        Update("stock", gorm.Expr("stock - ?", quantity))

    if result.Error != nil {
        // 数据库扣减失败，回滚Redis
        redis.Client.IncrBy(ctx, stockKey, quantity)
        return errors.New(errors.CodeInternalError, "扣减数据库库存失败")
    }

    if result.RowsAffected == 0 {
        // 数据库库存不足，回滚Redis
        redis.Client.IncrBy(ctx, stockKey, quantity)
        return errors.New(errors.CodeStockNotEnough, "库存不足")
    }

    // 4. 设置库存预扣减过期时间（15分钟）
    lockKey := fmt.Sprintf("stock_lock:%d:%d", productID, quantity)
    redis.Client.SetEX(ctx, lockKey, "1", 15*time.Minute)

    return nil
}

// 新增：确认扣减库存（支付成功后调用）
func (d *ProductDAO) ConfirmDeductStock(ctx context.Context, productID uint64, quantity int64) error {
    // 删除预扣减锁
    lockKey := fmt.Sprintf("stock_lock:%d:%d", productID, quantity)
    redis.Client.Del(ctx, lockKey)
    return nil
}

// 新增：释放库存（支付超时/取消订单时调用）
func (d *ProductDAO) ReleaseStock(ctx context.Context, productID uint64, quantity int64) error {
    // 1. 数据库恢复库存
    result := d.db.WithContext(ctx).Model(&model.Stock{}).
        Where("product_id = ?", productID).
        Update("stock", gorm.Expr("stock + ?", quantity))

    if result.Error != nil {
        return errors.New(errors.CodeInternalError, "恢复数据库库存失败")
    }

    // 2. Redis恢复库存
    stockKey := fmt.Sprintf("stock:%d", productID)
    redis.Client.IncrBy(ctx, stockKey, quantity)

    // 3. 删除预扣减锁
    lockKey := fmt.Sprintf("stock_lock:%d:%d", productID, quantity)
    redis.Client.Del(ctx, lockKey)

    return nil
}
```

### 2. 修改商品服务 Service 层 internal/productservice/service/product_service.go
```go
// PreDeductStock 预扣减库存
func (s *ProductServiceImpl) PreDeductStock(ctx context.Context, req *product.PreDeductStockRequest) (*product.PreDeductStockResponse, error) {
    productID := uint64(req.ProductId)
    logger.Info("PreDeductStock request", zap.Uint64("product_id", productID), zap.Int64("quantity", req.Quantity))

    if err := s.productDAO.PreDeductStock(ctx, productID, req.Quantity); err != nil {
        return nil, err
    }

    return &product.PreDeductStockResponse{Success: true}, nil
}

// ConfirmDeductStock 确认扣减库存
func (s *ProductServiceImpl) ConfirmDeductStock(ctx context.Context, req *product.ConfirmDeductStockRequest) (*product.ConfirmDeductStockResponse, error) {
    productID := uint64(req.ProductId)
    logger.Info("ConfirmDeductStock request", zap.Uint64("product_id", productID), zap.Int64("quantity", req.Quantity))

    if err := s.productDAO.ConfirmDeductStock(ctx, productID, req.Quantity); err != nil {
        return nil, err
    }

    return &product.ConfirmDeductStockResponse{Success: true}, nil
}

// ReleaseStock 释放库存
func (s *ProductServiceImpl) ReleaseStock(ctx context.Context, req *product.ReleaseStockRequest) (*product.ReleaseStockResponse, error) {
    productID := uint64(req.ProductId)
    logger.Info("ReleaseStock request", zap.Uint64("product_id", productID), zap.Int64("quantity", req.Quantity))

    if err := s.productDAO.ReleaseStock(ctx, productID, req.Quantity); err != nil {
        return nil, err
    }

    return &product.ReleaseStockResponse{Success: true}, nil
}
```


### 3. 库存超时自动释放消费者 internal/productservice/consumer/stock_consumer.go
```go
package consumer

import (
    "context"
    "encoding/json"
    "github.com/segmentio/kafka-go"
    "github.com/yourname/cloudwego-mall/api/kitex_gen/product"
    "github.com/yourname/cloudwego-mall/internal/api-gateway/client"
    "github.com/yourname/cloudwego-mall/pkg/logger"
    "go.uber.org/zap"
)

// StockReleaseConsumer 库存释放消费者
type StockReleaseConsumer struct {
    reader *kafka.Reader
}

func NewStockReleaseConsumer(brokers []string, topic string) *StockReleaseConsumer {
    return &StockReleaseConsumer{
        reader: kafka.NewReader(kafka.ReaderConfig{
            Brokers: brokers,
            Topic:   topic,
            GroupID: "stock-release-group",
        }),
    }
}

func (c *StockReleaseConsumer) Start(ctx context.Context) {
    logger.Info("stock release consumer started")
    for {
        select {
        case <-ctx.Done():
            c.reader.Close()
            return
        default:
            msg, err := c.reader.ReadMessage(ctx)
            if err != nil {
                logger.Error("read kafka message failed", zap.Error(err))
                continue
            }

            var event map[string]interface{}
            if err := json.Unmarshal(msg.Value, &event); err != nil {
                logger.Error("unmarshal event failed", zap.Error(err))
                continue
            }

            // 处理订单超时事件
            if event["event_type"] == "order_timeout" {
                orderID := uint64(event["order_id"].(float64))
                items := event["items"].([]interface{})

                for _, item := range items {
                    itemMap := item.(map[string]interface{})
                    productID := uint64(itemMap["product_id"].(float64))
                    quantity := int64(itemMap["quantity"].(float64))

                    // 调用商品服务释放库存
                    _, err := client.ProductClient.ReleaseStock(ctx, &product.ReleaseStockRequest{
                        ProductId: int64(productID),
                        Quantity:  quantity,
                    })
                    if err != nil {
                        logger.Error("release stock failed", zap.Uint64("order_id", orderID), zap.Error(err))
                    }
                }

                logger.Info("stock released for timeout order", zap.Uint64("order_id", orderID))
            }
        }
    }
}

```

---

## 三、Kafka 异步消费者完整实现
### 1. 订单事件消费者 internal/orderservice/consumer/order_consumer.go
```go
package consumer

import (
    "context"
    "encoding/json"
    "github.com/segmentio/kafka-go"
    "github.com/yourname/cloudwego-mall/pkg/logger"
    "go.uber.org/zap"
)

// OrderEventConsumer 订单事件消费者
type OrderEventConsumer struct {
    reader *kafka.Reader
}

func NewOrderEventConsumer(brokers []string, topic string) *OrderEventConsumer {
    return &OrderEventConsumer{
        reader: kafka.NewReader(kafka.ReaderConfig{
            Brokers: brokers,
            Topic:   topic,
            GroupID: "order-event-group",
        }),
    }
}

func (c *OrderEventConsumer) Start(ctx context.Context) {
    logger.Info("order event consumer started")
    for {
        select {
        case <-ctx.Done():
            c.reader.Close()
            return
        default:
            msg, err := c.reader.ReadMessage(ctx)
            if err != nil {
                logger.Error("read kafka message failed", zap.Error(err))
                continue
            }

            var event map[string]interface{}
            if err := json.Unmarshal(msg.Value, &event); err != nil {
                logger.Error("unmarshal event failed", zap.Error(err))
                continue
            }

            // 根据事件类型处理
            switch event["event_type"] {
            case "order_created":
                c.handleOrderCreated(ctx, event)
            case "order_paid":
                c.handleOrderPaid(ctx, event)
            case "order_cancelled":
                c.handleOrderCancelled(ctx, event)
            }
        }
    }
}

func (c *OrderEventConsumer) handleOrderCreated(ctx context.Context, event map[string]interface{}) {
    orderID := uint64(event["order_id"].(float64))
    logger.Info("handling order created event", zap.Uint64("order_id", orderID))

    // 发送订单创建通知（短信/邮件）
    // 统计订单数据
    // 其他异步任务
}

func (c *OrderEventConsumer) handleOrderPaid(ctx context.Context, event map[string]interface{}) {
    orderID := uint64(event["order_id"].(float64))
    logger.Info("handling order paid event", zap.Uint64("order_id", orderID))

    // 发送支付成功通知
    // 通知仓库发货
    // 更新用户积分
}

func (c *OrderEventConsumer) handleOrderCancelled(ctx context.Context, event map[string]interface{}) {
    orderID := uint64(event["order_id"].(float64))
    logger.Info("handling order cancelled event", zap.Uint64("order_id", orderID))

    // 发送订单取消通知
    // 退款处理
}
```

### 2. 在订单服务启动文件中启动消费者
```go
// cmd/orderservice/main.go 新增
func main() {
    // ... 其他初始化代码

    // 启动Kafka消费者
    consumer := consumer.NewOrderEventConsumer(
        config.GlobalConfig.Kafka.Brokers,
        config.GlobalConfig.Kafka.Topic,
    )
    go consumer.Start(context.Background())

    // ... 启动服务
}

```

## 四、K8s 完整部署配置
### 1. 命名空间 k8s/namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cloudwego-mall
```

### 2. 配置管理 k8s/configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mall-config
  namespace: cloudwego-mall
data:
  config.yaml: |
    etcd:
      endpoints:
        - etcd:2379
    jaeger:
      endpoint: http://jaeger:4317
    sentinel:
      dashboard: sentinel:8858
    redis:
      address: redis:6379
      password: ""
      db: 0
    kafka:
      brokers:
        - kafka:9092
      topic: order_events
    seata:
      service:
        grouplist:
          default: seata:8091
```

### 3. 数据库部署 k8s/mysql.yaml
```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: cloudwego-mall
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
        - name: MYSQL_DATABASE
          value: cloudwego_mall
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        - name: init-script
          mountPath: /docker-entrypoint-initdb.d
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-pvc
      - name: init-script
        configMap:
          name: mall-init-scripts
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: cloudwego-mall
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

### 4. 微服务部署示例（以 userservice 为例）k8s/userservice.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: userservice
  namespace: cloudwego-mall
spec:
  replicas: 2
  selector:
    matchLabels:
      app: userservice
  template:
    metadata:
      labels:
        app: userservice
    spec:
      containers:
      - name: userservice
        image: your-docker-registry/userservice:latest
        ports:
        - containerPort: 8081
        volumeMounts:
        - name: config-volume
          mountPath: /app/configs
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          tcpSocket:
            port: 8081
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          tcpSocket:
            port: 8081
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config-volume
        configMap:
          name: mall-config
---
apiVersion: v1
kind: Service
metadata:
  name: userservice
  namespace: cloudwego-mall
spec:
  selector:
    app: userservice
  ports:
  - port: 8081
    targetPort: 8081
```

### 5. API 网关部署 k8s/api-gateway.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: cloudwego-mall
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: your-docker-registry/api-gateway:latest
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: config-volume
          mountPath: /app/configs
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config-volume
        configMap:
          name: mall-config
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: cloudwego-mall
spec:
  type: LoadBalancer
  selector:
    app: api-gateway
  ports:
  - port: 80
    targetPort: 8080
```

### 6. 一键部署脚本 k8s/deploy.sh

```bash
#!/bin/bash

# 创建命名空间
kubectl apply -f namespace.yaml

# 部署基础设施
kubectl apply -f mysql.yaml
kubectl apply -f redis.yaml
kubectl apply -f etcd.yaml
kubectl apply -f kafka.yaml
kubectl apply -f jaeger.yaml
kubectl apply -f sentinel.yaml
kubectl apply -f seata.yaml

# 等待基础设施就绪
echo "Waiting for infrastructure to be ready..."
sleep 60

# 部署微服务
kubectl apply -f userservice.yaml
kubectl apply -f productservice.yaml
kubectl apply -f cartservice.yaml
kubectl apply -f orderservice.yaml
kubectl apply -f paymentservice.yaml

# 部署API网关
kubectl apply -f api-gateway.yaml

echo "All services deployed successfully!"
```

---

## 五、完整电商支付流程测试
```bash
# 1. 登录获取token
TOKEN=$(curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"123456"}' | jq -r '.data.token')

# 2. 创建订单（预扣减库存）
ORDER_ID=$(curl -X POST http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "items": [
      {
        "product_id": 1,
        "product_name": "iPhone 15 Pro",
        "quantity": 1,
        "price": 7999.00
      }
    ]
  }' | jq -r '.data.order_id')

# 3. 创建支付订单
PAYMENT_ID=$(curl -X POST http://localhost:8080/api/v1/payments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"order_id\": $ORDER_ID, \"amount\": 7999.00, \"pay_method\": 1}" | jq -r '.data.payment_id')

# 4. 模拟支付成功
curl -X POST http://localhost:8080/api/v1/payments/$PAYMENT_ID/callback \
  -H "Content-Type: application/json" \
  -d '{"success": true}'

# 5. 查询订单状态（已支付）
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/v1/orders/$ORDER_ID
```

---

## 六、项目最终完整架构
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   API Gateway   │────│  User Service   │────│     MySQL       │
│   (Hertz)       │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                      │                      │
        │                      │                      │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Cart Service   │────│ Product Service │────│     Redis       │
│                 │    │                 │    │ (缓存+库存锁)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                      │                      │
        │                      │                      │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Order Service  │────│ Payment Service │────│     Kafka       │
│                 │    │                 │    │ (异步消息)        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                      │                      │
        └──────────────────────┼──────────────────────┘
                               │
                        ┌─────────────────┐
                        │     Seata       │
                        │ (分布式事务)      │
                        └─────────────────┘
```

---

## 七、生产级特性总结
✅ 5 个核心微服务：用户、购物车、商品、订单、支付
✅ 完整电商闭环：注册→登录→加购→下单→预扣减→支付→确认扣减
✅ 高并发防超卖：Redis + 数据库双写预扣减 + 超时自动释放
✅ 分布式事务：Seata AT 模式保证订单 - 库存 - 支付一致性
✅ 异步解耦：Kafka 处理订单事件、通知、统计等异步任务
✅ 全链路可观测：OpenTelemetry+Jaeger 链路追踪 + Zap 结构化日志
✅ 高可用保障：Sentinel 熔断限流 + K8s 多副本部署 + 健康检查
✅ 生产级部署：完整 K8s 部署清单 + 配置管理 + 资源限制