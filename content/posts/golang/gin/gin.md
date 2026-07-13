---
title: Gin
date: 2021-08-21T14:21:02+08:00
lastmod: 2021-08-21T14:21:02+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /img/covers/草地和鹰.png
images:
  - /img/covers/草地和鹰.png
categories:
  - Golang
tags:
  - Gin
# nolastmod: true
draft: false
description: "Gin 是 Go 语言中最流行的 Web 框架之一，以高性能和简洁的 API 著称。本文详细介绍 Gin 的核心功能和最佳实践。"
---

## Gin的HelloWorld体验

### 安装Gin

```bash
go get -u github.com/gin-gonic/gin
```

### 基本示例

```go
package main

import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()

    r.GET("/", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "Hello, Gin!",
        })
    })

    r.Run(":8080")
}
```

### 运行方式

```bash
go run main.go
```

访问 `http://localhost:8080` 即可看到响应。

### 项目结构

```
gin-app/
├── go.mod
├── go.sum
└── main.go
```

## 使用New和Default初始化路由器的区别

### gin.New()

```go
r := gin.New()
```

创建一个不带任何默认中间件的路由器。

### gin.Default()

```go
r := gin.Default()
```

创建一个带有默认中间件的路由器，包括：

- **Logger**：日志中间件，记录请求信息
- **Recovery**：恢复中间件，捕获 panic 并返回 500 错误

### 对比

| 方法          | Logger | Recovery | 适用场景         |
| ------------- | ------ | -------- | ---------------- |
| gin.New()     | 否     | 否       | 需要自定义中间件 |
| gin.Default() | 是     | 是       | 快速开发、调试   |

### 手动添加中间件

```go
r := gin.New()
r.Use(gin.Logger())
r.Use(gin.Recovery())
```

## Gin的路由分组

### 基本路由分组

```go
r := gin.Default()

api := r.Group("/api")
{
    api.GET("/users", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "users"})
    })
    api.GET("/posts", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "posts"})
    })
}
```

### 嵌套路由分组

```go
v1 := api.Group("/v1")
{
    v1.GET("/users", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "v1 users"})
    })
}
```

### 分组中间件

```go
auth := r.Group("/auth")
auth.Use(AuthMiddleware())
{
    auth.POST("/login", LoginHandler)
    auth.POST("/register", RegisterHandler)
}
```

### 路由组的优势

- **代码组织**：按功能模块组织路由
- **复用中间件**：为整个分组添加中间件
- **版本管理**：便于 API 版本控制

## 获取URL中的变量

### 基本参数

```go
r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.JSON(200, gin.H{"user_id": id})
})
```

### 多个参数

```go
r.GET("/users/:id/posts/:post_id", func(c *gin.Context) {
    id := c.Param("id")
    postID := c.Param("post_id")
    c.JSON(200, gin.H{
        "user_id":  id,
        "post_id": postID,
    })
})
```

### 可选参数

```go
r.GET("/users/:id/*action", func(c *gin.Context) {
    id := c.Param("id")
    action := c.Param("action")
    c.JSON(200, gin.H{
        "user_id": id,
        "action":  action,
    })
})
```

### 参数验证

```go
r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    userID, err := strconv.Atoi(id)
    if err != nil {
        c.JSON(400, gin.H{"error": "invalid user id"})
        return
    }
    c.JSON(200, gin.H{"user_id": userID})
})
```

## 获取GET和POST表单信息

### GET参数

```go
r.GET("/search", func(c *gin.Context) {
    keyword := c.Query("keyword")
    page := c.DefaultQuery("page", "1")
    c.JSON(200, gin.H{
        "keyword": keyword,
        "page":    page,
    })
})
```

### POST表单

```go
r.POST("/login", func(c *gin.Context) {
    username := c.PostForm("username")
    password := c.PostForm("password")
    c.JSON(200, gin.H{
        "username": username,
        "password": password,
    })
})
```

### 默认值

```go
username := c.DefaultPostForm("username", "guest")
```

### 获取所有参数

```go
r.POST("/submit", func(c *gin.Context) {
    form := make(map[string]string)
    c.Bind(&form)
    c.JSON(200, form)
})
```

### JSON请求体

```go
type LoginRequest struct {
    Username string `json:"username"`
    Password string `json:"password"`
}

r.POST("/login", func(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    c.JSON(200, gin.H{
        "username": req.Username,
        "password": req.Password,
    })
})
```

