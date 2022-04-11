---
layout: post
title: Golang Advanced
tags: Golang
---

## Golang Advanced
### 常用结构
1. chan
缓冲区为环形队列，一个管道仅允许同时被一个协程读写

- 读缓冲区，缓冲区为空或没有缓冲区，协程阻塞，加入recvq队列
- 写缓冲区，缓冲区为满或没有缓冲区，协程阻塞，加入sendq队列
- 关闭管道，recvq唤醒返回零值，sendq唤醒panic
- nil管道读写永久阻塞
- select case读管道，读不到数据协程不会阻塞，不会加入队列
- for-range读管道，会阻塞

2. slice
切片与原数组或切片共享底层空间

- make创建，添加新元素不必重新分配内存
- 数组创建，切片头到数组尾为capacity
- 切片创建，可以超过len到达cap
- 作用于字符串将产生字符串，而不是切片
- 扩容会copy入新的空间
- copy不会扩容，copy数量为两个切片中最小值

3. map
一个bucket可以存放8个kv，负载因子=键数/bucket数=6.5，触发rehash。Hash值低位相同的放在同一个bucket里面，每个bucket存放8个Hash值高8位的数组，用于匹配。

4. struct
接收者
```
func (s Student) SetName(name string){} //作用于Student的copy对象
func (s *Student) SetName(name string){} //作用于Student原对象
```

5. string
单行拼接多个内存只会分配一次，反单引号中不用转义。默认UTF-8编码，长度为字节数。字符串不能修改。sting互转byte[]会内存copy

### 控制结构
1. select
- 管道没有缓冲区（运行时被忽略），无default，读写都不执行，阻塞
- 随机选择可执行case执行
- 无可执行case，执行default，否则循环case，阻塞

2. for-range
- 只有一个变量返回下标
- 作用于string，元素下标可能不连续(Unicode编码)
- 作用于数组、切片，循环次数开始时已经确定，map不要在range里修改
- 作用于channel，没有元素会阻塞，关闭则退出，nil channl永久阻塞

### 并发控制
1. channel
```
channels := make([]chan int, 2)
channels[0] = make(chan int)
go DoSth(channels[0])
go DoSth(channels[1])

for i, ch := range channels {
    <- ch //不返回会阻塞住
}
```

2. WaitGroup
```
func main(){
    var wg sycn.WaitGroup
    wg.Add(1)
    go DoSth(&wg) //这里取地址
    wg.Wait()
}
func DoSth(wg *sync.WaitGroup){
    wg.Done() //wg.Add(-1)
}
```

3. context
```
ctx, cancel := context.WithCancel(context.Background()) //参数传递ctx即可
cancel() //触发协程中<- ctx.Done()

ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
cancel() //5s后或主动触发协程中<- ctx.Done(), ctx.Err()打印信息不同

ctx := context.WithValue(context.Background(), "key", "val") //无法cancel
// ctx.Value("key")会一直向父节点查找
```

4. mutex
- 新进入会自旋，但阻塞1ms后会变为饥饿，饥饿优先于自旋
- RWMutex：RLock、RUnlock、Lock、Unlock，前两个读不会阻塞读

### 异常处理
1. error
```
err:= errors.New("new error") // 性能好
valErr:=fmt.Errorf("error: %v", err) //生成err包的errorString
wrapErr:=fmt.Errorf("error: %w", err) //生成fmt包的wrapError，内容同上，%w不能多于一个
errors.Unwrap(err) //nil
if errors.Is(wrapErr, targetErr){} //相当于循环errors.Unwrap(err)
if errors.As(wrapErr, targetErr){} //循环判断并转换
```

2. defer
用于关闭文件描述符、释放锁等，先入后出，return、panic触发

```
func test() {
    i: = 0
    aArray := [3]int{1,2,3}
    defer fmt.Println(i) //0
    defer func(x int) {fmt.Println(x)}(i) //0
    defer func(x *int){fmt.Println(*x)}(&i) //1
    defer func() {fmt.Println(i)}() //1
    defer func(array *[3]int){
        for i := range array {
            fmt.Print(array[i]) // 10 2 3
        }
    }(&aArray)
    defer func(){
        for i := range aArray {
            fmt.Print(aArray[i]) // 10 2 3
        }
    }
    i ++
    aArray[0]=10
    return
}

func deferTest() (result int) {
    i := 1
    defer func() {
        result ++
    }()
    return i // result = i; result++; return result
}

func deferTest() (result int) {
    defer func() {
        result ++
    }()
    return 0 // result = 0; result++; return result
}

func deferTest() int {
    i := 1
    defer func() {
        i ++
    }()
    return i // anony = i; i++; return anony
}

func()
```

3. panic

- 递归执行**当前协程的所有**defer，recover后，也会执行defer
- 针对于嵌套panic，程序立即终止当前defer函数，继续panic流程
- recover会吃掉panic，一个panic只能被吃掉一次（**defer会继续**），返回值为panic中的字符串，必须直接位于defer函数体中（不能再嵌套func）