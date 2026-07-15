---
title: 加密
date: 2024-12-05T14:21:02+08:00
lastmod: 2024-12-05T14:21:02+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /img/covers/同心锁.png
images:
  - /img/covers/同心锁.png
categories:
  - Golang
tags:
  - 加密
# nolastmod: true
draft: false
description: "在 Go 语言中，目前（2026年）密码加密的最佳实践依然是使用 Argon2id，其次是 bcrypt"
---

## 推荐实现：Argon2id

使用官方维护的 golang.org/x/crypto/argon2 包：

```go
package password

import (
    "crypto/rand"
    "crypto/subtle"
    "encoding/base64"
    "errors"
    "fmt"
    "strings"

    "golang.org/x/crypto/argon2"
)

// OWASP 2023 推荐参数（根据服务器性能调整）
const (
    memory      = 64 * 1024 // 64 MB
    iterations  = 3
    parallelism = 2
    saltLength  = 16
    keyLength   = 32
)

func Hash(password string) (string, error) {
    salt := make([]byte, saltLength)
    if _, err := rand.Read(salt); err != nil {
        return "", fmt.Errorf("generate salt: %w", err)
    }

    hash := argon2.IDKey([]byte(password), salt, iterations, memory, parallelism, keyLength)

    // 格式: $argon2id$v=19$m=65536,t=3,p=2$<salt>$<hash>
    encoded := fmt.Sprintf("$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
        argon2.Version, memory, iterations, parallelism,
        base64.RawStdEncoding.EncodeToString(salt),
        base64.RawStdEncoding.EncodeToString(hash),
    )
    return encoded, nil
}

func Verify(password, encodedHash string) (bool, error) {
    // 解析编码字符串提取参数和 salt/hash
    parts := strings.Split(encodedHash, "$")
    if len(parts) != 6 {
        return false, errors.New("invalid hash format")
    }

    var version int
    var memory uint32
    var iterations uint32
    var parallelism uint8
    fmt.Sscanf(parts[3], "m=%d,t=%d,p=%d", &memory, &iterations, &parallelism)
    fmt.Sscanf(parts[2], "v=%d", &version)

    salt, _ := base64.RawStdEncoding.DecodeString(parts[4])
    expectedHash, _ := base64.RawStdEncoding.DecodeString(parts[5])

    computed := argon2.IDKey([]byte(password), salt, iterations, memory, parallelism, uint32(len(expectedHash)))

    return subtle.ConstantTimeCompare(computed, expectedHash) == 1, nil
}

```

## bcrypt

如果你的项目需要兼容已有 bcrypt 数据、或部署环境内存极度受限（< 64MB），bcrypt 仍然是可接受的选择：

```go
import "golang.org/x/crypto/bcrypt"

hash, _ := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost) // cost=10
ok := bcrypt.CompareHashAndPassword(hash, []byte(password)) == nil
```
