# Go中GMP有哪些状态？

### **Go中GMP有哪些状态？**

![https://pic4.zhimg.com/80/v2-87beb4a53dd92ddccef4ecb486dfa213_720w.webp](https://pic4.zhimg.com/80/v2-87beb4a53dd92ddccef4ecb486dfa213_720w.webp)

G的状态：

**_Gidle**：刚刚被分配并且还没有被初始化，值为0，为创建goroutine后的默认值

**_Grunnable**： 没有执行代码，没有栈的所有权，存储在运行队列中，可能在某个P的本地队列或全局队列中(如上图)。

**_Grunning**： 正在执行代码的goroutine，拥有栈的所有权(如上图)。

**_Gsyscall**：正在执行系统调用，拥有栈的所有权，与P脱离，但是与某个M绑定，会在调用结束后被分配到运行队列(如上图)。

**_Gwaiting**：被阻塞的goroutine，阻塞在某个channel的发送或者接收队列(如上图)。

**_Gdead**： 当前goroutine未被使用，没有执行代码，可能有分配的栈，分布在空闲列表gFree，可能是一个刚刚初始化的goroutine，也可能是执行了goexit退出的goroutine(如上图)。

**_Gcopystac**：栈正在被拷贝，没有执行代码，不在运行队列上，执行权在

**_Gscan** ： GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在。

P的状态：

**_Pidle** ：处理器没有运行用户代码或者调度器，被空闲队列或者改变其状态的结构持有，运行队列为空

**_Prunning** ：被线程 M 持有，并且正在执行用户代码或者调度器(如上图)

**_Psyscall**：没有执行用户代码，当前线程陷入系统调用(如上图)

**_Pgcstop** ：被线程 M 持有，当前处理器由于垃圾回收被停止

**_Pdead** ：当前处理器已经不被使用

M的状态：

**自旋线程**：处于运行状态但是没有可执行goroutine的线程，数量最多为GOMAXPROC，若是数量大于GOMAXPROC就会进入休眠。

**非自旋线程**：处于运行状态有可执行goroutine的线程。