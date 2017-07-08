---
layout: post
title: “远离枯燥的golang系列”之这次决定这么使用mgo
date: 2017-06-18 23:19:04 +0800
comments: false
---

### 前言


之前的文章中提到mgo的使用方式是基于session对象的，关于mgo与session，参考旧文[mgo的session与连接池](https://jodezer.github.io/2017/03/mgo%E7%9A%84session%E4%B8%8E%E8%BF%9E%E6%8E%A5%E6%B1%A0)。本文将会探讨我们在生产中是具体如何使用这个库的。不意在确定一种万用的“正确”方法，而是记录一下方便日后批评与自我批评（XD）。

当然，严格来说，这个问题跟golang语言的使用没啥关系；但我就是觉得之前的使用方法很枯燥，好气哦！

### stateless的包方法使用

顾名思义，就是将mongo的各种数据库操作写成一个个方法调用，比如要根据id查找一条文档，那么就有如下的exported package function

```go
package mongo

func FindById(id) (DocStruct, err){
	session := GETSESSION()
	defer session.Close()
	Doc := DocStruct{}
	err := session.DB.C.Find({},&Doc)
	return Doc, err
}

````

使用的时候就是在业务代码中，调用一下这个方法。

```go

func () {
	doc ,err:= mongo.FindById(id)

}
```

这种操作，我自己称之为无状态（stateless）的数据库操作，因为这种操作只是一个动作，他也不会涉及到管理任何的连接或者数据状态。当然这种写法是没什么问题，在操作简单、固定的情况下；吹毛求疵的话，就是session的行为不太好控制。

但是当数据库操作变得复杂，一个结构的行为多样化的时候，就会产生很多这样的方法。比如需要根据id，对某个字段CRUD，那就要产生四个方法，在业务代码中进行组合。还有一些更复杂的针对mongo 嵌套文档的CURD就更是如此了。这样的问题第一是，要产生多个session对象，其实完全没必要，（session获得连接的效率问题在旧文中也提到过）；另一个是操作很多，操作过于松散，总感觉不好维护。有时候甚至会忘记某个操作之前实现过，而写了个差不多得方法满足新的需要。


### 新项目中的尝试

#### 首先上帝说要有对象

在目睹了一个包中，针对无数个文档有无数野生方法的惨状以后，我决定放弃这种exported package function的类C用法。我觉得使用struct抽象出对象模型，每个对象的声明周期内，都会持有一个session，在对象结束时使用session。这样的话，一个文档生命周期内所有的操作，就并不必产生多余的session对象，对mgo的并发问题也更加友好。struct拥有自己的CURD操作，感觉也更规整一些。

具体我是这么做的。

有一个以文档为单位的interface{}

```go

type IDoc interface {
	DB () string
	Col() string
	Close()
}
```

假设有一个mongo collection存储着数据模型为`Person`的数据

```go

type Person struct{
	Id bson.ObjectId `bson:"_id"`
	Name string `bson:"name"`
	Age int `bson:"age"`
}

```

首先规定所有的文档结构定义都要实现`IDoc`的接口，当然其实这个卵用不大，只是希望规范一下模型；具体用处下文细讲。

其实不实现也没问题，`IDoc`不是作为一个操作抽象存在的；但我还是会在全局定义的地方写

```go
var _ IDoc = &Person{}
```

保证实现了方法。

##### 然后上帝说要善待session

定义了一个包装mgo.Session的结构体。（我这次不希望mgo的import在我的项目里到处跑）

```go
type SessionHelper struct {
	*mgo.Session
}

func (s *SessionHelper)CloseHelper() {
	s.Close()
}
```

重新定一下`Person`，让他持有一个SessionHelper
```go

type Person struct{
	SessionHelper `bson:"-"`
	Id bson.ObjectId `bson:"_id"`
	Name string `bson:"name"`
	Age int `bson:"age"`
}

```

之后就可以基于这样的结构，实现任意的CRUD方法，比如：

```go
func (p *Persion) GetName() ()string,error){
	p.DB(p.DB()).Col(p.Col()).Find(bson.M{...}).Select(bson.M{...})
	...
```

对于业务代码来说，复杂的sql在method receiver上去现实，外部不关心的操作可以匿名；对session非常友好。

当然，doc还需要New方法

```go
func NewPerson() *Person{
	return &Person{	
		SessionHelper:GETSESSION(),
	}
}
```

使用时候的画风就是

```go
func foo() {
	p := NewPerson()
	defer p.Close()
	p.LoadById(id)
	p.GetName()
	...
	p.DoXXX()

}
```

反正我觉得比如下这种nice很多。包方法毕竟是共享名字的，有时候不看代码也搞不太清一个方法到底是要操作哪个数据结构。

```go

func foo() {
     p := &Person{}
     p.Name = mongo.FindName(id)
     p.xxx = mongo.Findxxx()
    ...
}
```
#### 一个缺点

大概一个缺点就是，因为集成了session，每个对象在生成以后必须手动Close，否则会造成mgo的连接泄露。其实这跟每次生成session的方法是一样的，也是要成对生成释放的，也没差。

### 爬坑时间

#### golang的作用域

事情大概是这样的，有一个大文档，里面引用了某些小文档的id，（我采用的方式是手动引用，mongo其实提供了一种[reference方法](https://docs.mongodb.com/manual/reference/database-references/)）我觉得没差就没用。

引用小对象的话，CURD之前先要确定这些小对象存不存在，那就有了如下的过程

```go
func (d *Doc) Foo() {
	obj1 := NewObj1()
	defer obj1.Close()
	obj1.Load(id)
	obj1.Exists()
	
	obj2 := NewObj1()
	defer obj2.Close()
	obj2.Load(id)
	obj2.Exists()
	
}
```
这当然也没什么问题，但`obj1`,`obj2`的声明周期过于长了，在这个过程中，会有三条mongo的连接被使用到，其中两个释放要等到整个函数结束。显然太浪费了。

于是我一抠脚，写出了如下的代码：

```go

func (d *Doc) Foo() {
	{
	obj1 := NewObj1()
	defer obj1.Close()
	obj1.Load(id)
	obj1.Exists()
	}
	
	{
	obj2 := NewObj1()
	defer obj2.Close()
	obj2.Load(id)
	obj2.Exists()
	}
	
}
```
十秒钟以后，我意识到这是没有任何意义的；因为goalng的defer是注册在函数栈上的，跟作用域没有半毛钱关系，这两个defer照样是在函数返回的时候执行的。

其实到这里，封装成两个函数，缩小一下函数栈就可以达到控制生命周期释放连接的目的；但是老师，我不服，我有看上去更酷炫的写法。

十秒钟以后，这个函数变成了这样

```go

func (d *Doc) Foo() {
	{
	obj1 := NewObj1()
	runtime.SetFinalizer(obj1,func(obj interface{}){
		obj.Close()
	}) obj1.Close()
	obj1.Load(id)
	obj1.Exists()
	}
	
	{
	obj2 := NewObj1()
	runtime.SetFinalizer(obj1,func(obj interface{}){
		obj.Close()
	}
	obj2.Load(id)
	obj2.Exists()
	}
	
}
```

这样就看上去给力多了是不是！`runtime.SetFinalizer`会在运行时发现注册对象失去引用的时候调用回调方法。

但是！现实是很残酷的，这段代码其实完全不给力，因为运行时发现对象失去引用是需要一段时间的，这段时间甚至比`defer`版本更久~

那老师！就没有更给力的了吗。当然有！

```go
func (d *Doc) Foo() {
	{
	obj1 := NewObj1()
	runtime.SetFinalizer(obj1,func(obj interface{}){
		obj.Close()
	}) obj1.Close()
	obj1.Load(id)
	obj1.Exists()
	}
	runtime.GC()
	
	{
	obj2 := NewObj1()
	runtime.SetFinalizer(obj1,func(obj interface{}){
		obj.Close()
	}
	obj2.Load(id)
	obj2.Exists()
	}
	
	runtime.GC()
	
}
```

每当我想要弄死的对象出了作用域，就调用一次强制GC，连接就会毫不犹豫立马释放。


不过最后，我还是为这个Doc实现了小函数来判断引用是否存在；因为这个世界上存在一种行为叫code review，写这种代码会被搞事情的！（逃）

##### bson.Unmarshal的置空行为

使用这种对象模型，在查询上面很容易地想到就是用自己的引用来反序列化

比如

```go
func (p *Persion)Load(id) {
	p.SessionHelper.Find(...).One(p)
	 
}

```

这个函数的预期就是把所有字段查询出来，再从bson格式反序列化到p的各个字段。但是，这个mgo下的bson实现有一个我现在还不太理解的动作，会将传入反序列化的实例置为空，而且是无视tag的粗暴的置为空。

也就是说，假设我的结构里面有一些跟数据库字段无关的`bson:"-"`的辅助字段，用于保存一些计算的结果之类的。

例如：

```go

type Person struct {
	...
	Field string `bson:"-"`
}
```

mgo这个bson老哥会把我的`Field`字段置为空。那好气哦。
内部调用的方法就是`bson.Unmarshal`，具体里面的实现这里就不多说了，代码很好找；这么做原因不懂，提了issue也没人理（好气QAQ）。
[没人理的issue](https://github.com/go-mgo/mgo/issues/452)

### 结语

希望mongo官方早日能出自己的驱动！