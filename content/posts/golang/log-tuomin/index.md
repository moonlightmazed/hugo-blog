---
title: 日志脱敏
date: 2023-06-12T14:21:02+08:00
lastmod: 2023-06-18T14:22:02+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: image.png
images:
  - image.png
categories:
  - Golang
tags:
  - log
# nolastmod: true
draft: false
description: "在企业级日志系统中，日志脱敏是保障数据安全、防止隐私泄露以合规要求的核心功能。"
---

# 敏感字段自动脱敏（Zap 自定义 Core 方案）

## 1、方案选型与核心原理

Zap 原生 `zap.Hooks` 只能获取日志元信息（级别、消息、调用栈），无法拦截和修改结构化 Fields，因此**生产级脱敏必须通过自定义 `zapcore.Core` 实现**。

核心思路：

- 用装饰器模式包装原生 Core，在日志真正写入前拦截所有 Field
- 采用**Key 名匹配 + 值正则匹配**双重脱敏规则，覆盖显式敏感字段和隐式敏感值
- 默认仅处理顶层 String 字段，避免递归反射损耗性能；深度脱敏作为可选项
- 先拷贝 Field 切片再修改，彻底避免并发写冲突

## 2、预定义脱敏规则与工具函数

先统一脱敏规则、预编译正则，保证全公司格式一致，同时最大化性能。

```go
import (
    "regexp"
    "strings"
    "go.uber.org/zap/zapcore"
)

// 预编译正则：全局只编译一次，避免性能损耗
var (
    // 手机号正则（11位、1开头）
    mobileRegex = regexp.MustCompile(`^1[3-9]\d{9}$`)
    // 18位身份证号
    idCardRegex = regexp.MustCompile(`^\d{17}[\dXx]$`)
    // 银行卡号（16-19位数字）
    bankCardRegex = regexp.MustCompile(`^\d{16,19}$`)
    // 邮箱正则
    emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
)

// 敏感Key集合：O(1)匹配，命中即全脱敏
var sensitiveKeyMap = map[string]struct{}{
    "password":     {},
    "pwd":          {},
    "token":        {},
    "access_token": {},
    "secret":       {},
    "secret_key":   {},
    "authorization":{},
    "cookie":       {},
    "mobile":       {},
    "phone":        {},
    "id_card":      {},
    "idcard":       {},
    "bank_card":    {},
    "email":        {},
    "id_no":        {},
}

// 脱敏处理函数
func desensitizeValue(key, value string) string {
    keyLower := strings.ToLower(key)

    // 1. Key命中敏感集合：按类型脱敏
    if _, ok := sensitiveKeyMap[keyLower]; ok {
        switch {
        case mobileRegex.MatchString(value):
            return value[:3] + "****" + value[7:]
        case idCardRegex.MatchString(value):
            return value[:6] + "********" + value[14:]
        case bankCardRegex.MatchString(value):
            return value[:4] + "********" + value[len(value)-4:]
        case emailRegex.MatchString(value):
            parts := strings.Split(value, "@")
            if len(parts) == 2 {
                prefix := parts[0]
                if len(prefix) <= 1 {
                    return "*@" + parts[1]
                }
                return prefix[:1] + "****@" + parts[1]
            }
        }
        // 密码、Token等非结构化敏感值：全掩码
        return "***"
    }

    // 2. Key未命中：值正则兜底匹配（防止漏脱敏）
    switch {
    case mobileRegex.MatchString(value):
        return value[:3] + "****" + value[7:]
    case idCardRegex.MatchString(value):
        return value[:6] + "********" + value[14:]
    }

    return value
}
```

## 3、自定义脱敏 Core 完整实现

