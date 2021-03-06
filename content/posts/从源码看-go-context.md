---
title: "从源码看 Go Context"
date: 2021-09-13T15:37:54+08:00
draft: false
tags: ["tech", "go"]
---

最近写业务发现该来学学 go 的 context 了，拿起 gopl 来看，发现书里没讲这个包，上网搜的相关教程讲的有点怪怪的，最后还是捞出源码来看了，于是就有了这篇。

# 源码

https://github.com/golang/go/tree/master/src/context

# 文件注释

看标准库，那肯定得从注释看起。

虽然上来就看注释很可能看不懂概念，不过万事开头难嘛，没什么怕的，我们遇到新概念的时候记录一下方便后面留个心眼就行。官方的注释文档还是很有参考意义的，所以我们分段来慢慢看：

以下是源码中的文件注释的第一段，亦即摘要（L5 - L7）：

```go
// Package context defines the Context type, which carries deadlines,
// cancellation signals, and other request-scoped values across API boundaries
// and between processes.
```

从摘要中可以看出来 context 包主要干的活儿就是定义类型 Context，这个类型里带了一些参数

注意到摘要里是这样描述参数的：request-scoped values，这个词信息量还挺大的：

1. 这个 Context 类型是 request 相关的，所以 Context 的主要应用场景会是网络请求的收发和处理上
2. 一个 request 应该对应一个 Context，这指导了 Context 的用法和使用时应该绑定请求，最好不要在多个请求的处理流程中复用同一个 Context

后面就是讲用法和注意事项了（L9 - L23，有点长，就不 copy docstring 过来了，这里是做个小总结，推荐大家对照着注释读）：

1. 收到请求时要创建 Context，调用外部函数了要传 Context 过去
2. 要维持 Context 链，这里提到了几个操作：`WithCancel`、`WithDeadline`、`WithTimeout`、`WithValue`，这几个操作可以用来制造子孙 Context，待会儿要重点看看
3. Context 的 Cancel 处理（Cancel 译成终止）
   1. 当一个 Context 被终止时，所有基于它生成的 Context 也会被终止
   2. `WithCancel`、`WithDeadline`、`WithTimeout` 都会生成一个 `CancelFunc` 用来终止 Context，使用时请确保每一条控制流路径上都会调用`CancelFunc`（go vet 可以检查这一项的 好东西）

接下来是几条「守则」（L25 - L44），推荐大家都遵守这些守则，可以保持各个包中的接口都一致，同时也能让静态分析工具（比如后面会提到的 go vet）来分析 Context 的使用。我们来看看都有些啥：

1. 不要把 Context 放在结构体里传
   1. Context 应该作为参数传递
   2. Context 还应该作为函数的第一个参数
   3. 推荐把这个参数命名为 ctx
2. 不要用 nil Context
   1. 有的函数能接受 nil Context，但是还是不要用
   2. 如果不知道用什么 Context 的话就用 `context.TODO()`
   3. 这儿出现了 `.TODO()`，这个名字有点怪怪的，之后我们也会重点讲讲它，`TODO()` 是个很灵性的设计
3. Context 中有个概念叫 Value，不要把 Value 当作传参工具，Value 应该用来传跨 API 和进程的数据
4. Context 是并发安全的

# 接口声明

包注释结束后，`type Context interface{`就来了，开门见山。Context 统一地约束了上下文的功能，向外提供了四个方法：

1. `Deadline() (deadline time.Time, ok bool)`
   - 超时控制相关
   - `deadline` 是该 Context 被终止的时刻
   - 如果还没设置终止时刻的话，`ok == false`
   - 成功调用 `Deadline` 的返回都是相同的（幂等性）
2. `Done() <-chan struct{}`
   - 当该 Context 被终止时，`Done()` 也会被关闭
   - `Done() == nil` 说明该 Context 不能被终止
   - 该方法也具有幂等性
   - 一般结合 `select` 使用，如：

```go
select {
	case <-ctx.Done():
		return ctx.Err()
	case ...
}
```

3. `Err() error`
   - 如果 `Done()` 还没被关闭，`Err() == nil`
   - 如果 `Done()` 被关闭了，`Err` 返回一个 `error` 解释终止的原因
     - 返回 `Canceled` 说明是被手动终止的
     - 返回 `DeadlineExceeded` 说明是超时终止的
   - 这个方法也具有幂等性
4. `Value(key interface{}) interface{}`
   - `Value` 返回该 Context 中与 `key` 关联的 `value`
     - 如果该 Context 中 `key` 没有关联 `value`，就会返回 `nil`
   - 这个方法也具有幂等性

# 具体实现

## error 的声明与实现

声明完 `Context` 接口，接着就是 `error` 的声明：

