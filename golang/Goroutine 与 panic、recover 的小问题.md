本文来源：https://golang2.eddycjy.com/posts/appendix/02-goroutine-panic/

在 Go 语言中，goroutine、panic、recover 是人尽皆知关键字，几乎在每一个项目中，你必定会主动地使用到它。即使你不主动使用，你也无法避免你所使用的标准库、第三方外部依赖模块使用它。

虽然它们在程序中非常常见，但依然会有许多刚入门的开发者在初次使用时遇到小 “坑”，并对这个处理结果都表现出很震惊，接下来在本文中我们将对这一个小 ”坑“ 进行说明。

# B.1 思考问题
```
func main() {
	go func() {
		panic("煎鱼焦了")
	}()

	log.Println("Go 语言编程之旅：一起用 Go 做项目")
}
```
我们思考一下上述程序，其输出的结果是书名，还是会因为 “煎鱼焦了” 而直接中断程序，结果如下：

```
panic: 煎鱼焦了

goroutine 6 [running]:
main.main.func1()
        /Users/eddycjy/go/src/github.com/eddycjy/awesomeProject/main.go:7 +0x39
created by main.main
        /Users/eddycjy/go/src/github.com/eddycjy/awesomeProject/main.go:6 +0x35
        
```
最终的结果是程序因为 “煎鱼焦了” 而中断运行，这时候经常会有人提出一个疑问，就是我的 panic 语句是写在子协程（goroutine）里的，怎么会影响外面的主协程（main goroutine）呢，它们不是应该相互隔离的吗，怎么会互相影响呢？

# B.2 如何解决
针对这个现象，我们要如何解决呢，首先对于 ```panic``` 事件，大家都知道要使用组合方法 ```recover``` 来进行处理，如下：

```
func main() {
	go func() {
		if e := recover(); e != nil {
			log.Printf("recover: %v", e)
		}
		panic("煎鱼焦了")
	}()

	log.Println("Go 语言编程之旅：一起用 Go 做项目")
	time.Sleep(time.Second)
}
```
但是单单使用 ```recover```，依旧会输出 “煎鱼焦了”，并且程序中断，正确的方式如下：

```
func main() {
	go func() {
		defer func() {
			if e := recover(); e != nil {
				log.Printf("recover: %v", e)
			}
		}()
		panic("煎鱼焦了")
	}()

	log.Println("Go 语言编程之旅：一起用 Go 做项目")
}
```
实际上 ```recover``` 要与 ```defer``` 联用，并且不跨协程（goroutine），才能够真正的拦截到 panic 事件，其最终的输出结果如下：

```
Go 语言编程之旅：一起用 Go 做项目
recover: 煎鱼焦了
```
# B.3 为什么要 defer 才能 recover
从前文中我们可以知道，除了 ```panic```、```recover``` 以外，还必须要有 ```defer``` 关键字，缺一不可，那么与 ```defer``` 又有什么关系呢，为什么必须要有 ```defer``` 后 ```recover``` 才能起作用？

## B.3.1 快速了解 panic
```panic``` 是 Go 语言中的一个内置函数，可以停止程序的控制流，改变其流转，并且触发恐慌事件。而 ```recover``` 也是一个内置函数，但其功能与 ```panic``` 相对，```recover``` 可以让程序重新获取恐慌后的程序控制权，但是必须在``` defer``` 中 ```recover``` 才会生效。

而 ```panic``` 的一切，都基于一个 ```_panic ```基础单元，基本结构如下：

```
type _panic struct {
	argp      unsafe.Pointer 
	arg       interface{}    
	link      *_panic       
	pc        uintptr        
	sp        unsafe.Pointer 
	recovered bool           
	aborted   bool        
	goexit    bool
}
```
在我们每执行一次 ```panic``` 语句时，都会创建一个 ```_panic```。它包含了一些基础的字段用于存储当前的 panic 调用情况，涉及的字段如下：


