---
title: go学习笔记(二)——基本数据类型
date: 2020-02-11 15:37:55
tags: [go]
categories: go
---


![Go语言.jpg](http://image.stxl117.top/数据类型.jpg)
<!--more -->
# 数据类型

## 1.基本类型

go语言中的基本类型如上图所示，go语言中有五类基数类型。

### 1） boolean型
同其他语言一样，boolean包括两个值 true 和false
```go
  var booleanVal boolean = true
```

### 2)  数值类型
数值类型中可以分为三大类：整型，浮点型和复数

#### a.整型
go语言中直接定义好了不同位数的整型，其中包括如下表所示类型
| 类型 |  备注 |
| ------ |------ |
| rune | int32的别称 |
| int8 | 8位带符号整数类型 |
| int16 | 16位带符号整数类型 |
| int32 | 32位带符号整数类型 |
| int64| 64位带符号整数类型 |
| byte | 相当于uint8别称 |
| uint8 | 8位不带符号整数类型 |
| uint16 | 16位不带符号整数类型 |
| uint32 | 32位不带符号整数类型 |
| uint64| 64位不带符号整数类型 |

> 注意：
1.int的默认类型为int32
2.go语言中不支持隐式类型转换，哪怕是int8转int32，使用隐式类型转换会在编译器报错
```go
var a int32 = 123
//编译报错cannot use a (type int32) as type int8 in assignment
var b int8 = a
```
#### b.浮点类型
浮点类型包括float32和float64两个，如果不显示声明为32位，则float默认为64位
```go
    var floatVal float64 =43.32
```
#### c.复数类型
复数类型是go语言增加支持的数据类型，复数类型就是我们在高中数学中学过的复数，表示方法也和上学时学的表述方法一致。go语言依然为了兼容32位系统和64位系统，提供了两种复数基本类型complex128(64位实数和64位虚数,默认)和complex64(32位实数和32位虚数).

```go
var complexVal complex128 = 58+8i
```
### 3) 字符串
字符串类型的声明有两种方式双引号("")和反引号(``)，两者的区别在于单引号可跨行，所见即所得，类似python中的三引号表示。

```go
   var s string = "a"
   var ss string =`a
   b`
  fmt.Println(s)
  fmt.Println(ss)
```

>基础类型的零值：
数值类型为 0，
布尔类型为 false，
字符串为 ""（空字符串）。
## 其他类型

### 1. 数组
同其他语言差不多，区别在于声明是需要声明容量，声明后不可扩缩容
```go
  var array [10]int32
  //多维数组
  b := [2][4]int{[4]int{1,2,3,4},[4]int{2,3,4,5}}
```
> 数组声明中需要注意的是，数组容量的声明需要写在类型的前边，否则会报错。不像java中即可以写在前边也可以写在后边
```go
  //编译错误
   var array int[32]
```
### 2. 切片(slice)
刚看到slice的时候，第一反应是python中的slice特性，两者有一些相似之处，也有不同之处。切片为数组元素提供动态大小的、灵活的视角,设计思想上python和go对于切片是一样的。区别在于python中slice更加灵活一些支持三个参数的切片处理（起止以及步长处理），而go中暂时支持起止处理。
```go
//python实现
li = [1,2,3,4,5]
r = li[1:5:2]  // [2,4]
//go实现
a := [5]int{1,2,3,4,5}
b := a[1:3] //[2,3,4]
```
go中的切片可以理解为
```go
 struct{
	var a [N]int
        //start end 为前必后开区间，同python
	var start int
	var end int
}

```
这里需要注意的是如下情况
```go
	var a = [6]int{1, 2, 3, 4, 5, 6}
	b := a[0:]
	c := a[0:]

	b[5] = 1
	fmt.Println(c[5]) //6

	b = append(b, 7)
	b[1] = 9
	fmt.Println(c[1]) //2
```
从上面代码的输出可以看到，在不发生扩容情况下，两个切片b和c底层都指向了数组a（地址引用），当我们修改切片b的index为5的数值后，切片c对应index结果也会改变。就像go指南中描述的一样，切片并不存储任何数据，它只是描述了底层数组中的一段。但是当我们向容量已满的切片b追加元素后，会触发扩容，扩容大致如下图，导致切片b的底层数组成了有变新的数组，所以在改变切片b的内数据后，切片c中数据不会发生改变。

![切片扩容](http://image.stxl117.top/slice扩容.jpg)

>这里还需要注意的是，切片触发扩容规则如下：
>1.切片每次新增个数不超过原来的1倍，且每次增加数不超过1024个，且增加后总长度小于1024个，这种情况下扩容后为原来的2倍
>2.切片一次新增个数超过原来1倍，但不超过1024个，且增加后总长度小于1024个，这种情况下扩容后比实际具有的总长度还要大1
>3.原切片长度超过1024时，一次增加容量不是2倍而是0.25倍，每次超过预定的都是0.25累乘,如果增加数量大于0.25增量，就是增加量+1

### 3.字典
采用哈希方式存储的数据类型，同java中的map,python中的字典。区别在于我们获取字典内数据值时，返回值有一些差异，如下代码
```go
    dictDemo := map[string]int {"test":1}
    //value为key对应的值，contain为boolean型，true为dictDemo包含key为test的键，false为不包含
    value,contain= dict["test"]
```


>参考内容：
>1.[go语法指南](https://tour.go-zh.org/)
>2. 《go web编程》