## Gin返回Protobuf

### 安装依赖

```bash
go get github.com/golang/protobuf/proto
go get google.golang.org/protobuf
```

### 定义Proto文件

```protobuf
syntax = "proto3";

package hello;

option go_package = "./pb";

message HelloResponse {
    string message = 1;
    int32 code = 2;
}
```

### 生成Go代码

```bash
protoc --go_out=. hello.proto
```

### 返回Protobuf

```go
import (
    "github.com/gin-gonic/gin"
    "github.com/golang/protobuf/proto"
    "yourproject/pb"
)

r.GET("/hello", func(c *gin.Context) {
    resp := &pb.HelloResponse{
        Message: "Hello, Protobuf!",
        Code:    200,
    }

    c.ProtoBuf(200, resp)
})
```

### 设置Content-Type

```go
c.Header("Content-Type", "application/x-protobuf")
c.ProtoBuf(200, resp)
```

### 客户端调用

```go
resp, err := http.Get("http://localhost:8080/hello")
if err != nil {
    panic(err)
}
defer resp.Body.Close()

data, _ := io.ReadAll(resp.Body)
var hello pb.HelloResponse
proto.Unmarshal(data, &hello)
fmt.Println(hello.Message)
```

## 登录的表单验证

### 使用Binding

```go
type LoginRequest struct {
    Username string `form:"username" binding:"required"`
    Password string `form:"password" binding:"required,min=6"`
}

r.POST("/login", func(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBind(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    c.JSON(200, gin.H{"message": "login success"})
})
```

### 验证规则

| 规则     | 说明       |
| -------- | ---------- |
| required | 必填       |
| min      | 最小长度   |
| max      | 最大长度   |
| len      | 固定长度   |
| email    | 邮箱格式   |
| url      | URL格式    |
| regexp   | 正则表达式 |

### 自定义验证

```go
func ValidatePassword(fl validator.FieldLevel) bool {
    password := fl.Field().String()
    return len(password) >= 6 && len(password) <= 20
}

r := gin.Default()
if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
    v.RegisterValidation("password", ValidatePassword)
}

type LoginRequest struct {
    Password string `form:"password" binding:"required,password"`
}
```

## 注册表单的验证

### 复杂验证示例

```go
type RegisterRequest struct {
    Username string `form:"username" binding:"required,min=3,max=20"`
    Email    string `form:"email" binding:"required,email"`
    Password string `form:"password" binding:"required,min=6,max=20"`
    Age      int    `form:"age" binding:"gte=18,lte=100"`
}

r.POST("/register", func(c *gin.Context) {
    var req RegisterRequest
    if err := c.ShouldBind(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    c.JSON(200, gin.H{"message": "register success"})
})
```

### 跨字段验证

```go
func ValidatePasswordConfirm(fl validator.FieldLevel) bool {
    password := fl.Field().String()
    confirmPassword := fl.Parent().FieldByName("PasswordConfirm").String()
    return password == confirmPassword
}

type RegisterRequest struct {
    Password        string `form:"password" binding:"required,min=6"`
    PasswordConfirm string `form:"password_confirm" binding:"required,eqfield=Password"`
}
```

### 验证错误处理

```go
if err := c.ShouldBind(&req); err != nil {
    var errs validator.ValidationErrors
    if errors.As(err, &errs) {
        result := make(map[string]string)
        for _, e := range errs {
            field := e.Field()
            tag := e.Tag()
            result[field] = fmt.Sprintf("%s validation failed: %s", field, tag)
        }
        c.JSON(400, gin.H{"errors": result})
    }
    return
}
```

## 表单验证错误翻译成中文

### 安装翻译包

```bash
go get github.com/go-playground/locales
go get github.com/go-playground/universal-translator
```

### 配置翻译

```go
import (
    "github.com/go-playground/locales/zh"
    "github.com/go-playground/universal-translator"
    ut "github.com/go-playground/universal-translator"
)

func main() {
    r := gin.Default()

    zh := zh.New()
    ut := ut.New(zh, zh)
    translator, _ := ut.GetTranslator("zh")

    if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
        _ = zh.RegisterDefaultTranslations(v, translator)

        // 自定义翻译
        v.RegisterTranslation("required", translator, func(ut ut.Translator) error {
            return ut.Add("required", "{0} 必填", true)
        }, func(ut ut.Translator, fe validator.FieldError) string {
            t, _ := ut.T("required", fe.Field())
            return t
        })
    }
}
```

