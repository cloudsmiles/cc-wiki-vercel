# channel与锁的性能

最近发现同事在一个需要多并发的场景下, 使用channel来保证并发安全, 大概问了下原因, 他说感觉channel应该要比加锁的方式性能要好. 然后网上搜了下居然有 “Channel 优于锁机制” 这种言论, 不明确具体的使用场景就说出这种言论的人太不负责了, 在有些场景下比如不同的goroutinue之间进行通信, 那么使用channel是最好不过了, 但是在一些多并发场景下使用channel来保证并发安全, 那么性能表现肯定比不上直接加锁, 代码的复杂度也相差无几.

下面来简单的做一个 benchmark, 来看一看 Mutex 和 channel 的性能差别.

首先在一个目录下新建两个文件:

- main.go
- main_test.go

main.go 文件内容如下:

```
// main.go
package main
import "sync"
var mutex = sync.Mutex{}
var ch = make(chanbool, 1)
funcUseMutex() {
	mutex.Lock()
	mutex.Unlock()
}
funcUseChan() {
	ch <-true
	<-ch
}
```

main_test.go 文件内容如下:

```
// main_test.go
package main
import "testing"
funcBenchmarkUseMutex(b *testing.B) {
for n := 0; n < b.N; n++ {
		UseMutex()
	}
}
funcBenchmarkUseChan(b *testing.B) {
for n := 0; n < b.N; n++ {
		UseChan()
	}
}
```

然后在该文件夹下面执行如下命令进行 benchmark:

```
$ go test -bench=.
```

压测结果如下:

```
BenchmarkUseMutex-4   	100000000	        17.9 ns/op
BenchmarkUseChan-4    	20000000	        66.2 ns/op
PASS
ok  	_/golang/bench/mutex_chan	3.239s
```

根据上面的压测结果来看, 使用加锁的方式性能是使用channel的方式性能的 **3.6倍** 左右, 这个数值可不低啊. 所以在做服务端开发的时候, 对性能有所疑惑的时候一定要自己做一下benchmark, 不能人云亦云.

reference

[https://toutiao.io/posts/7j995p/preview](https://toutiao.io/posts/7j995p/preview)