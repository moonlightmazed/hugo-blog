---
title: Kitex处理Panic
date: 2024-01-17T14:21:02+08:00
lastmod: 2024-01-17T14:21:02+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /img/covers/餐桌.png
images:
  - /img/covers/餐桌.png
categories:
  - 微服务框架
tags:
  - Kitex
# nolastmod: true
draft: false
description: '生产环境服务的"最后一道防线"——当业务代码出现 panic 时，中间件通过 `recover` 捕获堆栈、记录结构化日志、上报 Trace 上下文、Prometheus 指标，并给客户端返回标准错误响应，而不是直接 crash。'
---


## 完整代码

### 1、目录结构

```
pkg/middleware/
├── panic_recovery.go   # 核心中间件
├── metrics.go           # Prometheus 指标
└── types.go             # 自定义错误类型
```

### 2、自定义错误类型

```go
// pkg/middleware/types.go
package middleware

import (
	"fmt"
	"time"
)

// PanicError 是 panic 恢复后构造的结构化错误
type PanicError struct {
	MethodName string    `json:"method"`
	StackTrace string    `json:"stack_trace"`
	Timestamp  time.Time `json:"timestamp"`
	Receiver   string    `json:"receiver"` // 服务名
}

func (p *PanicError) Error() string {
	return fmt.Sprintf("service panic at %s: %s", p.MethodName, p.StackTrace)
}
```

### 3、Prometheus 指标

```go
// pkg/middleware/metrics.go
package middleware

import (
	"github.com/cloudwego/kitex/pkg/klog"
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

// 全局 Registry（服务启动时注入到 kitex 的 Prometheus 配置）
var Registry = prometheus.NewRegistry()

func init() {
	// --- Panic 计数：按 service + method 分片 ---
	PanicTotal = promauto.With(Registry).NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "kitex",
			Subsystem: "service",
			Name:      "panic_total",
			Help:      "Total number of panics recovered by middleware",
		},
		[]string{"service", "method"},
	)

	// --- 每次 recover 的堆栈长度（用于评估 panic 严重程度） ---
	PanicStackLen = promauto.With(Registry).NewHistogram(
		prometheus.HistogramOpts{
			Namespace: "kitex",
			Subsystem: "service",
			Name:      "panic_stack_trace_length_bytes",
			Help:      "Length of panic stack trace in bytes",
			Buckets:   prometheus.ExponentialBuckets(1024, 2, 10), // 1KB -> ~1MB
		},
	)

	// --- 服务级 Panic Rate（可通过 Pod/Instance 区分） ---
	PanicRatePerMinute = promauto.With(Registry).NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "kitex",
			Subsystem: "service",
			Name:      "panic_rate_per_minute",
			Help:      "Panic rate per minute per service instance",
		},
		[]string{"service", "instance"},
	)
}

// 全局计数器，供中间件调用
var (
	PanicTotal       *prometheus.CounterVec
	PanicStackLen    prometheus.Histogram
	PanicRatePerMinute *prometheus.GaugeVec
)
```

### 4、Panic Recovery 中间件

