# Go简介

## 优点

- 简单的部署方式
  - 直接编译成机器码（编译速度快，启动运行时间快）
  - 不依赖其他库
  - 直接运行即可部署
    - java需要依赖其他的包，或者java的jdk
- 静态类型的语言
  - 编译的时候能给检查出来隐藏的大多数问题（如果是动态语言只能运行时才能判断有无异常）
- 语言层面的并发
  - go语言原生支持（使用协程）
  - 充分的利用多核
- 标准库丰富
  - runtime系统调度机制
  - 高效的GC垃圾回收
  - 丰富的标准库
- 简单易学
  - 只有25个关键字
  - C语言简洁，内部实际使用C语言实现
  - 面相对象继承（继承、多态、封装）
  - 跨平台支持

docker 和k8s 都使用go语言编写

## 缺点

- 包管理，大部分在github上，不安全，不稳定
- 无泛化类型
- 所有Exception 都用Error 来处理
- 对C的降级处理，并非无缝，没有C降级到asm那么完美（序列化问题）



# Go语言新奇



## hello world

- package main  只有main包才会引入该main

- golang的表达式中，加`;`和不加 都可以，建议不加

- 引包方式可合并

  - 原本引包 import "fmt"   import "time" 可改为

    ```go
    import (
    	"fmt"
    	"time"
    )
    ```

- 函数的{  必须与函数名在同一行，否则编译错误





## 变量

### 变量声明

**若声明了变量，未使用会抛出异常**



- 声明一个变量 默认值是0、可以初始化值、可以不指定变量类型，由值自动匹配当前变量的数据类型、可以省去var关键字字节自动匹配

  - ```go
    var a int
    var b int = 100
    var c = 200
    d := 300
    ```

- 声明全局变量只能使用方法一，二，三 不能省略 var，即 `:=`只能在方法体上声明

### 多变量声明

- 单行声明多变量

  - ```go
    var xx, yy int = 100,200
    var kk, ll = 100, "abc"
    ```

- 多行声明多变量

  - ```go
    var {
      xx int = 100
      yy int = 200
    }
    ```

    



## 常量

- 单个常量定义

```go
const length int = 10
```

- 多个常量定义 itoa  可以批量定义常量值

```go
const (
		a = iota    // itoa 默认每行递增   
  	b
  	c
  	d
)
// a = 0,b = 1, c = 2,d = 3
const (
		a,b = itoa+1,itoa*2    // itoa = 0   a = itoa+1, b = itoa * 2  a = 1 b = 0
  	c,d										 // itoa = 1   c = itoa+1, d = itoa * 2  c = 2 b = 2
  	e,f										 // itoa = 2   e = itoa+1, f = itoa * 2  e = 3 b = 4
)
```



## 函数

### 多返回值



- 支持多返回值

```go
func method(a string, b int) int {
  return 100
}

func method(a string, b int) (int, int) {
  return 100,200
}
```

- 多返回值支持设置变量名，**return 不必返回该变量, 但返回值会有默认值0**

```go
func method(a string, b int) (r1 int,r2 int) {
  r1,r2 = 100,200
  return
}
```



### init函数

每个包可以有init函数，运行main方法时

1. 运行main方法时，先引包 package1，如果有package1有引用的话，会再次引包
2. 引其他包完成后，先执行如执行当前包的init() 函数，才完成当前引包

![image-20250422082724460](Go.assets/image-20250422082724460.png)



### 匿名导包

```go
import (
 	 	_ "pack1"  // go不支持无效导包（即引包但是没调用该包内方法），添加下划线就可允许无调用的引包，方便执行初始化
  	p1 "pack2" // 可以对导的包设置别名，调用该包内的函数时，通过 p1.func即可
  	. "pack3"  // 可以直接调用pack3的函数，无需 pack3.func 这种，而是 func。容易与其他包混淆，不建议使用
)
```





## 指针

**go语言默认是值传递**，可以在函数入参 设置未指针传入  *p 是修改p地址的内容 &a 传入的是地址而非值

```go
func changeValue(p *int) {
  *p = 10
}

func main() {
  var a int = 1
  changeValue(&a)
}
```





## defer

结束操作需要执行的关键字， defer 命令行

defer对应的Stack栈方式，defer x1 defer x2 结束操作时，先执行x2 后执行x1



## slice和map

### 数组

函数入参是数组，可指定数组长度，是值拷贝

```go
var array1 [10]int
var array2 [10]int := {1,2,3,4}
var array3 [4]int := {1,2,3,4}


func method( array [4]int) {
  // 只能传长度为4的数组，且是值传递，main函数创建的数组，调用该函数时，是传入的数组副本，修改数组内容不会影响main的数组，不建议使用
}
```

建议使用**slice** 切片，动态数组

```go
func method( array []int) {
  	// 不指定数组长度，就是地址传递，而非值传递
  for index, value := range array {
    
  }
  // 不关注 index 可以使用下划线 设置匿名变量
  for _, value := range array {
    
  }
}
```



- ### slice

  - 声明

  - ```go
    // 声明slice1 是切片，长度为3 默认值是1,2,3的数组
    slice1 := []int{1,2,3}
    
    // 声明slice2 是切片，长度为3 开辟长度为3的内存空间
    var slice2 []int   // 此时 slice2 = nil
    slice2 = make([]int, 3)
    
    // slice2 的另一种写法
    var slice3  []int = make([]int, 3)
    
    // 简略写法
    slice4 := make([]int, 3)
    ```

  - 切片的追加

    - 切片的长度和容量不同，长度表示实际使用数量，容量表示当前数组能够容纳容量

    - 切片的扩容机制，append，若数组容量超出容量，则将容量增加为2倍

      ```go
      slice1 := make([]int, 3)
      slice1.append(1) // 数组长度为4，容量为6
      ```

  - 切片的截取

    - 可以对原数组进行截取，但是实际指向的数组都是一个数组地址，即修改截取的数组内容，则两个数组内容都会改

    - ```go
      s := []int{1,2,3}
      
      s1 := s[0:2]
      s2 := s[:2]
      s3 := s[1:]
      ```

      