- argp：指向 ```defer``` 延迟调用的参数的指针。
- arg：```panic``` 的原因，也就是调用 ```panic``` 时传入的参数。
- link：指向上一个调用的 ```_panic```。
- pc：程序计数器，有时称为指令指针（IP），线程利用它来跟踪下一个要执行的指令。在大多数处理器中，PC 指向的是下一条指令，而不是当前指令。
- sp：函数栈指针寄存器，一般指向当前函数栈的栈顶。
- recovered：```panic``` 是否已经被处理，也就是是否被 ```recover```。
- aborted：```panic``` 是否被中止。
- goexit：是否调用 ```runtime.Goexit``` 方法中止过主 ```goroutine``` 及所属的 ```goroutine```。

通过查看 ```link``` 字段，可得知 ```panic``` 的基本单元是一个链表的数据结构，如下图：

<img src="https://golang2.eddycjy.com/images/appendix/panic.jpg">

## B.3.2 快速了解 defer
```defer``` 是 Go 语言中的一个内置函数，```defer``` 方法所注册的对应事件会在函数/方法结束后执行，常用于关闭各类资源以及兜底操作。而相对于 panic 的基础单元 ```_panic``` 结构体，```defer``` 也有 ```_defer``` 结构体，基本结构如下：

```
type _defer struct {
    siz     int32
    started bool
    sp      uintptr 
    pc      uintptr
    fn      *funcval
    _panic  *_panic
    link    *_defer
    ...
}

type funcval struct {
    fn uintptr
    // variable-size, fn-specific data here
}
```

- siz：所有传入参数的总大小。
- started：该 ```defer``` 是否已经执行过。
- sp：函数栈指针寄存器，在 ```_panic``` 时已介绍。
- pc：程序计数器，在 ```_panic``` 时已介绍。
- fn：指向传入的函数地址和参数。
- _panic：指向 ```_panic``` 链表。
- link：指向 ```_defer``` 链表。

通过查看 ```_panic``` 和 ```link``` 字段，我们可得知 ```defer``` 也是同时挂载着 ```panic``` 信息的，如下：

<img src="https://golang2.eddycjy.com/images/appendix/defer.jpg">

## B.3.3 recover 是如何和 defer 搭上关系的
刚刚我们一直看到的是 ```defer``` 和 ```panic``` 存在的一定的关联关系，那么 ```recover``` 又和它们是怎么产生关系的呢，为什么不用 ```defer```，```recover``` 就无法生效？

为了解答这些问题，我们要回到一切的起源 ```panic``` 才能知晓一二，```panic``` 关键字的具体代码实现如下：

```
func gopanic(e interface{}) {
    gp := getg()
    ...
    var p _panic
    p.arg = e
    p.link = gp._panic
    gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

    for {
        d := gp._defer
        if d == nil {
            break
        }

        // defer...
        ...
        d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

        p.argp = unsafe.Pointer(getargp(0))
        reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
        p.argp = nil

        // recover...
        if p.recovered {
            ...
            mcall(recovery)
            throw("recovery failed") // mcall should not return
        }
    }

    preprintpanics(gp._panic)
    fatalpanic(gp._panic) // should not return
    *(*int)(nil) = 0      // not reached
}
```

通过分析上述代码，我们可以大致了解到其处理过程：