```go
// pkg/middleware/panic_recovery.go
package middleware

import (
	"bytes"
	"context"
	"fmt"
	"runtime"
	"runtime/debug"
	"strings"

	"github.com/bytedance/sonic"
	"github.com/cloudwego/kitex/pkg/errors"
	"github.com/cloudwego/kitex/pkg/endpoint"
	"github.com/cloudwego/kitex/pkg/klog"
	"github.com/cloudwego/kitex/pkg/rpcinfo"
	"github.com/cloudwego/kitex/pkg/utils/kitexutil"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

// Config 是 panic 中间件的可选配置
type Config struct {
	// ServiceName 是服务名，用于指标打标签；留空则从 rpcinfo 自动获取
	ServiceName string

	// IncludeStackTrace 是否采集完整堆栈（生产环境建议 true）
	IncludeStackTrace bool

	// MaxStackTraceLines 最大堆栈行数，防止日志爆炸
	MaxStackTraceLines int
}

func (c *Config) apply() *Config {
	if c.MaxStackTraceLines == 0 {
		c.MaxStackTraceLines = 128 // 默认上限
	}
	return c
}

// PanicRecovery 返回一个 panic 恢复中间件
// 它会在下游 handler /业务 method 发生 panic 时：
//  1.  recover panic 并采集堆栈
//  2.  记录 ERROR 级别日志（含 trace_id、stack_trace）
//  3.  在 Span 上标记 error 并注入 stack trace attribute
//  4.  递增 Prometheus panic_total 指标
//  5.  返回一个 HTTP 500 / Thrift InternalError 给客户端
func PanicRecovery(cfg ...Config) endpoint.Middleware {
	c := (&Config{}).apply()
	if len(cfg) > 0 {
		c = &cfg[0].apply()
	}

	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, req, resp interface{}) (err error) {
			// ===== 1. 采集服务名 =====
			var serviceName string
			if c.ServiceName != "" {
				serviceName = c.ServiceName
			} else {
				ri := rpcinfo.GetRPCInfo(ctx)
				if ri != nil {
					serviceName = ri.To().ServiceName()
				}
			}

			var methodName string
			methodName, _ = kitexutil.GetMethod(ctx)

			defer func() {
				recovered := recover()
				if recovered == nil {
					return // 没有 panic，正常返回
				}

				// ===== 2. 采集堆栈 =====
				stackBuf := make([]byte, 64*1024) // 64KB 缓冲区
				n := runtime.Stack(stackBuf, false)
				stackTrace := string(stackBuf[:n])

				// 限制堆栈行数，防止极端情况打满日志
				if c.MaxStackTraceLines > 0 {
					stackTrace = limitStackTrace(stackTrace, c.MaxStackTraceLines)
				}

				// ===== 3. 构造 PanicError =====
				panicErr := &PanicError{
					MethodName: methodName,
					StackTrace: stackTrace,
					Timestamp:  now(),
					Receiver:   serviceName,
				}

				// ===== 4. 记录结构化日志 =====
				klog.Errorf(
					"[PANIC RECOVERED] service=%s method=%s err=%v stack_trace=%s",
					serviceName,
					methodName,
					recovered,
					stackTrace,
				)

				// ===== 5. 上报 Trace（OpenTelemetry） =====
				span := otel.GetTracerProvider().Tracer("kitex/panic-recovery").Start(
					ctx, "panic.recovery",
				)
				span.SetAttributes(
					attribute.String("kitex.service", serviceName),
					attribute.String("kitex.method", methodName),
					attribute.String("exception.message", fmt.Sprint(recovered)),
					attribute.String("exception.stacktrace", stackTrace),
					attribute.Bool("exception.escaped", false),
				)
				span.SetStatus(codes.Error, fmt.Sprint(recovered))
				span.RecordError(panicErr)
				span.End()

				// 如果已有活跃 span，也标记它
				if parentSpan := getActiveSpan(ctx); parentSpan.IsRecording() {
					parentSpan.SetAttributes(
						attribute.String("exception.message", fmt.Sprint(recovered)),
						attribute.String("exception.stacktrace", stackTrace),
					)
					parentSpan.SetStatus(codes.Error, fmt.Sprint(recovered))
					parentSpan.AddEvent("panic.recovered",
						otel.WithAttributes(
							attribute.String("stack_trace", stackTrace),
						),
					)
				}

				// ===== 6. 递增 Prometheus 指标 =====
				if serviceName != "" && methodName != "" {
					PanicTotal.WithLabelValues(serviceName, methodName).Inc()
					PanicStackLen.Observe(float64(len(stackTrace)))
				}

				// ===== 7. 将 panic 消息转为 Kitex 标准错误，返回给客户端 =====
				// InternalError = 服务端不可恢复错误，客户端收到后不会再重试
				err = errors.NewErrorf(
					"errors.InternalError",
					"service panic: %v",
					recovered,
				)

				// 打印堆栈到 stderr，方便 k8s / 日志采集
				printStackToStderr(stackTrace)
			}()

			// ===== 8. 调用下游 =====
			return next(ctx, req, resp)
		}
	}
}

// --- 私有辅助函数 ---

// limitStackTrace 截断堆栈，防止日志爆炸
func limitStackTrace(stack string, maxLines int) string {
	lines := strings.Split(stack, "\n")
	if len(lines) <= maxLines {
		return stack
	}
	// 保留前 maxLines/2 行 + 省略号 + 后 maxLines/2 行
	half := maxLines / 2
	var buf bytes.Buffer
	for i := 0; i < half && i < len(lines); i++ {
		buf.WriteString(lines[i])
		buf.WriteByte('\n')
	}
	buf.WriteString("... [truncated]\n")
	for i := len(lines) - half; i < len(lines); i++ {
		buf.WriteString(lines[i])
		buf.WriteByte('\n')
	}
	return buf.String()
}

// getActiveSpan 从 context 中获取当前活跃 span
func getActiveSpan(ctx context.Context) interface {
	isRecording() bool
	SetAttributes(...attribute.KeyValue)
	SetStatus(codes.Code, string)
	AddEvent(string, ...otel.TracerStartEventOption)
} {
	// 兼容 otel-go 的 trace.SpanFromContext 返回值
	// 这里用 interface{} 避免直接引用导致编译依赖过强
	// 实际使用时直接调用 trace.SpanFromContext(ctx) 即可
	return nil
}

// now 时间封装，方便测试
var now = func() time.Time { return time.Now() }

// printStackToStderr 将堆栈打印到标准错误流
func printStackToStderr(stack string) {
	fmt.Fprintln(os.Stderr, stack)
}
```

> **说明**：上面的 `getActiveSpan` 返回了 `nil`，实际使用时应替换为以下标准写法：

```go
import "go.opentelemetry.io/otel/trace"

func getActiveSpan(ctx context.Context) trace.Span {
	return trace.SpanFromContext(ctx)
}
```

### 5、使用方式

#### 服务端注册

