---
title: Go Routine 的异步闭包问题
date:  2023-09-26T17:40:47+08:00
draft: false
tags:
  - go
---

>参考文章[LoopvarExperiment](https://github.com/golang/go/wiki/LoopvarExperiment#what-is-the-problem-this-solves)

从一道经典的js面试题说起
```javascript
for (var i= 0 ; i <3 ;i++){
    setTimeout(()=>{
        console.log(i)
    })
}
// output: 
// 3 
// 3 
// 3
```

由于`var`的作用域与`setTimeout`的异步执行顺序问题.  将闭包函数同步推入异步队列, 异步执回调行时取i值为同步循环后的最终值, 即跳出循环的值`3`

golang (在go version 1.21.1下测试)中也有类似的问题

`for`的声明语句中定义的变量会有值状态丢失的情况, 包括双分号和`range`语法

```go
func main() {
  done := make(chan bool)
  for i := 0; i < 3; i++ {
    go func() {
      fmt.Println(i)
      done <- true
    }()
  }
  // chan blocks the code to reach the end before all the routines are done
  for i := 0; i < 3; i++ {
    <-done
  }
}
// output: 
// 3
// 3
// 3
```
同时借此可以类比拓展出其他类似js同异步打印问题
```go

///...
go func() {
    fmt.Println("second")
    done <- true
}()
fmt.Println("first")
///...
// output: 
// first 
// second

```
推测也是go routine同异步导致的问题, `go`关键字将回调开启并发, 也可以看作成一种异步.

__但是 golang 中 go routine 不是异步队列模型, 所以不会保证输出顺序__

>解决方式也差不多, 本质上指定一个循环体局部变量

```go
for i := 0; i < 3; i++ {
  val := i
  go func() {
    fmt.Println(val)
    done <- true
  }()
  /* alternative 
  go func(i int) {
    fmt.Println(i)
    done <- true
  }(i)
  */
}
// output:
// 2 
// 1 
// 0 // not strictly ordered. can be  2 0 1 ....
```
这个问题在官方wiki上被称作为`loopvar`, 1.21会有一个同名实验特性用来避免这个问题, 并且这个特性拟在1.22 默认开启
```bash
#add env variable when executing go commands
GOEXPERIMENT=loopvar go run main.go
```
输出结果:

![对比](/post/loop_var/compare.png)