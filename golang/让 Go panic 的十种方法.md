本文来源：https://golang2.eddycjy.com/posts/appendix/04-go-panic/

本章节有且仅有一个目的，那就是让你的 Go 程序遇到 panic。

# D.1 数组/切片索引越界

```
func main() {
	names := []string{
		"煎鱼",
		"eddycjy",
		"Go 语言编程之旅",
	}
	name := names[len(names)]
	fmt.Printf("name: %s", name)
}
```
运行结果：

```
panic: runtime error: index out of range [3] with length 3

goroutine 1 [running]:
main.main()
        /Users/eddycjy/go-application/awesomePanic/main.go:11 +0x1b
```
# D.2 空指针调用

```
type User struct {
	Name string
}

func (u *User) GetName() string {
	return u.Name
}

func main() {
	s := &User{Name: "煎鱼"}
	s = nil
	s.GetName()
}
```
运行结果：

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x1056f62]

goroutine 1 [running]:
main.(*User).GetName(...)
        /Users/eddycjy/go-application/awesomePanic/main.go:8
main.main()
        /Users/eddycjy/go-application/awesomePanic/main.go:14 +0x2
```
# D.3 过早关闭 HTTP 响应体

```
func main() {
	resp, err := http.Get("https://xxx")
	defer resp.Body.Close()
	if err != nil {
		log.Fatalf("http.Get err: %v", err)
	}
}
```

运行结果：

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x40 pc=0x123d4d3]

goroutine 1 [running]:
main.main()
        /Users/eddycjy/go-application/awesomePanic/main.go:10 +0x63
```
# D.4 除以零

```
func divide(a, b int) int {
	return a / b
}

func main() {
	divide(1, 0)
}
```
运行结果：


```
panic: runtime error: integer divide by zero

goroutine 1 [running]:
main.divide(...)
        /Users/eddycjy/go-application/awesomePanic/main.go:4
main.main()
        /Users/eddycjy/go-application/awesomePanic/main.go:8 +0x12
```
# D.5 向已关闭的通道发消息

```
func main() {
	ch := make(chan string, 1)
	ch <- "煎鱼"
	close(ch)
	ch <- "Go 语言编程之旅"
}
```
运行结果：
```
panic: send on closed channel

goroutine 1 [running]:
main.main()
        /Users/eddycjy/go-application/awesomePanic/main.go:7 +0x7d
```
# D.6 重复关闭通道

```
func main() {
	ch := make(chan string, 1)
	ch <- "Go 语言编程之旅"
	close(ch)
	close(ch)
}

```
运行结果：

```
panic: close of closed channel

goroutine 1 [running]:
main.main()
        /Users/eddycjy/go-application/awesomePanic/main.go:7 +0x71
```
# D.7 关闭未初始化的通道

```
func main() {
	var ch chan string
	close(ch)
}
```
运行结果：

```
panic: close of nil channel

goroutine 1 [running]:
main.main()
        /Users/eddycjy/go-application/awesomePanic/main.go:5 +0x2a

```
# D.8 未初始化 map

```
func main() {
	var m map[string]string
	m["Go 语言编程之旅"] = "一起用 Go 做项目"
}
```
运行结果：


```
panic: assignment to entry in nil map

goroutine 1 [running]:
main.main()
        /Users/eddycjy/go-application/awesomePanic/main.go:5 +0x4b
```
# D.9 跨协程的恐慌处理

```
func main() {
	go func() {
		defer func() {
			if err := recover(); err != nil {
				log.Fatalf("recover err: %v", err)
			}
		}()
		handle()
	}()

	time.Sleep(time.Second)
}

func handle() {
	go divide(1, 0)
}

func divide(a, b int) int {
	return a / b
}
```
运行结果：


```
panic: runtime error: integer divide by zero

goroutine 17 [running]:
main.divide(0x1, 0x0, 0x0)
        /Users/eddycjy/go-application/awesomePanic/main.go:34 +0x40
created by main.handle
        /Users/eddycjy/go-application/awesomePanic/main.go:30 +0x47

```
# D.10 sync 计数为负值

```
func main() {
	wg := sync.WaitGroup{}
	wg.Add(-1)
	wg.Wait()
}
```
运行结果：


```
panic: sync: negative WaitGroup counter

goroutine 1 [running]:
sync.(*WaitGroup).Add(0xc000104000, 0xffffffffffffffff)
        /usr/local/Cellar/go/1.14/libexec/src/sync/waitgroup.go:74 +0x139
main.main()
        /Users/eddycjy/go-application/awesomePanic/main.go:7 +0x44
```