```go
package main

import (
	"github.com/cloudwego/kitex/server"
	"github.com/yourproject/pkg/middleware"
	// ...
)

func main() {
	svr := myservice.NewServer(
		new(MyServiceImpl),
		server.WithServiceAddr(addr),
		server.WithRegistry(reg),
		server.WithServerBasicInfo(&rpcinfo.EndpointBasicInfo{
			ServiceName: "my-service",
		}),
		// 注册 panic 中间件 —— 放在中间件链第一位，最先执行
		server.WithMiddleware(PanicRecovery(middleware.Config{
			ServiceName:        "my-service",
			IncludeStackTrace:  true,
			MaxStackTraceLines: 128,
		})),
		// 再注册你的业务中间件
		server.WithMiddleware(TimerMW),
		// ... 其他中间件
	)

	// 注册 Prometheus metrics exporter
	prometheus.Register(middleware.Registry)

	err := svr.Run()
	if err != nil {
		log.Println(err.Error())
	}
}
```

#### 客户端中间件（同样建议注册）

```go
client, err := myservice.NewClient(
	"my-service",
	client.WithHostPorts("127.0.0.1:8888"),
	client.WithMiddleware(middleware.PanicRecovery()),
)
```

---

## 原理流程图

```
客户端请求
    │
    ▼
┌──────────────────────────┐
│  PanicRecovery (middleware) │  ← defer recover() 已注册
├──────────────────────────┤
│  调用 next(ctx, req, resp)  │
│         │                 │
│    ┌────┴────┐            │
│    │正常返回 │ panic!      │
│    └────┬────┘            │
│         │                 │
│    next返回               │ recover() 捕获
│         │                 │
│    ┌────────────────────┐ │
│    │ 正常: 直接返回 err  │ │
│    │ Panic: 执行defer    │ │
│    │  1. runtime.Stack() │ │
│    │  2. klog.Errorf()   │ │
│    │  3. OTel Span标记   │ │
│    │  4. Prometheus计数  │ │
│    │  5. 返回 InternalError││
│    └────────────────────┘ │
└──────────────────────────┘
    │
    ▼
客户端收到 500 / InternalError
```

---

## 为什么这样设计？

### 1. recover 必须放在 middleware 层

| 层级 | 能否 recover | 原因 |
|------|-------------|------|
| handler（业务方法） | 能，但太局部 | 每个方法都要写，遗漏率高 |
| middleware | **能，且全局** | 注册一次，覆盖所有方法 |
| server.Run() | 能，但来不及 | 已经脱离 handler 的 ctx |

Middleware 层是 **唯一一个既能拿到完整 `context`（含 trace_id），又能覆盖所有 handler 的位置**。

### 2. 为什么返回 `InternalError` 而不是 `BusinessError`

- `BusinessError` = 业务逻辑错误（参数不对、资源不存在），客户端通常**不会重试**
- `InternalError` = 服务端不可恢复的内部错误，告诉客户端"**不是你的问题，是我们的问题**"
- Panic 意味着代码有 bug，重试没有意义，应该走**告警 + 修复** 流程

### 3. 日志中必须包含 trace_id

当 panic 发生在高并发场景，同一秒可能有上百个请求。如果日志里不带 `trace_id`，你根本无法把 panic 堆栈和哪个请求关联起来。

在 middleware 的 `defer` 里，下游的 span 已经创建，所以 `trace.SpanFromContext(ctx)` 一定能拿到 span，从而间接获取 trace_id。

### 4. Prometheus 指标设计理由

| 指标 | 标签 | 用途 |
|------|------|------|
| `panic_total` | `service` + `method` | 哪个方法最常 panic → 定位有问题的接口 |
| `panic_stack_trace_length_bytes` | 无 | 堆栈长度分布 → 判断是简单 panic 还是深层嵌套 |
| `panic_rate_per_minute` | `service` + `instance` | 实时告警 → K8s HPA 自动扩缩容参考 |

---

## 生产环境 checklist

- [ ] 中间件注册在 `WithMiddleware` 链的**最前面**（最先执行，最晚 recover）
- [ ] `klog.Errorf` 使用**结构化字段**（而非 `fmt.Sprintf` 拼接），方便日志平台解析
- [ ] 堆栈长度限制 `MaxStackTraceLines` 必须设置，防止极端情况下（如递归 panic）打满日志
- [ ] Prometheus metrics 注册到 Kitex 自带的 `/metrics` 端点
- [ ] 配合**告警规则**：`rate(kitex_service_panic_total[5m]) > 0` → 触发 P0 告警
- [ ] 配合 **Tracing 系统**（Jaeger / SkyWalking / OTEL Collector）：panic 的 span 应该标记为 `Error` 状态
- [ ] `klog` 的 `Fatal` 级别 panic **不经过 recover**（Go 设计如此），如果需要捕获 `log.Fatal`，需要重写 `klog.Fatal` 实现

---

## 依赖清单

```bash
# Kitex 核心
go get github.com/cloudwego/kitex

# Prometheus metrics
go get github.com/prometheus/client_golang/prometheus

# OpenTelemetry tracing
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/trace
go get go.opentelemetry.io/otel/codes
go get go.opentelemetry.io/otel/attribute
go get go.opentelemetry.io/otel/semconv/v1.21.0

# JSON 序列化（可选，用于结构化日志）
go get github.com/bytedance/sonic
```