```go
// Canceled is the error returned by Context.Err when the context is canceled.
var Canceled = errors.New("context canceled")

// DeadlineExceeded is the error returned by Context.Err when the context's
// deadline passes.
var DeadlineExceeded error = deadlineExceededError{}

type deadlineExceededError struct{}

func (deadlineExceededError) Error() string   { return "context deadline exceeded" }
func (deadlineExceededError) Timeout() bool   { return true }
func (deadlineExceededError) Temporary() bool { return true }
```

在看 `Context.Err` 的注释时，我还以为 `Canceled` 和 `DeadlineExceeded` 是 `type`，用的时候再初始化，看了实现才知道是用默认对象的方法来做的

go 的标准库中有很多功能都是通过默认值/默认实现来提供的，而我以前写其他语言的代码时就比较少接触这种模式，这也算一种特色吧，要学习一下

另：context 包这个排布方法比较反我的个人直觉，我目前想到的原因：

1. 在 `Context.Err` 的注释中提到了 `error`，所以按就近原则要前置
2. 这两个 `error` 是用默认值实现的，算常量，应该前置

## emptyCtx: 起源

这是最基础的 Context：不会被终止、不能携带 value、没有 deadline，emptyCtx 的声明：

```go
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int
```

底层使用 int 作为占位符，接口使用空实现来提供，

包中定义了两个常量 emptyCtx：

```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

// Background returns a non-nil, empty Context. It is never canceled, has no
// values, and has no deadline. It is typically used by the main function,
// initialization, and tests, and as the top-level Context for incoming
// requests.
func Background() Context {
	return background
}

// TODO returns a non-nil, empty Context. Code should use context.TODO when
// it's unclear which Context to use or it is not yet available (because the
// surrounding function has not yet been extended to accept a Context
// parameter).
func TODO() Context {
	return todo
}
```

又是默认对象实现方法，这两个 context 的使用场景也在注释中给出，可谓是万物之源

### 问题

- 为什么要分成两个 context，background 表达能力不足吗
  - 自答：background 是所有 context 之根，所以使用情景也很单一：main 线程中，其他情况下要创建新的 context 树都使用`TODO()`
  - 如果只用 background 的话，context 树就可能出现环了

## WithCancel()：第一步

WithCancel 接受一个 parent context，返回一个 child context 和一个 CancelFunc，调用该 CancelFunc 就可以终止 child

这是我们接触到的第一个派生新 context 的函数。WithCancel 让我们能够手动控制一个 context 的终止

### CancelFunc

先来看看这个 WithCancel 给我们的榔头怎么用和长啥样，CancelFunc 的注释与 typedef 如下：

```go
// A CancelFunc tells an operation to abandon its work.
// A CancelFunc does not wait for the work to stop.
// A CancelFunc may be called by multiple goroutines simultaneously.
// After the first call, subsequent calls to a CancelFunc do nothing.
type CancelFunc func()
```

1. CancelFunc 是用来通知下游该收手了
2. CancelFunc 即调即起效（当然也需要下游适配）
3. CancelFunc 也是并发安全的、幂等的

### 内部实现

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

可见逻辑非常简单：

1. 检查入参
2. 在 parent 上挂一个新的 child context
3. `propagateCancel` 对 parent 和 child 进行设置，使得 parent 被终止时同步终止 child
   - 回想：context 是树形结构排布的
4. 使用闭包做出 CancelFunc 并返回
   - 注意：这个 CancelFunc 是用以终止 child 的

## propagateCancel

这个函数用来设置终止操作的传播关系，使得「当一个 Context 被终止时，所有基于它生成的 Context 也会被终止」这条性质得以成立

源码：

```go
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err())
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		atomic.AddInt32(&goroutines, +1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

流程如下，大家可以对照着理解：

1. 检查入参（包括检查 parent 是否已经被终止）
2. 获取 parent 内部的 cancelCtx 用以注册终止事件调用链
   - 当 p 被终止时，会递归地终止 p.children
3. 如果 parent 不是一个 cancelCtx，那么就手动开一个 goro 监听 parent 的终止事件（终止信号是 parent.Done()被 closed）

在`parentCancelCtx`中，有一个出于兼容性考量的设计：

```go
// checking whether
// parent.Done() matches that *cancelCtx. (If not, the *cancelCtx
// has been wrapped in a custom implementation providing a
// different done channel, in which case we should not bypass it.)
```

> 来源于`parentCancelCtx`的注释

在拿到 parent 的 cancelCtx 后，为了避免用户自定义了 Done channel 的实现导致我们「终止错了」context，所以此处还检查了`parent.Done()==parent.cancelCtx.done`

这个设计还体现了「接口为本」：具体实现（parent.cancelCtx.done）和接口实现（parent.Done()）出现了冲突，以 parent.Done() 为准（抛到外面手动开一个 goro 监听 parent.Done() 被 close 的事件）

## cancelCtx

cancelCtx 顾名思义，是可被终止的 context，结构定义：

```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

