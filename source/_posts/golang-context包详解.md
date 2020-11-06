---
title: golang context包详解
date: 2020-11-05 20:01:36
tags:
---

[快速掌握Context包](https://deepzz.com/post/golang-context-package-notes.html)

[官方Context博客](https://blog.golang.org/context)

[golang 如何正确使用context](https://juejin.im/post/6844903929340231694)



#### 为什么需要context

一个请求会有许多goroutine,此时就需要

* 一个信息传递者，用于传递这个请求携带的认证信息，deadline等等
* 一个通知者，通知各个goroutine释放资源

可以认为context是一棵以Background为根的树