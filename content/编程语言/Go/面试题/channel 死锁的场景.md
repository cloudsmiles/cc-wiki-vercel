# channel 死锁的场景

### 

- 当一个`channel`中没有数据，而直接读取时，会发生死锁：

`q := make(chan int,2)
<-q`

解决方案是采用select语句，再default放默认处理方式：

`q := make(chan int,2)
select{
   case val:=<-q:
   default:
         ...

}`

- 当channel数据满了，再尝试写数据会造成死锁：

`q := make(chan int,2)
q<-1
q<-2
q<-3`

解决方法，采用select

`func main() {
	q := make(chan int, 2)
	q <- 1
	q <- 2
	select {
	case q <- 3:
		fmt.Println("ok")
	default:
		fmt.Println("wrong")
	}

}`

- 向一个关闭的channel写数据。

注意：一个已经关闭的channel，只能读数据，不能写数据。