- 获取指向当前 ```Goroutine``` 的指针。
- 初始化一个 ```panic``` 的基本单位 ```_panic``` 用作后续的操作。
- 获取当前 ```Goroutine``` 上挂载的 ```_defer```。
- 若当前存在 ```defer``` 调用，则调用 ```reflectcall``` 方法去执行先前 ```defer``` 中延迟执行的代码，若在执行过程中需要运行 ```recover``` 将会调用 ```gorecover``` 方法。
- 中断程序结束前，调用 ```preprintpanics``` 方法打印出所涉及的 ```panicv 消息。
- 最后调用 ```fatalpanic``` 中止应用程序，实际是执行 ```exit(2)``` 进行最终退出行为的。

再回到我们的问题 “recover 是如何和 defer 搭上关系的”，我们可得知在调用 ```panic``` 方法后，```runtime.gopanic``` 方法实际上处理的是当前 Goroutine 上所挂载的 ```._panic``` 链表（所以无法对其他 Goroutine 的异常事件响应），然后会对其所属的 ```defer``` 链表和 ```recover``` 进行检测并处理，最后调用退出命令中止应用程序。

<img src="https://golang2.eddycjy.com/images/appendix/g-defer-panic.jpg">

从代码实现上来讲，因为 ```panic``` 会触发延迟调用（defer），那么假设当前 Goroutine 不存在 ```defer``` 的话，就会直接跳出，也就无法进行 ```recover ``` 了，也就是在 ```panic``` 时 Go 只会在 ```defer``` 中对 ```reocver``` 进行检测。

而从设计实现上来讲，这也是相对合理的，因为我们无法执行到哪都写一个 ```recover```，很多错误你是无法预料在哪里发生的，又是如何发生的。

# B.4 recover 是万能的吗
想太多了，有了 ```recover``` 并不代表你能够捕获到所有的错误。

就在某一天，你的程序还在线上环境运行着，突然就挂了，刚好这程序在容器里运行，它反复的重启，但每次都不是马上出问题，都是运行了一段时间后出现了宕机，你一脸懵逼，难道有泄露了？

但不过这个程序非常简短，就是个简单的并发清洗、组装数据，你查看到核心（伪）代码如下：

```
func main() {
	m := make(map[int]string)
	for i := 0; i < 10; i++ {
		go func() {
			defer func() {
				if e := recover(); e != nil {
					log.Printf("recover: %v", e)
				}
			}()

			m[i] = "Go 语言编程之旅：一起用 Go 做项目"
		}()
	}

	// do something...
}
```

你心想，我都听着煎鱼说的把 recover 都加进 goroutine 里了，怎么还会出现无法捕获的错误，还导致程序挂了，难道煎鱼教的是错的吗？

同时你查看了对应的控制台日志，其关键信息如下：

```
fatal error: concurrent map writes

goroutine 21 [running]:
runtime.throw(0x10d2c3b, 0x15)
        /usr/local/Cellar/go/1.14/libexec/src/runtime/panic.go:1112 +0x72 fp=0xc000029f50 sp=0xc000029f20 pc=0x102e892
runtime.mapassign_fast64(0x10b58e0, 0xc000090180, 0x8, 0x0)
        /usr/local/Cellar/go/1.14/libexec/src/runtime/map_fast64.go:101 +0x323 fp=0xc000029f90 sp=0xc000029f50 pc=0x100f733
```

通过错误信息我们可得知这是一个很常见的问题，就是并发写入 map 导致的致命错误，但是为什么不可以被 ```recover``` 捕获到呢，我们关注到 ```runtime.throw``` 方法，代码如下：

```
func throw(s string) {
	systemstack(func() {
		print("fatal error: ", s, "\n")
	})
	gp := getg()
	if gp.m.throwing == 0 {
		gp.m.throwing = 1
	}
	fatalthrow()
	*(*int)(nil) = 0 // not reached
}
```
关键的中断步骤在于 ```fatalthrow``` 方法，如下：

```
func fatalthrow() {
	...
	systemstack(func() {
		...
		exit(2)
	})

	*(*int)(nil) = 0 // not reached
}
```
我们可以看到该方法是直接通过调用 ```exit``` 方法进行中断的，而实际上在 Go 语言中，是存在着这些无法恢复的 ”恐慌“，例如像是 ```fatalthrow```、```fatalpanic``` 等等方法，因此自然而然使用 recover 就无法捕获到了，因为它是直接退出程序，结果是中断程序。

因此 ```recover``` 并非万能，它只针对用户态下的 ```panic``` 关键字有效。

# B.5 小结
在本文中我们针对 ```panic``` 的常见问题，基于 ```goroutine```、```panic```、```recover``` 做了初步的分析，而在解析 ```recover``` 相关行为时，我们发现其与 ```defer``` 是存在关联关系的，其四者本质上是一个互相联动的关系。

在最后我们可以总结出如下使用细节：

```panic``` 只能触发当前 Goroutine 的 ```defer``` 调用，在 ```defer``` 调用中如果存在 ```recover``` ，那么就能够处理其所抛出的恐慌事件。但是需要注意的是在其它 Goroutine 中的 ```defer``` 是对其没有用的，并不支持跨协程（goroutine），需要分清楚。
想捕获/处理 ```panic``` 所造成的恐慌，```recover``` 必须与 ```defer``` 配套使用，否则无效。
在 Go 语言中，是存在着无法处理的致命错误方法的，例如：```fatalthrow```、```fatalpanic``` 方法，一般会在并发写入 map 等等处理时抛出，需要谨慎。
