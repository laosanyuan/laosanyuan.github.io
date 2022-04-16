---
layout: new
title: C#模式匹配使用简介
date: 2022-04-12 08:44:46
tags:
  - .Net
  - C#
  - 模式匹配
---

## 模式匹配是什么

模式匹配是C#7开始引入的一项新功能，之后的每个版本都扩展了该功能。模式匹配是一种测试表达式是否具有特定特征的一种方法，简而言之，模式是一项根据“模式”来“匹配”结果的用法。通过模式匹配这一功能，可以使用更简洁明了的语法从表达式中提取目标结果。

模式匹配主要包含`is`和`switch`两种表达式形式，后续虽然添加了其他关键字和用法也是基于二者的。

## 常见的模式匹配类型

### 类型模式【From C#9.0】

用于检查表达式运行时变量的类型。

```csharp
if(a is AType)
{
	// do something
}

// 以下用法自C#9.0之后的版本开始支持。
var result = a switch
{
        Atype => 3,
        BType => 4,
        null => 4,
        // _是弃元，相当于传统switch语句中的default。
        _ => 5
};
```

### 声明模式【From C#7.0】

用于检查表达式运行时的类型，如果匹配成功，则将结果分配给新的声明的局部变量。声明模式再某种程度上相当于基于类型模式的一种拓展，即在类型模式匹配成功的基础上声明一个新的局部变量。在声明模式声明变量的局部范围内，可以正常使用。

```c#
if(a is AType tmpA)
{
	console.WriteLine("id:"+tmpA.Id);
}

// 以下用法自C#9.0之后的版本开始支持。
var result = a switch
{
        Atype tmpA=> tmpA.Id,
        BType tmpB=> tmpB.Id,
        _ => 5
};
```

### 常量模式【From C#7.0】

常量模式使用方法与类型模式类似，写法相同，只不过是由对类型的判断变为对常量的判断。is/switch都支持此功能。

与类型模式不同的是，在C#9.0之后，常量模式支持了not关键字的用法，用于判断非匹配情况：

```csharp
if (input is not "test string")
{
    // do something
}
```

### 关系模式【From C#9.0】

通过关系模式，可以将表达式结果与常量进行比较。写法上可以直接使用关系运算符进行直接比较进行匹配。

但是需要注意的是，关系模式的右侧部分必须是常数表达式，表达式可以是`整型、浮点型、char型或enum类型`。

```c#
static string Classify(double measurement) => measurement switch
{
    < -4.0 => "Too low",
    > 10.0 => "Too high",
    double.NaN => "Unknown",
    _ => "Acceptable",
};
```

### 逻辑模式【From C#9.0】

逻辑模式的关键字为`and`、`or`、`not ，分别对应与、或、非三种逻辑关系。使用三种逻辑关系，可在匹配条件中进行多条件的逻辑组合，达到更便捷、清晰的效果。

```csharp
var result = b switch
{
    > 10.0 and < 11.0 => "Too high",
    8.0 or 4.0 or 3.0 => "test",
    not double.NaN => "Unknown",
    _ => "Acceptable",
};
```

### 属性模式【From C#8.0】

属性模式是专门针对对复杂对象的特定属性进行匹配的一种方法。当一个对象的多个属性中只有某一个或某几对匹配有影响时，显然只对特定的属性进行匹配就可以：

```csharp
// 此函数只对2020-5-19/20/21进行匹配，则DateTime中的其他属性就是不重要的。
static bool IsConferenceDay(DateTime date) => date is { Year: 2020, Month: 5, Day: 19 or 20 or 21 };
// switch表达式中的用法：
var result = dt switch
{
        { Day: 20 } => "20日",
        { Month: 2 } => "2月",
        { Hour: >= 12, Minute: 30 } or { Hour: 0, Minute: 30 } => "12:30",
        // 递归匹配
        { TimeOfDay:{  Hours:20 } } => "Day 20 hours",
        // C#1.0之后针对递归匹配新增的语法，逻辑与上一条相同。
        { TimeOfDay.Hours:10} => "Day 10 hours",
        _ => throw new NotImplementedException(),
};
```

属性模式是一种递归模式，即对属性的匹配可以继续向属性内部进行，使匹配目标更为具体深入。

### 位置模式【From C#8.0】

位置模式针对元组和解构函数使用。通过对元组或者解构函数位置元素的匹配达到匹配效果，本质上与属性模式类似：

```csharp
public readonly struct Point
{
    public int X { get; }
    public int Y { get; }
    public Point(int x, int y) => (X, Y) = (x, y);
    public void Deconstruct(out int x, out int y) => (x, y) = (X, Y);
}

static string Classify(Point point) => point switch
{
    (0, 0) => "Origin",
    (1, 0) => "positive X basis end",
    (0, 1) => "positive Y basis end",
    (>1,<=0) => "Not valid",
    (_,1) => "give up",
    _ => "Just a point",
};
```