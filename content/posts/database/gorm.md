---
title: Gorm
date: 2021-06-11T14:21:02+08:00
lastmod: 2021-06-11T14:21:02+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /img/covers/大桥.png
images:
  - /img/covers/大桥.png
categories:
  - Database
tags:
  - ORM
  - MySQL
# nolastmod: true
draft: false
description: "GORM 是 Go 语言中最流行的 ORM 框架，提供了强大的数据库操作能力，支持多种数据库，简化了数据库操作的复杂度。"
---

## 什么是ORM

### ORM概念

ORM（Object-Relational Mapping）即对象关系映射，是一种将面向对象编程语言中的对象与关系型数据库中的表进行映射的技术。

**核心思想**：通过操作对象来操作数据库，而不需要编写原生SQL语句。

### ORM的优点

- **简化开发**：无需编写复杂的SQL语句
- **类型安全**：编译时检查类型错误
- **数据库无关**：切换数据库只需修改配置
- **提高可维护性**：代码更易理解和维护

### ORM的缺点

- **性能开销**：相比原生SQL有一定性能损失
- **学习成本**：需要学习框架的API和规则
- **灵活性受限**：复杂查询可能难以实现

### 如何正确看待ORM

- **适合场景**：常规CRUD操作、快速开发、团队协作
- **不适合场景**：高性能要求、复杂查询优化、数据库特定功能

## GORM连接数据库

### 安装GORM

```bash
go get gorm.io/gorm
go get gorm.io/driver/mysql
```

### 基本连接

```go
package main

import (
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

func main() {
    dsn := "user:password@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        panic("failed to connect database")
    }
}
```

### DSN格式说明

```
user:password@tcp(host:port)/dbname?params
```

| 参数      | 说明                |
| --------- | ------------------- |
| user      | 数据库用户名        |
| password  | 数据库密码          |
| host      | 数据库地址          |
| port      | 数据库端口          |
| dbname    | 数据库名称          |
| charset   | 字符集，推荐utf8mb4 |
| parseTime | 是否解析时间        |
| loc       | 时区                |

### 连接池配置

```go
sqlDB, err := db.DB()
if err != nil {
    panic("failed to get DB")
}

sqlDB.SetMaxIdleConns(10)
sqlDB.SetMaxOpenConns(100)
sqlDB.SetConnMaxLifetime(time.Hour)
```

## Auto Migrate功能

### 什么是Auto Migrate

Auto Migrate 可以自动根据 Model 定义创建或更新数据库表结构。

### 使用示例

```go
type User struct {
    ID   uint
    Name string
    Age  int
}

func main() {
    db.AutoMigrate(&User{})
}
```

### 字段类型映射

| Go类型           | MySQL类型       |
| ---------------- | --------------- |
| int, int64       | bigint          |
| uint, uint64     | bigint unsigned |
| float32, float64 | float           |
| string           | longtext        |
| bool             | tinyint(1)      |
| time.Time        | datetime        |

### 注意事项

- **创建表**：如果表不存在，自动创建
- **更新表**：只添加新字段，不会删除已有字段
- **不会修改**：字段类型、字段名、约束条件

## GORM的Model逻辑删除

### 软删除概念

软删除不是真正删除记录，而是通过标记字段表示记录已删除。

### 启用软删除

```go
type User struct {
    gorm.Model
    Name string
}
```

`gorm.Model` 包含以下字段：

```go
type Model struct {
    ID        uint           `gorm:"primaryKey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

### 查询时自动过滤

```go
var users []User
db.Find(&users)
```

上述查询会自动添加 `WHERE deleted_at IS NULL` 条件。

### 查找已删除的记录

```go
var users []User
db.Unscoped().Find(&users)
```

### 永久删除

```go
db.Unscoped().Delete(&user)
```

## 通过NullString解决不能更新零值问题

### 问题描述

在 GORM 中，零值（如空字符串、0、false）不会被更新到数据库。

```go
user.Name = ""
db.Save(&user)
```

上述代码中，`Name` 字段不会被更新为空字符串。

### 解决方案

使用 `sql.NullString` 或 GORM 的 `gorm.NullString`：

```go
import (
    "gorm.io/gorm"
    "database/sql"
)

type User struct {
    gorm.Model
    Name sql.NullString
    Age  sql.NullInt64
}

user := User{
    Name: sql.NullString{String: "", Valid: true},
}
db.Save(&user)
```

### 使用 GORM 的 Scanner

```go
type User struct {
    gorm.Model
    Name *string
    Age  *int
}

name := ""
user := User{
    Name: &name,
}
db.Save(&user)
```

