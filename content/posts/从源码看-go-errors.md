---
title: "从源码看 Go errors"
date: 2022-03-09T18:03:08+08:00
draft: false
categories: ["tech"]
tags: ["tech", "go"]
---

_go version: 1.15.7_

# TL;DR for usage only

- error 是一个 interface，只包含一个 method: `Error() string`，返回该 error 的信息
  - 两种常用新建 error 的方法：
    1. errors.New()
    2. fmt.Errorf()
- wrap error：保留核心 error 信息，方便细粒度错误处理
  - wrap error 的方法：
    1. 自己实现一个包含 `Unwrap() error` 方法的结构
    2. 使用 `fmt.Errorf()` + `%w` 格式标记符
- unwrap error：解出核心 error，透过封装看本质，进行更精细的错误处理
  - errors 包提供的两个函数：
    1. `Is(err, target error) bool`：判断 err 能否匹配 target
    2. `As(err error, target interface{}) bool`：如果 err 匹配 target，将匹配结果放入 target
       - 注意：target 需要为指向 error 的指针

拓展：

[推荐包：`github.com/pkg/errors`](#拓展：github.com/pkg/errors)

接口语义自然、用法简洁，支持更多功能（如为 error 打印栈、try-catch style error handling）

---

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

所以 error wrap 就来了，error wrap 可以保留核心 error，方便错误处理

使用这个机制可以封装（wrap）一个 error，而后可以通过拆包（unwrap）把内部的 error 拆出来

## error wrap 使用 & 实现

### 怎么 wrap 一个 error

见 errors.go#L10-L14:

```go
// An error wraps another error if its type has the method
//
//	Unwrap() error
//
// If e.Unwrap() returns a non-nil error w, then we say that e wraps w.
```

所以我们只需要实现一个 struct，包含一个 `Unwrap() error` 方法就可以

这个 `Unwrap()` 方法在语义上应该遵循：返回的是内部被 wrap 的 error

官方库中的例子（来自 `fmt/errors.go#L32-L41`）：

```go
type wrapError struct {
	msg string
	err error
}

func (e *wrapError) Error() string {
	return e.msg
}

func (e *wrapError) Unwrap() error {
	return e.err
}
```

msg 用于实现 `error.Error()`，err 用以实现 `error.Unwrap()`，很简洁方便

但是每次都自己写一个 struct 也挺难过的，Go 标准库也提供了相关的支持，就是刚刚提到的 fmt 包：

errors.go#L19-L24

```go
// A simple way to create wrapped errors is to call fmt.Errorf and apply the %w verb
// to the error argument:
//
//	errors.Unwrap(fmt.Errorf("... %w ...", ..., err, ...))
//
// returns err.
```

调用 `fmt.Errorf()` 时将插值 err 的格式标记符（verb/format specifier）换成 `%w` 即可

_题外话：大家在还没用上%w 的时候一般用什么？我一般用%v/%+v_

### wrap 之后怎么用

wrap 是为了让我们能方便地获取内部的 error 类型以进行更细粒度的错误处理，那么怎么来处理呢？

#### Is()

例子：

```go
	f, err := myOpen()
	if err != nil {
		if errors.Is(err, os.ErrNotExist) {
			// an ErrNotExist, handle stuff here
		} else if errors.Is(err, os.ErrInvalid) {
			// an ErrInvalid, handle stuff here
		} else {
			fmt.Println("unable to cover error: ", err)
			return errors.Wrap(err, "myOpen")
		}
	}
```

`func Is(err, target error) bool` 用以判断 err 层层 unwrap（像洋葱一样）之后能否匹配到 target

`Is()` 也和很多库一样，优先支持自定义的 `Is()` 方法覆写行为：

如果 unwrap err 的过程中，遇到一个 err 自定义了 `Is()` 方法，那就优先用该自定义的方法

实现源码：

```go
func Is(err, target error) bool {
	if target == nil {
		return err == target
	}

	isComparable := reflectlite.TypeOf(target).Comparable()
	for {
		if isComparable && err == target {
			return true
		}
		if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
			return true
		}
		// TODO: consider supporting target.Is(err). This would allow
		// user-definable predicates, but also may allow for coping with sloppy
		// APIs, thereby making it easier to get away with them.
		if err = Unwrap(err); err == nil {
			return false
		}
	}
}
```

流程：

1. 2-4: 入参检查、快速返回
2. 6: 用反射检查 target 是不是 Comparable（能直接比较，能作为 ==、!= 的操作数）
3. 进入剥洋葱比较循环
   1. 1-3: 如果 target 能直接比较就直接比、匹配成功快速返回
   2. 4-6: target 不能直接比，看看当前洋葱有没有自定义 `Is()`，有的话就用，匹配成功快速返回
   3. 10-12: 再剥一层洋葱，要是剥没了就返回 false

这也指导我们平时固化错误时有两种推荐做法：

1. 做成 Comparable 的（不包含 map/slice/func）
   1. 比如常用的`var ErrKey = fmt.Error("errKey")`
2. 做一层封装，封装实现 `Is(error) bool`
   1. 要是想在 error 里带动态信息以方便处理的话推荐做这个

#### As()

`As()` 是 `Is()` 的拓展：不仅会匹配 err 与 target，而且匹配成功时会把匹配结果放到 target 里

这也对 target 提出了要求：

1. 非 nil
2. 是个指针
3. \*target 是 interface 或者实现了 error

例子（来自[官方 blog](https://go.dev/blog/go1.13-errors)）：

```go
// Similar to:
//   if e, ok := err.(*QueryError); ok { … }
var e *QueryError
// Note: *QueryError is the type of the error.
if errors.As(err, &e) {
    // err is a *QueryError, and e is set to the error's value
}
```

实现源码：

```go
func As(err error, target interface{}) bool {
	if target == nil {
		panic("errors: target cannot be nil")
	}
	val := reflectlite.ValueOf(target)
	typ := val.Type()
	if typ.Kind() != reflectlite.Ptr || val.IsNil() {
		panic("errors: target must be a non-nil pointer")
	}
	if e := typ.Elem(); e.Kind() != reflectlite.Interface && !e.Implements(errorType) {
		panic("errors: *target must be interface or implement error")
	}
	targetType := typ.Elem()
	for err != nil {
		if reflectlite.TypeOf(err).AssignableTo(targetType) {
			val.Elem().Set(reflectlite.ValueOf(err))
			return true
		}
		if x, ok := err.(interface{ As(interface{}) bool }); ok && x.As(target) {
			return true
		}
		err = Unwrap(err)
	}
	return false
}
```

流程：

1. 检查非 nil
2. 检查为指针且非 nil
3. 检查\*target 是 interface 或
4. 进入剥洋葱比较循环
   1. 检查当前 err 能不能放进 target，能就 set 后快速返回
   2. 检查当前 err 有没有自定义 `As()`，有的话就调用，set 后快速返回
   3. 剥一层洋葱

# 拓展：github.com/pkg/errors

这个包很好用，推荐一手，接口语义自然、用法简洁，我现在基本不用 `fmt.Errorf()` 了

## wrap error

```go
errors.Wrap(err, "foo")
// equals to fmt.Errorf("foo: %w", err)
```

且 err 为 nil 时 `errors.Wrap()` 也返回 nil，所以部分场景下代码能简洁很多：

原来的大众写法：

```go
if err := foo(); err != nil{
	return fmt.Errorf("foo: %w", err)
}
return nil
```

换用 errors 的等效写法：

```go
return errors.Wrap(foo(), "foo")
```

## 更自然的错误处理

这个包还提供了一个非常类似 try-catch 机制的接口，看例子：

```go
switch err := errors.Cause(err).(type) {
case *MyError1:
        // handle specifically
case *MyError2:
		// handle specifically
default:
        // unknown error
}
```

用 `Is()` 来做的话需要这么写：

```go
var myErr1 MyError1
var myErr2 MyError2
//...

if errors.As(err, &myErr1) {
	// handle specifically
}else if errors.As(err, &myErr2) {
	// handle specifically
}else{
	// unknown error
}
```

如果习惯其他语言中 try-catch work flow 的话应该会很亲切

不过 try-catch 也常被人诟病容易滥用，所以大家根据自己和项目的习惯来选择就好

## 打印栈

如果 err 递归返回的一路上都用的是 errors.Wrap，就可以打印从 err raise 到当前的栈

一是栈的排版方便看（从一把 log 里面抓一行超长的 error 的经历想必大家都体验过），
二是配合 IDE/编辑器内置的终端可以快速跳转，
还是很方便的

示例代码有点长：
https://pkg.go.dev/github.com/pkg/errors#hdr-Retrieving_the_stack_trace_of_an_error_or_wrapper
