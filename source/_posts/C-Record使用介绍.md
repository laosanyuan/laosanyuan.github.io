---
title: C# Record使用介绍
date: 2021-12-14 23:10:00
tags:
  - c#
  - .Net
---

`record`是C#9.0之后的版本新引入的一种引用类型——记录。一般而言，我们常用到class和struct两种数据类型，两种类型各有优势，record可以说是各取二者所长的一种新的形式。

## class、struct与record的区别

### class

class是我们日常开发中最常用的类型，属于引用类型，可以继承、支持多态，数据存放于堆上，属于引用类型。在进行两个对象的==比较时，默认比较的是对象的引用地址是否相等。

### struct

struct不支持继承和多态，但可以实现接口。struct存放于栈上，属于值类型，故常用于简单类型。==比较时则比较的是struct的真实值数据。

自C#7.2开始，新增了一种`readonly struct`新特性，通过名字可以看出，这种用法的主要是针对不可变数据设计的。声明了radonly struct之后，struct内的属性只能在实例化时设置，再这之后将不能进行修改，保证了数据的安全。这种用途与record的一些使用场景类似，但实际使用体验上还是record更优。

### record

本身属于引用类型，实际上是通过class特殊处理实现其功能，所以类支持的功能在record也是可行的。其兼顾class和struct的特定而成，数据存放于对上属于引用类型，支持继承和多态。但它又有值类型的特征，==或Equals的判断时，判断的是各属性的真实值。record重写了Equals和GetHashCode函数等，比较的内容改为了个属性的值而非引用地址(但是考虑到有些场景下还可能需判断引用地址是否一致，仍可以使用ReferenceEquals进行判断，它是没有被重写的)。

从本质上来说，record仍然是一个类，只是包含了一些像值类型的行为。在一些使用场景下可以作为class或struct的替代，可能会有更好的使用效果。

## 使用方式

record的定义方法可以与类一样，只是关键字从class变为record。但还提供了一种更加简洁的声明方式：

```csharp
// 1.常规的声明方式，与class一致。
record TestClass
{
    public string Name { get; set;}
}
// 2.简洁声明方式，也是声明了一个具有Name属性的记录。
record TestClass2(string Name);
```

通过上面两种方式可以看到，第二种简洁版的使用起来更简单易写。但第二种写法所声明的属性是init型，而非set，实际上创建的是一种不可变类型，这种区别在后文有讲。

这种创建不可变类型的方式，正体现了record设计的初衷——那就是推荐用来存放不可变的、简短的、需要值比较的数据。与复杂的class和struct使用场景不同，record的主旨，我认为是“数据”。虽说不可变类型是record的推荐使用形式，但它依然可以像常规的类或结构体那样使用复杂的函数和可变类型。

## init和with

在C#9.0中，与record同时增加的关键字还包括init和with。

### init

其中，`init`应用于属性中，相当于一种特殊的set，即只在初始化时可应用。但其不仅局限于record，可以使用属性的地方都适用，struct和class中一样有效。以下面class的案例来说：

```csharp
class TestClass
{
    public string Name { get; private set;}
    public int Id { get; init;}
    public string Description { get; set;}
    
    public TestClass()
    {
    }
    public TestClass(string name, int id, string desc)
    {
        Name = name;
        Id = id;
        Description = desc;
    }
}
```

通过上面的写法可以看出，init的写法于set一致。但是它提供的功能时什么样呢？

```c#
// 这样通过构造函数创建对象肯定是没问题的
var test = new TestClass("小明",1234,"少先队员");
// 这样改变一个属性的值也是没问题的
test.Description = "群众";
// 但是下面这两种写法就会报错
test.Id = 1235;
test.Name = "小刚";
```

可见，此时的Id属性和Name属性都是一种不可变类型，即创建之后不可以再发生改变的属性。虽然二者都达到了不可变的效果，但是实际上还是有区别的。那是以为private set这种形式并不是真正的不可变，因为在class的内部它依然是可变的！而init最大的不同就是，仅在创建时可以赋值，其他任何时候都不可再进行修改。事实上，readonly struct也可以达到这样的使用效果。但是只读结构体设置属性的值却只能通过构造函数来赋予。record则可以这样写：

```csharp
// 此时对于init来说也算作“初始化时”。
var test = new TestClass(){ Id = 12};
```

可见，一方面只读结构体相较record缺少了一点灵活性，另一方面，record作为一种特殊的class也有不同于 struct的使用场景。

### with

而`with`关键字则是C#提供的一种克隆语法糖，可以更快捷的创建对象副本——一种强类型的浅拷贝。但其是针对struct和record而设立，对class是不能使用的：

```csharp
//定义record
record TestRecord(string Name,int Id);
// 创建record实例
var test = new("小明",1234);
// 使用with克隆
var test1 = test with {};
// 使用with克隆并修改属性
var test2 = test with { Id = 20};
```

## record应用场景

基于record推出的初衷，可以方便地使用不可变类型和值比较功能。那么在实际使用种就可以针对性的进行使用。将其应用在需要数据不可变或值比较的场景下，无疑是十分合适的。配合init和with表达式，更加丰富了record的使用形式。那么它的常见应用场景可以参考以下几种：

#### Wep Api

用于web api返回的数据，通常作为一种一次性的传输型数据，是不需要改变的，可以使用record。

#### 并发和多线程计算

作为不可变数据类型对于并行计算和数据共享十分安全。

#### 基于浅拷贝的原型模式

通过使用with表达式，可以便捷的进行浅拷贝，而无需自己去实现克隆方法。

#### 其他情况

其他涉及到大量数据基于值类型比较和复制的场景。

## .Net6(C#10)中的新用法

在最新的C#10中，有了新的record用法。在前面的分享中，我们知道record本质上是一个特殊的class。但在C#10中，这种说法已经不再准确。因为更准确的说record变成了一个关键字。它不再是只针对class而言的数据形式了，而是出现了一种结构类型的record（或者说record类型的结构）。

```csharp
// 之前的record声明方法。
public record TestRecord1(string Name);
// 新的record声明方法，效果等同于第一种。
public record class TestRecord2(string Name);
// 新增的record struct形式。
public record struct TestRecord3(string Name);
```

从写法上也可以猜出，在C#10中新增的写法是为了区分record struct这种新形式。我们知道第二这种写法，默认是声明了不可变类型的record，但是在record struct的简化写法中，声明的却是一个可变类型的struct。因为本身就是值类型，所以也不需要再去重写比较方法（但是新增了==/!=操作符，常规struct是不支持==/!=比较的）。

那么相较于传统struct，它剩下的优势恐怕就是with表达式了。感觉并没有一些特殊的亮点，也许是我们看懂，等将来有其他感悟时再更新。

## 总结

上面的介绍可能略微笼统，但足够日常使用参考。我们需要了解的是，record并不是不可或缺的，即便没有record的出现，我们仍可以实现它所实现的种种功能，仍能处理它所面对的使用场景。无论是值比较或不可变类型，都可以通过class实现（毕竟record本质上还是class）,但不同的是，record提供给了开发者一种更便捷易用的使用方法，减少了代码量。你可以说它没有带来多巨大的变革，但不可否认的是使用record却能提高生产效率，这正是一个新事物的使命。