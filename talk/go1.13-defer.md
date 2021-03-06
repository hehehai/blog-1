# Go1.13 defer 的性能是如何提高的？

最近 Go1.13 终于发布了，其中一个值得关注的特性就是 **defer 在大部分的场景下性能提升了 30%**，但是官方并没有具体写是怎么提升的，这让大家非常的疑惑。而我因为之前写过[《深入理解 Go defer》](https://book.eddycjy.com/golang/defer/defer.html) 和 [《Go defer 会有性能损耗，尽量不要用？》](https://book.eddycjy.com/golang/talk/defer-loss.html) 这类文章，因此我挺感兴趣它是做了什么改变才能得到这样子的结果，所以今天和大家一起探索其中奥妙。

## 一、测试

### Go1.12

```
$ go test -bench=. -benchmem -run=none
goos: darwin
goarch: amd64
pkg: github.com/EDDYCJY/awesomeDefer
BenchmarkDoDefer-4      	20000000	        91.4 ns/op	      48 B/op	       1 allocs/op
BenchmarkDoNotDefer-4   	30000000	        41.6 ns/op	      48 B/op	       1 allocs/op
PASS
ok  	github.com/EDDYCJY/awesomeDefer	3.234s
```

### Go1.13

```
$ go test -bench=. -benchmem -run=none
goos: darwin
goarch: amd64
pkg: github.com/EDDYCJY/awesomeDefer
BenchmarkDoDefer-4      	15986062	        74.7 ns/op	      48 B/op	       1 allocs/op
BenchmarkDoNotDefer-4   	29231842	        40.3 ns/op	      48 B/op	       1 allocs/op
PASS
ok  	github.com/EDDYCJY/awesomeDefer	3.444s
```

在开场，我先以不标准的测试基准验证了先前的测试用例，确确实实在这两个版本中，`defer` 的性能得到了提高，但是看上去似乎不是百分百提高 30 %。

## 二、看一下

### 之前（Go1.12）

```
    0x0070 00112 (main.go:6)    CALL    runtime.deferproc(SB)
    0x0075 00117 (main.go:6)    TESTL    AX, AX
    0x0077 00119 (main.go:6)    JNE    137
    0x0079 00121 (main.go:7)    XCHGL    AX, AX
    0x007a 00122 (main.go:7)    CALL    runtime.deferreturn(SB)
    0x007f 00127 (main.go:7)    MOVQ    56(SP), BP
```

### 现在（Go1.13）

```
	0x006e 00110 (main.go:4)	MOVQ	AX, (SP)
	0x0072 00114 (main.go:4)	CALL	runtime.deferprocStack(SB)
	0x0077 00119 (main.go:4)	TESTL	AX, AX
	0x0079 00121 (main.go:4)	JNE	139
	0x007b 00123 (main.go:7)	XCHGL	AX, AX
	0x007c 00124 (main.go:7)	CALL	runtime.deferreturn(SB)
	0x0081 00129 (main.go:7)	MOVQ	112(SP), BP
```

从汇编的角度来看，像是 `runtime.deferproc` 改成了 `runtime.deferprocStack` 调用，难道是做了什么优化，我们**抱着疑问**继续看下去。

## 三、观察源码

### \_defer

```go
type _defer struct {
	siz     int32
	siz     int32 // includes both arguments and results
	started bool
	heap    bool
	sp      uintptr // sp at time of defer
	pc      uintptr
	fn      *funcval
	...
```

相较于以前的版本，最小单元的 `_defer` 结构体主要是新增了 `heap` 字段，用于标识这个 `_defer` 是在堆上，还是在栈上进行分配，其余字段并没有明确变更，那我们可以把聚焦点放在 `defer` 的堆栈分配上了，看看是做了什么事。

### deferprocStack

```go
func deferprocStack(d *_defer) {
	gp := getg()
	if gp.m.curg != gp {
		throw("defer on system stack")
	}

	d.started = false
	d.heap = false
	d.sp = getcallersp()
	d.pc = getcallerpc()

	*(*uintptr)(unsafe.Pointer(&d._panic)) = 0
	*(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))
	*(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

	return0()
}
```

这一块代码挺常规的，主要是获取调用 `defer` 函数的函数栈指针、传入函数的参数具体地址以及 PC（程序计数器），这块在前文 [《深入理解 Go defer》](https://book.eddycjy.com/golang/defer/defer.html) 有详细介绍过，这里就不再赘述了。

那这个 `deferprocStack` 特殊在哪呢，我们可以看到它把 `d.heap` 设置为了 `false`，也就是代表 `deferprocStack` 方法是针对将 `_defer` 分配在栈上的应用场景的。

### deferproc

那么问题来了，它又在哪里处理分配到堆上的应用场景呢？

```go
func newdefer(siz int32) *_defer {
	...
	d.heap = true
	d.link = gp._defer
	gp._defer = d
	return d
}
```

那么 `newdefer` 是在哪里调用的呢，如下：

```go
func deferproc(siz int32, fn *funcval) { // arguments of fn follow fn
	...
	sp := getcallersp()
	argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
	callerpc := getcallerpc()

	d := newdefer(siz)
	...
}
```

非常明确，先前的版本中调用的 `deferproc` 方法，现在被用于对应分配到堆上的场景了。

### 小结

- 第一点：可以确定的是 `deferproc` 并没有被去掉，而是流程被优化了。
- 第二点：编译器会根据应用场景去选择使用 `deferproc` 还是 `deferprocStack` 方法，他们分别是针对分配在堆上和栈上的使用场景。

## 四、编译器如何选择

### esc

```
// src/cmd/compile/internal/gc/esc.go
case ODEFER:
	if e.loopdepth == 1 { // top level
		n.Esc = EscNever // force stack allocation of defer record (see ssa.go)
		break
	}
```

### ssa

```
// src/cmd/compile/internal/gc/ssa.go
case ODEFER:
	d := callDefer
	if n.Esc == EscNever {
		d = callDeferStack
	}
    s.call(n.Left, d)
```

### 小结

这块结合来看，核心就是当 `e.loopdepth == 1` 时，会将逃逸分析结果 `n.Esc` 设置为 `EscNever`，也就是将 `_defer` 分配到栈上，那这个 `e.loopdepth` 到底又是何方神圣呢，我们再详细看看代码，如下：

```
// src/cmd/compile/internal/gc/esc.go
type NodeEscState struct {
	Curfn             *Node
	Flowsrc           []EscStep
	Retval            Nodes
	Loopdepth         int32
	Level             Level
	Walkgen           uint32
	Maxextraloopdepth int32
}
```

这里重点查看 `Loopdepth` 字段，目前它共有三个值标识，分别是:

- -1：全局。
- 0：返回变量。
- 1：顶级函数，又或是内部函数的不断增长值。

这个读起来有点绕，结合我们上述 `e.loopdepth == 1` 的表述来看，也就是当 `defer func` 是顶级函数时，将会分配到栈上。但是若在 `defer func` 外层出现显式的迭代循环，又或是出现隐式迭代，将会分配到堆上。其实深层表示的还是迭代深度的意思，我们可以来证实一下刚刚说的方向，显式迭代的代码如下：

```go
func main() {
	for p := 0; p < 10; p++ {
		defer func() {
			for i := 0; i < 20; i++ {
				log.Println("EDDYCJY")
			}
		}()
	}
}
```

查看汇编情况：

```
$ go tool compile -S main.go
"".main STEXT size=122 args=0x0 locals=0x20
	0x0000 00000 (main.go:15)	TEXT	"".main(SB), ABIInternal, $32-0
	...
	0x0048 00072 (main.go:17)	CALL	runtime.deferproc(SB)
	0x004d 00077 (main.go:17)	TESTL	AX, AX
	0x004f 00079 (main.go:17)	JNE	83
	0x0051 00081 (main.go:17)	JMP	33
	0x0053 00083 (main.go:17)	XCHGL	AX, AX
	0x0054 00084 (main.go:17)	CALL	runtime.deferreturn(SB)
	...
```

显然，最终 `defer` 调用的是 `runtime.deferproc` 方法，也就是分配到堆上了，没毛病。而隐式迭代的话，你可以借助 `goto` 语句去实现这个功能，再自己验证一遍，这里就不再赘述了。

## 总结

从分析的结果上来看，官方说明的 Go1.13 defer 性能提高 30%，主要来源于其延迟对象的堆栈分配规则的改变，措施是由编译器通过对 `defer` 的 `for-loop` 迭代深度进行分析，如果 `loopdepth` 为 1，则设置逃逸分析的结果，将分配到栈上，否则分配到堆上。

的确，我个人觉得对大部分的使用场景来讲，是优化了不少，也解决了一些人吐槽 `defer` 性能 “差” 的问题。另外，我想从 Go1.13 起，你也需要稍微了解一下它这块的机制，别随随便便就来个狂野版嵌套迭代 `defer`，可能没法效能最大化。

如果你还想了解更多细节，可以看看 `defer` 这块的的[提交内容](https://github.com/golang/go/commit/fff4f599fe1c21e411a99de5c9b3777d06ce0ce6)，官方的测试用例也包含在里面。
