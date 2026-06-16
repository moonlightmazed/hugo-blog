---
title: MySQL
date: 2023-06-03T14:11:02+08:00
lastmod: 2023-06-03T14:11:02+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: image.png
images:
  - image.png
categories:
  - MySQL
tags:
  - MySQL
# nolastmod: true
draft: false
description: "MySQL 是最流行的关系型数据库管理系统，在 WEB 应用方面 MySQL 是最好的 RDBMS(Relational Database Management System：关系数据库管理系统)应用软件之一。"
---

## 索引
### 1. MySQL如何实现的索引机制？

​	MySQL中索引分三类：B+树索引，hash索引，全文索引

### 2. InnoDB索引与MyISAM索引实现的区别是什么？

​	InnoDB和MyISAM索引实现的核心区别在于**索引结构**、**数据存储方式**以及**事务支持**，具体差异如下：

​	存储不同：

​			  InnoDB索引在存储的时候是和数据文件放在一起的，`.ibd`文件存储索引和数据，数据按主键顺序组织，减少范围查询的磁盘I/O

​			  MyISAM索引和数据文件分开存储，`.MYD`（数据）和`.MYI`（索引）分离。数据按插入顺序存储，主键索引与普通索引无本质区别。

​	聚焦不同：

​			  InnoDB是聚集索引，主键索引的叶子节点直接存储数据行（即数据与主键索引绑定）

​			  MyISAM是非聚焦索引，所有索引（包括主键）的叶子节点仅存储数据行的物理地址（如`MYD`文件中的偏移量）。

### 3. 一个表中如果没有创建索引，还会创建B+树吗？

| **存储引擎** | **是否强制生成 B+ 树** |                           **说明**                           |
| :----------: | :--------------------: | :----------------------------------------------------------: |
|    InnoDB    |         **是**         | 必须通过聚集索引（显式或隐式主键）组织数据，数据文件本身就是 B+ 树。 |
|    MyISAM    |         **否**         | 数据按堆表存储，无索引时不生成 B+ 树；显式创建索引才会生成独立的 B+ 树。 |

### 4. B+树





