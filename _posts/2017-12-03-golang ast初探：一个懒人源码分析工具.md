---
layout: post
title: golang ast初探：一个懒人源码分析工具
date: 2017-12-02 17:30:23 +0800
comments: false
---

### 前言
##### 实现在此
[https://github.com/JodeZer/lazydog](https://github.com/JodeZer/lazydog)
#### 无关
从工作以来，渐渐明白了一个作为软件工程师的哲学：在大多数公司，好的工程师定义不是学术水平、认知、涉猎这样比较务虚的东西，这些属于架构师、CTO的范围；而作为一线工程师，非常简单的好的定义就是解决问题。在linux服务端的开源世界里，所有的问题都能通过阅读源码解决。

#### 有关
golang的源码我认为是我用过的所有编程语言里最容易读的，但还是有一个颇不爽的点就是读接口。golang使用组合interface{}的方式实现复杂功能的抽象。具体概念，在此不表。因为是隐式实现，没有诸如java、c++那样的显式继承。例如：

```go
type I interface{
	Say()
}

type Cat struct {
}

func(this *Cat) Say(){
	println("fucking miao")
}

type Dog struct {
}

func (this *Dog) Say(){
	pringln("fucking dog")
}
```

例子中Dog与Cat的结构实现了I。
在复杂工程中，接口当然不会如此简单，从运行入口开始看，见到的都是更高级、组合多次的高级抽象，生扒代码有时候绕来绕去，还真的一下子不能把握某个接口到底具体的实现是什么。因为我们读源码，不是为了看设计，有时候就是想看某个功能到底是怎么实现的。尤其有时还有运行时确定接口实现的代码，例如：

```go
func GenI(i int) I {
	if i >0 {
		return &Dog{}
	}
	return &Cat{}
}
```

别的不多说，类似这种代码，在mgo里是有的，有时候代码定位到这种接口，心里就有点MMP了...


### 简单粗暴的偷懒梦想

从刚参加工作，面对十万行计C语言的金融业务系统到阅读mongodb、mysql、nginx等著名开源项目代码，我都有一个梦想，运行的时候，输出日志能够告诉我，代码路径到底是什么。这样我就能沿着代码路径直接找到关键点，而不是在代码跳转里转圈子。想想有时候走投无路的打桩调试大法（手动滑稽），其实就是这么一件事。在做C的金融系统的时候，我就想写一个工具，在代码文本的每行代码之间都插入一条日志输出，这样所有的if...else跳到了那里,循环了多少次，递归层数，我都能通过分析日志看出来。
这个梦想在我转做了golang之后终于有了进展，因为golang的依赖简单，有runtime，更重要得是...他有一个parser...

### golang-疯起来连自己都parse的语言
也许有一部分用了很久golang的程序员都不一定知道，golang标准库中自带了golang自身的parser。[具体官方例子在此](https://golang.org/src/go/ast/example_test.go)
功能已经非常完整了，那就是把指定的源码文件，解析成golang标准ast。有了ast结构，给想看的源码插入任意代码就成为了可能。比如：
```go
type I interface{
	Say()
}

type Cat struct {
}

func(this *Cat) Say(){
	println("this is cat say") // 插入的代码
	println("fucking miao")
}

type Dog struct {
}

func (this *Dog) Say(){
	println("this is dog say") // 插入的代码
	pringln("fucking dog")
}

```

这样的代码，说白了就是在Dog.Say与Cat.Say的声明结构里的第一个语句变成一个我想要的语句。

### lazydog所做的事情

1. 备份目标目录下所有的.go源码
2. 在每个目录下生成一个相应package的帮助文件
2. parse源文件的ast，注入代码，生成新文件，本质是在每个函数的第一行插入一个打印日志的调用，调用的是同包下的帮助函数


重编译源代码项目后，输出的日志中就会打印出目前的调用是在哪个文件的哪一行，调用函数名称是什么。
完成后可以用相应的命令恢复源代码。

### 结束

用ast其实可以做很多很强大的代码生成功能，甚至想过是不是可以为此写一种支持泛型的新语言，后端编译成golang的代码。目前这个工具只是业余时间尝试一下。