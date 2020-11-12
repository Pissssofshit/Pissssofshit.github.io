---
title: go-实现分布式缓存
date: 2020-11-11 22:13:46
tags:
---

1. 单机缓存的实现
   1. 缓存淘汰策略
   2. 代码实现
2. 分布式节点
   1. 分布式节点分配算法
      1. 一致性哈希
         1. 原理
         2. 数据倾斜问题
         3. 代码实现
   2. 分布式节点通信
   3. 数据抽象
      1. 服务端的抽象
      2. 客户端的抽象

#### 缓存淘汰策略

##### 常见的几种淘汰策略

##### lru算法的go实现

##### go的链表操作

引入库

```
import "container/list"
```

这个库提供了链表的基本操作

```go
func New() *List  //初始化链表
func (l *List) insert(e, at *Element) *Element //插入元素
func (l *List) PushFront(v interface{}) *Element //队首插入
func (l *List) PushBack(v interface{}) *Element //队尾插入
```

不一一列举了，方法很全，看名字就知道是干什么的了

