---
title: "从源码看 Go Context"
date: 2021-09-13T15:37:54+08:00
draft: true
---

最近写业务发现该来学学 go 的 context 了，拿起 gopl 来看，发现书里没讲这个包，上网搜的相关教程讲的有点怪怪的，最后还是捞出源码来看了，于是就有了这篇。

# 学习列表

围绕 context，找出以下问题的答案：

1. 是什么东西
2. 用在什么地方（情景）
3. 怎么用（功能、接口）
4. 引入了哪些概念

拓展一些的问题：

1. 整体设计、性能设计的方案和亮点
2. 结合使用例子看

# 源码

https://github.com/golang/go/tree/master/src/context

# 文件注释

看标准库，那肯定得从注释看起。

虽然上来就看注释很可能看不懂概念，不过万事开头难嘛，没什么怕的，我们遇到新概念的时候记录一下方便后面留个心眼就行。官方的注释文档还是很有参考意义的，所以我们分段来慢慢看：

以下是源码中的文件注视的第一段，亦即摘要（L5 - L7）：

```go
// Package context defines the Context type, which carries deadlines,
// cancellation signals, and other request-scoped values across API boundaries
// and between processes.
```

从摘要中可以看出来 context 包主要干的活儿就是定义类型 Context，这个类型里带了一些参数

注意到摘要里是这样描述参数的：request-scoped values，这个词信息量还挺大的：

1. 这个 Context 类型是 request 相关的，所以 Context 的主要应用场景会是网络请求的 send、handle 上（结合 across API boundaries and between processes 这句也可以读出来）
2. 一个 request 应该对应一个 Context，这指导了 Context 的用法和使用时应该绑定 request，同时也说明了取名 Context 的合理性

后面就是讲用法和注意事项了（L9 - L23，有点长，就不 copy docstring 过来了，这里是做个小总结，推荐大家对照着注释读）：

1. 收到请求时要创建 Context，调用外部函数了要传 Context 过去
2. 要维持 Context 链，这里提到了几个操作：`WithCancel`、`WithDeadline`、`WithTimeout`、`WithValue`，这几个操作可以用来制造子孙 Context，待会儿要重点看看
3. Context 的 Cancel 处理
   1. 自己的子孙要和自己一起 canceled
   2. `WithCancel`、`WithDeadline`、`WithTimeout`都会生成一个`CancelFunc`用来 cancel context，确保每一条控制流路径上都调用`CancelFunc`（go vet 可以检查的 好东西）

接下来是几条「守则」（L25 - L44），一个 package 提出了约束各种包和接口的守则，这点想想很有意思，文档提出这些守则是为了：

```go
// Programs that use Contexts should follow these rules to keep interfaces
// consistent across packages and enable static analysis tools to check context
// propagation:
```

态度还挺强硬的，我们来看看都有些啥：

1. 不要把 Context 放在结构体里传
   1. Context 应该作为参数传递
   2. Context 还应该作为函数的第一个参数
   3. 推荐把这个参数命名为 ctx
2. 不要用 nil Context
   1. 有的函数能接受 nil Context，但是还是不要用
   2. 如果不知道用什么 Context 的话就用`context.TODO()`
   3. 这儿出现了`.TODO()`，这个名字有点怪怪的，之后我们也会重点讲讲它，`TODO()` 是个很灵性的设计
3. context 中有个概念叫 Value，不要把 Value 当作传参工具，Value 应该用来传跨 API 和进程的数据
4. Context 是并发安全的

# 是什么东西

协调 goroutines 之间同步和数据的、每个请求独立的上下文功能组件

# 用在什么地方（情景）

主要用在网络请求的处理中

go 在他们的 [blog 中](https://go.dev/blog/context)就提到了：用 go 写的网络服务器一般会用一堆 goroutines 来处理同一个请求，比如接请求、打解包、调用数据端等等等等都是 goroutine，那这些服务于同一个请求的 goroutine 之间怎么互传数据、怎么做超时控制等等功能呢？

context 就被设计出来救世了

context 是 google 内部设计出来协调这些「服务于同一个请求」的整套 goroutine 的功能组件，所以 context 也被设计为「request-specific」的，亦即「一个请求、一套 Context」理念

# 怎么用（功能、接口）

context 包中主要提供了接口 Context，这个接口有四个方法：

1. `Deadline() (deadline time.Time, ok bool)`：返回该 Context 的 deadline（新概念），这是个 getter 型方法
2. `Done() <-chan struct{}`：返回的 channel 用来通知业务这个 context 被 cancel（新概念）了
3. `Err() error`：如果`Done()`已经被关闭了，那`Err()`就会返回一个 error，解释为什么`Done()`被关闭；反之则返回 nil
4. `Value(key interface{}) interface{}`：从该 Context 所拥有的 Value（新概念）中取值，Value 是一个类似 map 的结构，所以需要 key

其实这几个新概念捋完，context 包基本就解清了，context 本身也只是一个小包，源码（不含测试）加上注释也才不到六百行

顺带一提，源码：
https://github.com/golang/go/tree/master/src/context ，推荐一看

# 引入了哪些概念

## cancel

cancel 是 context 的功能中「控制」部分的实现方式，context 的使用是呈树状的，cancel 一个 context 会关闭它本身和所有的子 context

通过 cancel 这个手段，我们能够控制 context 的使用、通知协调同步众多子 goroutine

## deadline

deadline 就是一个 Context 的时限，这个 context 会在到达 deadline 时自动被 cancel

这个概念用于 context 的超时控制

### 用法

为一个 Context 加 deadline 的方法：

```go
newCtx, cancel = context.WithDeadline(ctx, deadline)
```

`newCtx`的 deadline 就被设定为了`deadline`，cancel 是一个函数，调用 cancel 可以手动 cancel newCtx

### 看看源码

```go
// WithDeadline returns a copy of the parent context with the deadline adjusted
// to be no later than d. If the parent's deadline is already earlier than d,
// WithDeadline(parent, d) is semantically equivalent to parent. The returned
// context's Done channel is closed when the deadline expires, when the returned
// cancel function is called, or when the parent context's Done channel is
// closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

整体还是比较好理解的，`propagateCancel(parent, c)`让 c 跟着 parent 被 cancel（context 的使用是树状的，所以 parent 被 cancel 时所有子孙 context 都会被 cancel）
