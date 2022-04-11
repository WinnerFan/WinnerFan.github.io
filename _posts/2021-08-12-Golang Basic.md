---
layout: post
title: Golang Basic
tags: Golang
---
## Golang Basic
### 常量变量
1. 声明常量

```
const LENGTH int = 10 //常量

//iota特殊常量，iota在const关键字出现时将被重置为0
const (
    a = iota   //0
    b          //1
    c          //2
    d = "ha"   //独立值，iota += 1
    e          //"ha"   iota += 1
    f = 100    //iota +=1
    g          //100  iota +=1
    h = iota   //7,恢复计数
    i          //8
)
//iota表示行号
const (
    i=1<<iota //1<<0
    j=3<<iota //3<<1
    k         //3<<2
    l         //3<<3
)
```
2. 声明变量
```
var a, b float64 // 默认值为0, stirng为空字符串
var c, d int = 1, 2
var e, f = true, "run"
g := "run" //之前不能声明，仅在函数体内
var s string // 默认值为""

// 下面默认值为nil
var a *int
var a []int
var a map[string] int
var a chan int
var a func(string) int
var a error // error 是接口
```

- 未使用的局部变量会报错。
- 空白标识符。如值 5 在：`_, b = 5, 7`中被抛弃。
- 当标识符（包括常量、变量、类型、函数名、结构字段等等）以一个大写字母开头（public），一个小写开头包的内部是可见并且可用的（protected）。
- 并行赋值。如果你想要交换两个变量的值，则可以简单地使用`a, b = b, a`，两个变量的类型必须是相同。

### 数据结构
1. 数组和切片

数组，长度和容量定长，**按值传递**。

```
[length]Type
[N]Type{value1, value2, ... , valueN}
[...]Type{value1, value2, ... , valueN}
```
切片，长度可变、容量固定。容量是指从切片的起始开始到其底层数组中的最后的个数，**引用传递**。
```
make([]Type ,length, capacity)
make([]Type, length)
[]Type
[]Type{value1 , value2 , ... , valueN}
```

- 切片底层为隐藏数组，容量与底层数组一致，所以固定。
- 切片与此切片的切片共用一个底层数组，容量会根据切片**起始位置**变化。
- 切片append时，新长度小于等于容量，不会更换底层数组，否则会**新建底层数组**，会影响原始数组。
- copy会新建底层数组，copy(to, from)，以较短的复制完毕为准

遍历方法一致，index必须存在
```
for i, ele := range arr {
    fmt.Print(arr[i]) //只能通过i改变值
}
for i := range arr {
    fmt.Print(arr[i]) 
}
for _, ele:= range arr {
    fmt.Print(ele)
}
```

2. Map

```
kvs := map[string]string{"a": "apple", "b": "banana"} //[key]val
for k, v := range kvs {
    fmt.Printf("%s -> %s\n", k, v)
}
for k := range kvs {
    fmt.Printf("%s\n", k)
}
delete(kvs, "a") //map删除
kvs["a"]="apple" //map增加
```

3. 结构体
```
type Circle struct {
  radius float64
}
func (c Circle) getArea() float64 {
  //c.radius 即为 Circle 类型对象中的属性
  return 3.14 * c.radius * c.radius
}
```

### 逻辑
1. 条件语句

- if
```
if a<20 {}
else if a < 10 {}
else {}
```

- switch，不用break；case里面最后有fallthrough会强制执行下一个case不用判断
```
switch marks {
    case 90: grade = "A"
    case 80: grade = "B"
    case 50,60,70 : grade = "C"
    default: grade = "D"
}
```

- select
	1. case必须为通信操作，并发执行channel被求值
	2. 仅有一个可以通信，则执行；多个可以通信，SELECT随机选取；信道都阻塞，default分支；无default，阻塞
	3. case为nil被忽略
	4. case为超时语句，一段时间内出现可操作case则执行，否则执行超时语句

```
func main() {
    ch := make(chan int)
    go func(c chan int) {
        // 修改时间后,再查看执行结果
        time.Sleep(time.Second * 1)
        ch <- 1
    }(ch)
 
    select {
    case v := <-ch:
        fmt.Print(v)
    case <-time.After(2 * time.Second): // 等待 2s
        fmt.Println("no case ok")
    }
}
```
2. 循环语句
```
for i := 0; i <= 10; i++ {
    sum += i
}

for sum <= 10 {
    sum += sum
}

numbers := [6]int{1, 2, 3, 5}
for i,x:= range numbers {
    fmt.Printf("第 %d 位 x 的值 = %d\n", i,x) //1,2,3,5,0,0
}  
```
### 函数接口
1. 函数

匿名函数，可作为闭包
```
func getSequence() func() int {
   i:=0
   return func() int {
      i+=1
      return i  
   }
}

func main(){
   /* nextNumber 为一个函数，函数 i 为 0 */
   nextNumber := getSequence()  

   /* 调用 nextNumber 函数，i 变量自增 1 并返回 */
   fmt.Println(nextNumber()) // 1
   fmt.Println(nextNumber()) // 2
   fmt.Println(nextNumber()) // 3
}
```

2. 接口

```
type Phone interface {
    call()
}
type NokiaPhone struct {
}
func (nokiaPhone NokiaPhone) call() {
    fmt.Println("I am Nokia, I can call you!")
}
type IPhone struct {
}
func (iPhone IPhone) call() {
    fmt.Println("I am iPhone, I can call you!")
}
func main() {
    var phone Phone

    phone = new(NokiaPhone)
    phone.call()

    phone = new(IPhone)
    phone.call()

}
```

3. 传值

按引用传值，引用类型（slice、map、interface、channel）都默认使用引用传递。
```
var arr = [5]int{1, 2, 3, 4, 5} //创建数组
mod(arr[:]) //切片，引用传递
mod2(arr)   //数组副本，按值传递
mod3(&arr)  //指针
var p *[]int
p = &arr
mod4(p)     //指针
```