## 表结构定义细节

### 字段标签

```go
type User struct {
    ID        uint      `gorm:"primaryKey"`
    Name      string    `gorm:"size:255;not null;unique"`
    Age       int       `gorm:"default:0"`
    Email     string    `gorm:"column:email_address"`
    CreatedAt time.Time `gorm:"autoCreateTime"`
    UpdatedAt time.Time `gorm:"autoUpdateTime"`
}
```

### 常用标签

| 标签           | 说明         |
| -------------- | ------------ |
| primaryKey     | 主键         |
| size           | 字段大小     |
| not null       | 非空约束     |
| unique         | 唯一约束     |
| default        | 默认值       |
| column         | 列名         |
| autoCreateTime | 自动创建时间 |
| autoUpdateTime | 自动更新时间 |
| index          | 创建索引     |
| uniqueIndex    | 创建唯一索引 |

### 嵌入结构体

```go
type Address struct {
    City  string
    State string
}

type User struct {
    gorm.Model
    Name    string
    Address Address `gorm:"embedded"`
}
```

## 通过create方法插入记录

### 基本插入

```go
user := User{Name: "Alice", Age: 25}
result := db.Create(&user)

// 返回插入记录的主键
fmt.Println(user.ID)

// 返回错误
fmt.Println(result.Error)

// 返回插入的记录数
fmt.Println(result.RowsAffected)
```

### 批量插入

```go
users := []User{
    {Name: "Alice", Age: 25},
    {Name: "Bob", Age: 30},
}
db.Create(&users)
```

### 通过Map插入

```go
db.Create(map[string]interface{}{
    "Name": "Charlie",
    "Age":  35,
})
```

### 选择字段插入

```go
db.Select("Name").Create(&User{Name: "Dave", Age: 40})
```

### 忽略字段

```go
db.Omit("Age").Create(&User{Name: "Eve", Age: 45})
```

## 批量插入和通过map插入记录

### 批量插入优化

```go
users := make([]User, 100)
for i := range users {
    users[i] = User{Name: fmt.Sprintf("User%d", i)}
}

db.CreateInBatches(users, 100)
```

### 使用Upsert

```go
db.Clauses(clause.OnConflict{
    Columns:   []clause.Column{{Name: "name"}},
    DoUpdates: clause.AssignmentColumns([]string{"age"}),
}).Create(&users)
```

### 通过Map批量插入

```go
db.Model(&User{}).Create([]map[string]interface{}{
    {"Name": "Alice", "Age": 25},
    {"Name": "Bob", "Age": 30},
})
```

## 通过take, first、last获取数据

### 获取第一条记录

```go
var user User
db.First(&user)
```

### 获取最后一条记录

```go
var user User
db.Last(&user)
```

### 获取任意一条记录

```go
var user User
db.Take(&user)
```

### 根据条件获取

```go
var user User
db.Where("name = ?", "Alice").First(&user)
```

### 处理不存在的情况

```go
var user User
result := db.First(&user)

if errors.Is(result.Error, gorm.ErrRecordNotFound) {
    fmt.Println("记录不存在")
}
```

## GORM的基本查询

### 查询所有记录

```go
var users []User
db.Find(&users)
```

### 带条件查询

```go
var users []User
db.Where("age > ?", 18).Find(&users)
```

### 多个条件

```go
db.Where("name = ? AND age > ?", "Alice", 18).Find(&users)
```

### IN查询

```go
db.Where("name IN ?", []string{"Alice", "Bob"}).Find(&users)
```

### LIKE查询

```go
db.Where("name LIKE ?", "%li%").Find(&users)
```

### 排序

```go
db.Order("age DESC").Find(&users)
```

### 分页

```go
var users []User
db.Limit(10).Offset(0).Find(&users)
```

### 计数

```go
var count int64
db.Model(&User{}).Count(&count)
```

## GORM的更新操作

### 保存所有字段

```go
user := User{ID: 1, Name: "Alice", Age: 26}
db.Save(&user)
```

### 更新指定字段

```go
db.Model(&user).Update("name", "Alice Updated")
```

### 更新多个字段

```go
db.Model(&user).Updates(map[string]interface{}{
    "name": "Alice Updated",
    "age":  26,
})
```

### 使用结构体更新

```go
db.Model(&user).Updates(User{Name: "Alice Updated", Age: 26})
```

### 更新选中字段

```go
db.Model(&user).Select("name").Updates(User{Name: "Alice", Age: 27})
```

### 更新忽略字段

```go
db.Model(&user).Omit("age").Updates(User{Name: "Alice", Age: 27})
```

### 批量更新