### canceler

cancelCtx.children 是一个 set，key 的类型为 canceler，canceler 是所有可被**直接**终止的 context 所需要实现的接口，定义也很简单：

```go
// A canceler is a context type that can be canceled directly. The
// implementations are *cancelCtx and *timerCtx.
type canceler interface {
	cancel(removeFromParent bool, err error) // 调用cancel()来终止
	Done() <-chan struct{} // 同 Context.Done
}
```

### cancelCtx.Value()

cancelCtx.Value 对 cancelCtxKey 进行了特殊处理，参见 cancelCtxKey 的注释：

```go
// &cancelCtxKey is the key that a cancelCtx returns itself for.
var cancelCtxKey int
```

## WithDeadline()

这是我们接触到的第二个派生新 context 的函数，我们先来看一眼签名：

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
```

来试试看名说话，猜猜实现方法：

1. 接收一个 time.Time，名字又是 WithDeadline，想必是到了 d 指定的时间就终止 child
   - 这个定时终止的实现方法我觉得是开个定时器+用闭包终止，毕竟都引入 time 了，用定时器很方便
   - 我最开始（只看函数名还没看入参类型时）的想法是开个 goro 自己终止，这显然也可以，定时器内部也是开 goro 的
2. 返回 Context+CancelFunc，按 WithCancel 的路子来推测的话，前者是 child，后者应该是用来提前手动终止 child 的
   - 这是不是也暗示 WithDeadline 内部会调用/参考 WithCancel
   - 比如先 WithCancel 出 child 和 cancel() 来，然后开个定时器调用 cancel()

看看答案：

```go
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

处理流程：

1. 检查入参
   1. parent 是否为 nil
   2. parent.Deadline() 是否比 d 还早
      - 如果比 d 还早，那就退化为 WithCancel(parent)
2. 创建 child
   - 注意：这里用 newCancelCtx 包装了 parent
   - 所以这里也调用了 propagateCancel，mark 一下
   - 问题 1：为什么要包装 parent
3. 检查 d 是不是已经过了
   - 如果已经过了，那就终止掉 child，然后正常返回 child 和 cancelFunc
   - 问题 2：为什么在这里才检查
4. 设置 c.timer
   - 问题 3：这里检查了 c.err==nil，为什么要检查，然后为什么检查的是 err 字段

答案（个人观点）：

1. 为了简化代码：Don't repeat yourself，包装 parent 来造一个 sidecar，这样就可以把 cancel 相关的逻辑交给 cancelCtx，不用 copy-paste 了
   - 用这种方法的话需要注意一下 child, parent 和 sidecar 之间的指向关系，推荐画个图
2. 为了逻辑的完整性与连贯性：这种情况不能回退，那就应该正常创建 child，所以此处需要先创建 child 再检查 d
3. c.err 是定义在 cancelCtx 中的，err!=nil 表示已经被终止，设置 timer 前检查是否已经被终止是为了避免重复终止（避免重复终止同一个 context 也是使用原则之一）；而检查是否终止其实是有多种方式的，检查 err 字段我认为是最简单的

## timerCtx

看到这里的话，timerCtx 应该比较好理解了：组合了一个 cancelCtx，Context 的四个接口中委托出去三个，只重写了 Deadline()

不过 timerCtx 也是支持手动终止的，且手动终止 timerCtx 时需要关闭自己的 timer，所以重写了 cancel（这是 canceler 接口的函数），当然，细看 cancel 的话还能发现这里特别处理了 cancelCtx（前文提到的 sidecar，伏笔了）

## WithTimeout

这个就很简单了：

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

## WithValue

```go
func WithValue(parent Context, key, val interface{}) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

这是给 context 加 value 的函数，这个函数同样会生成一个子 Context，流程也很简单，对三个入参进行检查，然后新建子 Context 并返回

## valueCtx

valueCtx 很有意思：长得像个 map，但实际上每个 valueCtx 只存一对 key-value，如果自己没搜到 key 的话就递归地往父 context 搜

关于这个特性的设计，我的理解是为了做 variable shadowing：这样就可以在不污染父 context 的 value map 的同时重写同名 key

其他地方就很简单直白了，因为只要管一对 key-value，所以也只需要实现 Value() 接口，其他的接口都委托给父 Context：

```go
// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) String() string {
	return contextName(c.Context) + ".WithValue(type " +
		reflectlite.TypeOf(c.key).String() +
		", val " + stringify(c.val) + ")"
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}

```

String() 接口是 stringer 的接口，大家很熟悉就不作展开了

# 总结

五百多行的小包写得很漂亮，作为一个网络的基础包，做到了足够精简，同时也提供了拓展性默认提供的四种实现（emptyCtx cancelCtx timerCtx valueCtx）覆盖了
