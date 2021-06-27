---
title: go学习笔记(三)——语法
date: 2020-02-13 20:10:30
tags: [go]
categories: go
---


![基础语法.jpg](http://image.stxl117.top/基础语法.jpg)

<!-- more -->
![go预留关键字.png](http://image.stxl117.top/go关键字.png)

## 1.变量
变量是每种语言都不可或缺的声明方式，go提供了以下几种声明方式
```go
var a string
var a string = "a"
var a = "b"
a := "c" //只能在函数内使用
var a,b,c string //同类型多变量的声明
```
>需要注意的是，在go语言中声明的变量如果在接下来作用域中不使用，编译时会报**xxx declared and not used**错

除了上述几种声明，go语言还可用分组声明的方式
```go
  var(
   a int =1
   b string = "a"
  )
```
同时  go中还有一个特殊的变量，匿名变量(_)在表示接收但不会使用的变量
```go
    dictValue := map(string)int{"test":1}
    value,_ := dictValue["test"]
```
当然除了上面的使用方式，匿名变量还有其他的作用，在稍后会做解释。

## 2.常量
常量可以是字符、字符串、布尔值或数值。不支持 推断赋值（:=）
```go
  const constValue int =1
```

```go
package main

import "fmt"

const(
	black = 0
	red = iota
	_
	blue
	green =4
	organge,pink =iota ,iota
)

func main(){
	fmt.Println("black value: ",black)
	fmt.Println("red value: ",red)
	fmt.Println("blue value: ",blue)
	fmt.Println("green value: ",green)
	fmt.Println("organge value: ",organge)
	fmt.Println("pink value: ",pink)
}
```
![输出结果](https://upload-images.jianshu.io/upload_images/7504708-0f889ad2474c6f6e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上面的代码中，可以看到常量也是支持分组定义和匿名变量的。还有一个很陌生的itoa。
iota代表了**const声明块的行索引（下标从0开始）**，还有就是我们看到_和blue并没有赋值，输出的时候结果blue输出的3，这里是const的特性**第一个常量必须指定一个表达式，后续的常量如果没有表达式，则和上一行的表达式相同**。

## 3.指针
除了go语言没有提供指针运算，其余和C语言中的指针一致——保存了值的内存地址。
```go
    var p *int  //零值为nil

    i := 42
    p = &i
```

>1.`&` 操作符会生成一个指向其操作数的指针;`*`操作符表示指针指向的底层值。
>2.由于go中函数都是值传递，所以经常使用指针传递的方式在函数中修改原有对象信息



## 4.流程控制

### a.条件语句
条件语句主要有`if...else...`和`switch`。
`if...else...`语句和其他语言的没有什么区别，go中增加在if语句定义变量，该变量的作用域为`if...else`内

`switch`语句，与其他语言最大的差异是，只会执行第一个命中`case`条件的程序段，不需要额外的`braek`跳出。
```go
   package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Print("Go runs on ")
	switch os := runtime.GOOS; os {
	case "windows":
		fmt.Println("windows10.")
		//fallthrough
	case "darwin":
		fmt.Println("OS X.")
	case "linux":
		fmt.Println("Linux.")
	default:
		fmt.Printf("%s.\n", os)
	}
}
```

> **fallthrough**关键字可以使命中后的case继续执行

### b.循环
go中只有一种循环的写法`for` 语句，基本语法同java和python。
```go
    for <初始化语句>;<条件语句>;<后置语句>{
   }
```

> `break`和`continue`效果与其他语言一样，不做介绍

> e.g. [for循环例子](https://github.com/skystmm/learn-go/blob/master/exercise/loopandfunc/exercise-loops-and-func.go)

## 5.包
go程序是由多个包构成的，其中每个可执行的go程序都是从main包开始的，main保重还需要有程序入口main函数，相当于java中的 ` public static void main(String [] args) `方法和python中的` if __name__ == "main " `函数，关于go中的main函数会在函数中介绍。

go程序中引入包使用` import `关键字，和python，java中用法类似；同时go中import依然支持分组的写法
``` go
    package main
    import (
       _  "strings"
       "fmt"
       . "os"
       http "net/http"
   )
```
上面的代码中对于包的倒入有了几种不同方式，使用匿名(_)接包名导入的方式，表示导入该包，但是不可使用其导出方法，但会执行该包中的init方法；点操作是一种便捷方式，使用点操作导入的包，在使用其导出方法的的时候，可以不使用包名，直接使用导出的函数进行调用；最后` import http "net/http" `是给引入包起别名的一种方式，类似Python中的` import xxx as xx `。

> 注意：
1.大写字母开头的变量和函数是可导出的，即其他包可以读取，是公用变量，公有函数；
2.小写字母开头的不可导出，是私有变量，私有函数。

## 6.函数
函数同java中的方法，python中的函数，声明如下所示
```go
   func <func_name>(param pramType ,...)(result reusltType,...){
      //函数内部处理
     // 有返回值时
     return xxx,xxx
  }
```
与java不同的是，go中的函数可以有多个返回值，同时生命时返回值在入参的后边（有点像scala的语法）。

> 几个特殊的函数
main函数：go程序的入口函数，每个go包只能有1个或0个main函数
init函数：包导入的时候的初始化操作函数，有点像java中的构造函数
panic和revocer函数：go中的异常处理函数，类似 throw 和catch机制

### defer关键字
在函数中可以使用defer关键字帮我们处理一些资源关闭和return前处理，defer的功能很像java中` finally`的处理，不同之处在于defer像一个栈一样，当程序即将运行完当前函数作用域时，会更具defer声明顺序的逆序一直执行defer的内容。
```go
package main

import . "fmt"

func main(){
	Println("first")
    defer Println("six")
	Println("second")
    defer Println("five")
	Println("third")
	defer Println("four")
}
```
![执行结果](https://upload-images.jianshu.io/upload_images/7504708-6419a0a64784ee09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 函数闭包
闭包是一个函数值，它引用了其函数体之外的变量，该函数可以访问并赋予其引用的变量的值。
```go
package main

import "fmt"

func adder() func(int) int {
	sum := 0
	return func(x int) int {
		sum += x
		return sum
	}
}

func main() {
	pos, neg := adder(), adder()
	for i := 0; i < 10; i++ {
		fmt.Println(
			pos(i),
			neg(-2*i),
		)
	}
}
```
## 7.结构体

上次听到结构体这个名词还是在大一学c语言的时候，之后这个词的出镜率基本为0。go中的结构体声明和c异曲同工。
```go
type <structName> struct {
	<propertyName> <propertyType>
	....
}
```
在go结构体重有一种特殊的写法，如下代码所示
```go
package main

import "fmt"

type Human struct {
	name string
	sex  bool
	age  int
}

type Employee struct {
	Human
	salary float32
	deaprt string
	age    int
}

func main() {
	employee := Employee{Human: Human{name: "Sky", sex: true}, salary: 100.00}
	employee.age = 30
	employee.Human.age = 30
	employee.deaprt = "go learn"
	fmt.Println(employee)

	employee.name = "Test"
	fmt.Println(employee)
}
```
![执行结果](https://upload-images.jianshu.io/upload_images/7504708-03dad36379f6c116.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面代码中，声明了Human和Employee两个结构体，在Employee结构体通过匿名的方式引入了Human，这里可以理解成一种继承，当我们初始化Employee的变量employee 时，employee会拥有Human的所有属性；但是当父亲结构体和子结构体拥有相同的属性时，子结构体会覆盖服结构体的属性；如果需要修改父中的同名属性，如main函数第3行所示。**go中可以通过结构体声明匿名的结构体属性，来实现继承。**

## 8.方法（method）
方法就是一类带特殊的*接收者*参数的**函数**。

```go
package main

import "fmt"

type Human struct {
	name string
	sex  bool
	age  int
}

func (human Human) CheckSex() string {
	if human.sex == true {
		return "男"
	} else {
		return "女"
	}
}

func main() {
	boy := Human{"Tom", true, 30}
	girl := Human{"Luck", false, 18}

	fmt.Println("Tom sex :", boy.CheckSex())
	fmt.Println("Luck sex :", girl.CheckSex())
}
```
## 9.interface
接口是由一组方法签名定义的集合。
```go
type <interfaceName> interface {
	<funcNmae>( params)(results)
}
```
接口也是值。它们可以像其它值一样传递。接口值可以用作函数的参数或返回值,保存了一个具体底层类型的具体值。

接口值调用方法时会执行其底层类型的同名方法。



> [文中代码实例](https://github.com/skystmm/learn-go)
