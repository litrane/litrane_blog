---
title: golangStart
subtitle:
date: 2023-08-26T11:28:25+08:00
draft: false
author:
  name:
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - draft
categories:
  - draft
hiddenFromHomePage: false
hiddenFromSearch: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: true
  url:
---
## if-else 语句

注意点：

1. }与 else 是同一行的，因为不是同一行的话编译器会在}后面加上；
2. 可以使用 if statement;condition{}进行编写，但 statement 初始化的是临时变量

## for 循环

语法

```go
for initialisation; condition; post {  
}
```

注意点：

1. 在 for 循环中声明的变量只能在循环体内访问，因此 i 不能够在循环体外访问。
2. 分号也可以省略

break

continue

无限循环

```go
for {  
}
```

## switch 语句

```go
package main

import (
    "fmt"
)

func main() {
    letter := "i"
    switch letter {
    case "a", "e", "i", "o", "u": // 一个选项多个表达式
        fmt.Println("vowel")
    default:
        fmt.Println("not a vowel")
    }
}
```

**fall through**

```go
func main() {

    switch num := number(); { // num is not a constant
    case num < 50:
        fmt.Printf("%d is lesser than 50\n", num)
        fallthrough
    case num < 100:
        fmt.Printf("%d is lesser than 100\n", num)
        fallthrough
    case num < 200:
        fmt.Printf("%d is lesser than 200", num)
    }

}
```

## 数组与切片

### 数组初始化

var a [3]int vs [4]int

a := [3]int {12,34}

[...]int

len()

```go
a := [3][2]string{
        {"lion", "tiger"},
        {"cat", "dog"},
        {"pigeon", "peacock"}, // this comma is necessary. The compiler will complain if you omit this comma
    }
```

### 数组的遍历

i,v=range a

```go
func printarray(a [3][2]string) {
    for _, v1 := range a {
        for _, v2 := range v1 {
            fmt.Printf("%s ", v2)
        }
        fmt.Printf("\n")
    }
}

```

其是值类型

### 多维数组

```
// Step 1: 创建数组
    values := [][]int{}

    // Step 2: 使用 append() 函数向空的二维数组添加两行一维数组
    row1 := []int{1, 2, 3}
    row2 := []int{4, 5, 6}
    values = append(values, row1)
    values = append(values, row2)

a := [3][4]int{  
 {0, 1, 2, 3} ,   /*  第一行索引为 0 */
 {4, 5, 6, 7} ,   /*  第二行索引为 1 */
 {8, 9, 10, 11},   /* 第三行索引为 2 */
}
```

切片

```go
c:==[]int{7,8}
```

其有 len 和 cap 两种属性，len 是获取该切片的长度，cap 则是获取其第一个元素到其映射的底层数组的最后一个元素的容量。

对切片的修改也会反映在数组中

```go
func make（[]T，len，cap）
```

```go
func main() {
    cars := []string{"Ferrari", "Honda", "Ford"}
    fmt.Println("cars:", cars, "has old length", len(cars), "and capacity", cap(cars)) // capacity of cars is 3
    cars = append(cars, "Toyota")
    fmt.Println("cars:", cars, "has new length", len(cars), "and capacity", cap(cars)) // capacity of cars is doubled to 6
}
```

names==nil(零值)

```go
func main() {
    veggies := []string{"potatoes", "tomatoes", "brinjal"}
    fruits := []string{"oranges", "apples"}
    food := append(veggies, fruits...)
    fmt.Println("food:",food)
}
func main() {  
    var names []string //zero value of a slice is nil
    if names == nil {
        fmt.Println("slice is nil going to append")
        names = append(names, "John", "Sebastian", "Vinay")
        fmt.Println("names contents:",names)
    }
}

```

因此，当切片作为参数传递给函数时，函数内所做的更改也会在函数外可见。

一种解决方法是使用 [copy](https://golang.org/pkg/builtin/#copy) 函数 `func copy(dst，src[]T)int` 来生成一个切片的副本。这样我们可以使用新的切片，原始数组可以被垃圾回收。

数组与切片的区别在于切片由以下方法产生

1. 通过数组创建一个切片
2. 直接声明一个切片 a :=[]int {} vs 数组的 a:=[...]int{}
3. 通过 make
4. 切片生切片

切片删除

```go
//我们创建了一个切片，有四个元素，下标命名为0，1，2，3
slice11 := []int{1,2,3,4} 
//假设我们要删除下标为0的元素，这段代码的含义是
创建一个空的切片 slice11[:0]
创建一个从下标1到末尾元素的切片 slice11[1:]
给空切片添加元素并返回一个新的切片
slice12 := append(slice11[:0],slice11[1:]...)
//同上我们得出删除下标为i的元素的切片的公式时
sliceTemp := append(slice11[:i],slice[i+1:]...)
```


