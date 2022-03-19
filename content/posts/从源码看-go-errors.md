---
title: "从源码看 Go errors"
date: 2022-03-09T18:03:08+08:00
draft: true
categories: ["tech"]
tags: ["tech", "go"]
---

_go version: 1.15.7_

这是一个很短小精悍的包，提供了操作 error 的接口，日常使用中经常会用到

# interface: error

需要先说的是 error，这是 errors 包的中心

error 是一个 interface，只包含一个 method: `Error() string`，返回该 error 的信息

# errors.go

errors.go 东西很少：一个 func 和一个 struct

## New()

`New` 很简单：返回一个 `errorString`

## errorString

```go
// errorString is a trivial implementation of error.
type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

可以看到 errorString 就只包含一个 string 用来保存 `Error()` 需要的信息

# wrap.go

wrap.go 顾名思义，用来处理 error wrap 的 func 都放在这里了

## 什么是 error wrap & 为什么要用它

go 中的异常处理对象都是基于 error interface 进行的，
而这个 interface 只有一个 `Error()`，
这就导致处理的粒度很粗：

1. 很多情况下你只能知道 `err != nil`，而不知道这个 err 具体是什么类型的错误
2. 如果你想知道这个 err 的具体类型，就需要解析 `err.Error()` 了
3. 而 `err.Error()` 是个字符串，手工解析字符串比较折磨、也很不优雅、还很容易坏

所以 error wrap 就来了

顾名思义，这个特性可以封装（wrap）一个 error，而后可以通过拆包（unwrap）把内部的 error 拆出来

## error wrap

见 errors.go#L10-L14:

```go
// An error wraps another error if its type has the method
//
//	Unwrap() error
//
// If e.Unwrap() returns a non-nil error w, then we say that e wraps w.
```