### 使用翻译

```go
func TranslateErrors(err error, translator ut.Translator) map[string]string {
    var errs validator.ValidationErrors
    if errors.As(err, &errs) {
        result := make(map[string]string)
        for _, e := range errs {
            result[e.Field()] = e.Translate(translator)
        }
        return result
    }
    return nil
}

r.POST("/login", func(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBind(&req); err != nil {
        errors := TranslateErrors(err, translator)
        c.JSON(400, gin.H{"errors": errors})
        return
    }
})
```

## 表单中文翻译的JSON格式化细节

### 自定义格式化函数

```go
func FormatValidationErrors(errs validator.ValidationErrors, translator ut.Translator) []map[string]interface{} {
    result := make([]map[string]interface{}, 0)
    for _, e := range errs {
        result = append(result, map[string]interface{}{
            "field":   e.Field(),
            "message": e.Translate(translator),
            "tag":     e.Tag(),
            "value":   e.Value(),
        })
    }
    return result
}
```

### 返回格式化JSON

```go
r.POST("/register", func(c *gin.Context) {
    var req RegisterRequest
    if err := c.ShouldBind(&req); err != nil {
        var errs validator.ValidationErrors
        if errors.As(err, &errs) {
            formatted := FormatValidationErrors(errs, translator)
            c.JSON(400, gin.H{
                "code":    400,
                "message": "validation failed",
                "errors":  formatted,
            })
            return
        }
    }
    c.JSON(200, gin.H{"message": "success"})
})
```

### 响应示例

```json
{
  "code": 400,
  "message": "validation failed",
  "errors": [
    {
      "field": "Username",
      "message": "Username 必填",
      "tag": "required",
      "value": ""
    },
    {
      "field": "Password",
      "message": "Password 长度必须至少为 6 个字符",
      "tag": "min",
      "value": "123"
    }
  ]
}
```

## 自定义Gin中间件

### 基本中间件

```go
func LoggerMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        startTime := time.Now()

        c.Next()

        duration := time.Since(startTime)
        log.Printf("[%s] %s %s %v",
            c.Request.Method,
            c.Request.URL.Path,
            c.Request.Proto,
            duration,
        )
    }
}

r.Use(LoggerMiddleware())
```

### 带参数的中间件

```go
func AuthMiddleware(requiredRole string) gin.HandlerFunc {
    return func(c *gin.Context) {
        role := c.GetHeader("X-Role")
        if role != requiredRole {
            c.JSON(403, gin.H{"error": "forbidden"})
            c.Abort()
            return
        }
        c.Next()
    }
}

r.GET("/admin", AuthMiddleware("admin"), func(c *gin.Context) {
    c.JSON(200, gin.H{"message": "admin page"})
})
```

### 全局中间件

```go
r := gin.Default()
r.Use(LoggerMiddleware())
r.Use(RecoveryMiddleware())
```

### 路由组中间件

```go
api := r.Group("/api")
api.Use(AuthMiddleware())
{
    api.GET("/users", GetUsers)
}
```

### 单个路由中间件

```go
r.GET("/profile", AuthMiddleware(), func(c *gin.Context) {
    c.JSON(200, gin.H{"message": "profile"})
})
```

## 通过Abort终止中间件后续逻辑的执行

### 基本用法

```go
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.JSON(401, gin.H{"error": "unauthorized"})
            c.Abort()
            return
        }
        c.Next()
    }
}
```

### AbortWithStatus

```go
c.AbortWithStatus(401)
```

### AbortWithStatusJSON

```go
c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
```

### AbortWithError

```go
c.AbortWithError(401, errors.New("unauthorized"))
```

### 中间件执行流程

```
中间件1 → 中间件2 → 中间件3 → 处理器 → 中间件3 → 中间件2 → 中间件1
            ↑
         Abort()
         终止执行，直接返回
```

### 检查是否已终止

```go
func LoggerMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        if c.IsAborted() {
            return
        }
        // 日志逻辑
        c.Next()
    }
}
```

## Gin的中间件原理源码分析

### HandlerChain

```go
type HandlerChain []HandlerFunc
```

### Context结构

```go
type Context struct {
    handlers HandlersChain
    index    int8
    // ...
}
```

### Next方法