```go
db.Model(&User{}).Where("age > ?", 18).Update("age", 19)
```

## GORM的软删除细节

### 查询未删除记录

```go
var users []User
db.Find(&users)
```

### 查询已删除记录

```go
var users []User
db.Unscoped().Find(&users)
```

### 恢复已删除记录

```go
db.Unscoped().Model(&user).Update("deleted_at", nil)
```

### 永久删除

```go
db.Unscoped().Delete(&user)
```

### 软删除条件

```go
var users []User
db.Where("deleted_at IS NOT NULL").Find(&users)
```

## 表的关联插入

### 一对一关系

```go
type User struct {
    gorm.Model
    Name    string
    Profile Profile
}

type Profile struct {
    gorm.Model
    UserID uint
    Bio    string
}

user := User{
    Name: "Alice",
    Profile: Profile{
        Bio: "Software Engineer",
    },
}

db.Create(&user)
```

### 保存关联

```go
user := User{Name: "Alice"}
db.Create(&user)

user.Profile = Profile{Bio: "Engineer"}
db.Model(&user).Association("Profile").Append(&user.Profile)
```

## 通过preload和joins查询多表

### Preload预加载

```go
var users []User
db.Preload("Profile").Find(&users)
```

### 多级预加载

```go
db.Preload("Orders").Preload("Orders.Items").Find(&users)
```

### 带条件的预加载

```go
db.Preload("Orders", "status = ?", "paid").Find(&users)
```

### Joins连接查询

```go
db.Joins("JOIN profiles ON profiles.user_id = users.id").
    Where("profiles.bio LIKE ?", "%Engineer%").
    Find(&users)
```

## Has Many关系

### 定义关系

```go
type User struct {
    gorm.Model
    Name   string
    Orders []Order
}

type Order struct {
    gorm.Model
    UserID uint
    Amount float64
}
```

### 查询关联

```go
var user User
db.Preload("Orders").First(&user)
```

### 添加关联

```go
order := Order{Amount: 100}
db.Model(&user).Association("Orders").Append(&order)
```

### 删除关联

```go
db.Model(&user).Association("Orders").Delete(&order)
```

### 替换关联

```go
orders := []Order{{Amount: 200}, {Amount: 300}}
db.Model(&user).Association("Orders").Replace(&orders)
```

### 清空关联

```go
db.Model(&user).Association("Orders").Clear()
```

## GORM处理多对多的关系

### 定义关系

```go
type User struct {
    gorm.Model
    Name  string
    Roles []Role `gorm:"many2many:user_roles;"`
}

type Role struct {
    gorm.Model
    Name  string
    Users []User `gorm:"many2many:user_roles;"`
}
```

### 创建关联表

```go
db.AutoMigrate(&User{}, &Role{})
```

### 添加关联

```go
user := User{Name: "Alice"}
role := Role{Name: "Admin"}

db.Create(&user)
db.Create(&role)

db.Model(&user).Association("Roles").Append(&role)
```

### 查询关联

```go
var user User
db.Preload("Roles").First(&user)
```

### 删除关联

```go
db.Model(&user).Association("Roles").Delete(&role)
```

### 替换关联

```go
roles := []Role{{Name: "User"}, {Name: "Editor"}}
db.Model(&user).Association("Roles").Replace(&roles)
```

### 清空关联

```go
db.Model(&user).Association("Roles").Clear()
```

### 获取关联计数

```go
var count int64
db.Model(&user).Association("Roles").Count(&count)
```

## GORM的表名自定义、自定义beforecreate逻辑

### 表名自定义

```go
func (u User) TableName() string {
    return "my_users"
}
```

### 全局禁用复数

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    NamingStrategy: schema.NamingStrategy{
        SingularTable: true,
    },
})
```

### BeforeCreate钩子

```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
    u.UUID = uuid.New().String()
    return
}
```

### AfterCreate钩子

```go
func (u *User) AfterCreate(tx *gorm.DB) (err error) {
    tx.Create(&Profile{UserID: u.ID})
    return
}
```

### 其他钩子

```go
func (u *User) BeforeUpdate(tx *gorm.DB) (err error) {
    // 更新前逻辑
    return
}

func (u *User) AfterUpdate(tx *gorm.DB) (err error) {
    // 更新后逻辑
    return
}

func (u *User) BeforeDelete(tx *gorm.DB) (err error) {
    // 删除前逻辑
    return
}

func (u *User) AfterDelete(tx *gorm.DB) (err error) {
    // 删除后逻辑
    return
}
```

### 事务中的钩子

钩子会在事务中执行，可以访问当前事务对象 `tx`。