```go
// DesensitizeCore 脱敏Core装饰器
type DesensitizeCore struct {
    zapcore.Core
    enable bool // 脱敏开关
}

// With 继承字段时保持包装
func (c *DesensitizeCore) With(fields []zapcore.Field) zapcore.Core {
    if !c.enable {
        return c.Core.With(fields)
    }
    // 先脱敏再传给底层Core
    desensitizedFields := c.doDesensitize(fields)
    return &DesensitizeCore{
        Core:   c.Core.With(desensitizedFields),
        enable: c.enable,
    }
}

// Check 委托给底层Core判断
func (c *DesensitizeCore) Check(entry zapcore.Entry, ce *zapcore.CheckedEntry) *zapcore.CheckedEntry {
    if c.Enabled(entry.Level) {
        return ce.AddCore(entry, c)
    }
    return ce
}

// Write 核心：写入前执行脱敏
func (c *DesensitizeCore) Write(entry zapcore.Entry, fields []zapcore.Field) error {
    if !c.enable {
        return c.Core.Write(entry, fields)
    }

    // 必须拷贝切片，禁止修改原切片，避免并发数据竞争
    desensitizedFields := c.doDesensitize(fields)
    return c.Core.Write(entry, desensitizedFields)
}

// 执行脱敏：仅处理String类型顶层字段，兼顾性能与合规
func (c *DesensitizeCore) doDesensitize(fields []zapcore.Field) []zapcore.Field {
    if len(fields) == 0 {
        return fields
    }

    // 拷贝切片，不污染原数据
    newFields := make([]zapcore.Field, len(fields))
    copy(newFields, fields)

    for i := range newFields {
        field := &newFields[i]
        // 仅处理String类型字段，跳过数字、布尔等
        if field.Type == zapcore.StringType {
            field.String = desensitizeValue(field.Key, field.String)
        }
    }
    return newFields
}
```

## 4、集成到现有日志构建链路

在原 `buildMultiCores` 函数中，对每个 Core 统一包装一层脱敏装饰器，通过配置开关控制。

```go
// LogConfig 新增脱敏开关
type LogConfig struct {
    // ... 原有字段保持不变
    Desensitize bool // 是否开启敏感字段脱敏
}

func buildMultiCores(cfg *LogConfig, encoder zapcore.Encoder, atomicLevel zap.AtomicLevel) ([]zapcore.Core, error) {
    var cores []zapcore.Core

    // 1. 主日志Core（原有逻辑不变）
    mainSyncer, _ := buildRotateSyncer(...)
    mainLevelEnabler := zap.LevelEnablerFunc(func(l zapcore.Level) bool {
        return atomicLevel.Enabled(l) && l < zap.ErrorLevel
    })
    mainCore := zapcore.NewCore(encoder, mainSyncer, mainLevelEnabler)
    samplingMainCore := zapcore.NewSamplerWithOptions(mainCore, time.Second, 100, 100)

    // 包装脱敏Core
    if cfg.Desensitize {
        samplingMainCore = &DesensitizeCore{Core: samplingMainCore, enable: true}
    }
    cores = append(cores, samplingMainCore)

    // 2. 错误日志Core（原有逻辑不变）
    errorSyncer, _ := buildRotateSyncer(...)
    errorLevelEnabler := zap.LevelEnablerFunc(func(l zapcore.Level) bool {
        return l >= zap.ErrorLevel
    })
    errorCore := zapcore.NewCore(encoder, errorSyncer, errorLevelEnabler)

    // 错误日志同样脱敏
    if cfg.Desensitize {
        errorCore = &DesensitizeCore{Core: errorCore, enable: true}
    }
    cores = append(cores, errorCore)

    // 3. 控制台Core
    if cfg.Console {
        consoleCore := zapcore.NewCore(encoder, zapcore.AddSync(os.Stdout), atomicLevel)
        if cfg.Desensitize {
            consoleCore = &DesensitizeCore{Core: consoleCore, enable: true}
        }
        cores = append(cores, consoleCore)
    }

    return cores, nil
}
```

## 5、进阶：结构化对象深度脱敏

如果需要对 `zap.Any("user", user)` 传入的结构体 / Map 做深度递归脱敏，可基于反射扩展 `doDesensitize` 方法，但需注意：

- 反射会带来 3~5 倍性能损耗，高 QPS 服务不推荐开启
- 建议仅对明确标注 `log:"desensitize"` tag 的字段做脱敏
- 更优实践：业务层在打印前手动处理敏感字段，日志层仅做兜底校验

## 6、脱敏核心注意事项

- **并发安全**：绝对禁止直接修改传入的 Field 切片，必须拷贝后再修改，否则高并发下会出现数据竞争、日志乱码。
- **性能优先**：正则预编译、Key 用 Map 查找、仅处理 String 字段，三者缺一不可，避免脱敏成为性能瓶颈。
- **白名单机制**：`trace_id`、`service`、`span_id` 等排障核心字段禁止加入脱敏规则，必须保障链路可追踪。
- **规则统一**：全公司统一敏感 Key 集合与脱敏格式，避免日志平台检索、告警规则失效。