```go
func (c *Context) Next() {
    c.index++
    for c.index < int8(len(c.handlers)) {
        c.handlers[c.index](c)
        c.index++
    }
}
```

### Abort方法

```go
func (c *Context) Abort() {
    c.index = abortIndex
}
```

### Use方法

```go
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
    engine.RouterGroup.Use(middleware...)
    return engine
}

func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
    group.Handlers = append(group.Handlers, middleware...)
    return group.returnObj()
}
```

### 路由匹配

```go
func (engine *Engine) Handle(httpMethod, relativePath string, handlers ...HandlerFunc) IRoutes {
    absolutePath := group.calculateAbsolutePath(relativePath)
    handlers = group.combineHandlers(handlers)
    engine.addRoute(httpMethod, absolutePath, handlers)
    return group.returnObj()
}
```

### 中间件执行顺序

1. 全局中间件（Use添加）
2. 路由组中间件（Group.Use添加）
3. 路由中间件（Handle时传入）

## Gin返回HTML

### 基本用法

```go
r.LoadHTMLGlob("templates/*")

r.GET("/", func(c *gin.Context) {
    c.HTML(200, "index.html", gin.H{
        "title": "Hello, Gin!",
        "name":  "Alice",
    })
})
```

### 模板文件

```html
<!-- templates/index.html -->
<!DOCTYPE html>
<html>
  <head>
    <title>{{ .title }}</title>
  </head>
  <body>
    <h1>Hello, {{ .name }}!</h1>
  </body>
</html>
```

### 模板函数

```go
r.SetFuncMap(template.FuncMap{
    "upper": strings.ToUpper,
})

r.LoadHTMLGlob("templates/*")
```

### 使用模板函数

```html
<h1>{{ upper .name }}</h1>
```

### 自定义分隔符

```go
r.Delims("{{{", "}}}")
```

## 加载多个HTML文件

### 加载多个目录

```go
r.LoadHTMLGlob("templates/**/*")
```

### 模板结构

```
templates/
├── index.html
├── about.html
└── admin/
    ├── dashboard.html
    └── users.html
```

### 引用模板

```go
r.GET("/admin/dashboard", func(c *gin.Context) {
    c.HTML(200, "admin/dashboard.html", gin.H{})
})
```

### 模板继承

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html>
  <head>
    <title>{{ .title }}</title>
  </head>
  <body>
    {{ block "content" . }}{{ end }}
  </body>
</html>
```

```html
<!-- templates/index.html -->
{{ template "base.html" . }} {{ define "content" }}
<h1>Hello, {{ .name }}!</h1>
{{ end }}
```

### 包含模板

```html
{{ template "header.html" }}
<div>Content</div>
{{ template "footer.html" }}
```

## Static静态文件的处理

### 基本用法

```go
r.Static("/static", "./static")
```

### 访问静态文件

```html
<link rel="stylesheet" href="/static/css/style.css" />
<img src="/static/images/logo.png" />
```

### 静态文件结构

```
static/
├── css/
│   └── style.css
├── js/
│   └── app.js
└── images/
    └── logo.png
```

### 单个文件

```go
r.StaticFile("/favicon.ico", "./static/favicon.ico")
```

### 静态文件索引

```go
r.StaticFS("/static", http.Dir("./static"))
```

### 生产环境优化

- 使用 CDN 托管静态文件
- 启用 Gzip 压缩
- 设置缓存头

## Gin的优雅退出

### 基本实现

```go
func main() {
    r := gin.Default()

    r.GET("/", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "hello"})
    })

    srv := &http.Server{
        Addr:    ":8080",
        Handler: r,
    }

    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %s\n", err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt)
    <-quit

    log.Println("Shutdown Server ...")

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server Shutdown:", err)
    }

    log.Println("Server exiting")
}
```

### 处理信号

```go
signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
```

### 超时控制

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
```

### 优雅退出流程

1. 接收到退出信号（SIGINT/SIGTERM）
2. 停止接收新请求
3. 等待当前请求处理完成
4. 关闭数据库连接等资源
5. 退出进程

### 完整示例

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()

    r.GET("/", func(c *gin.Context) {
        time.Sleep(5 * time.Second)
        c.JSON(200, gin.H{"message": "hello"})
    })

    srv := &http.Server{
        Addr:    ":8080",
        Handler: r,
    }

    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %s\n", err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down server...")

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }

    log.Println("Server exiting")
}
```
