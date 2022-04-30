---
title: DryIOC使用介绍
date: 2022-04-10 22:58:50
tags:
  - IOC
  - C#
  - .Net
---

## IOC和DI

IOC，全称Inversion of Control，即“**控制反转**”，是一种**设计原则**。而DI（Dependency injection）指“**依赖注入**”，是实现IOC的一种方法。DI实现IOC的方法是引入“容器”，在使用前将会被依赖的对象统一注册到容器中，将对象的分配权交给容器。当需要实例对象时，并不是通过new来创建，而是经由容器自动装配得到。这个过程就是所谓的依赖注入。

请看下面这个例子：

```c#
class A
{
    public string Description { get; } = "I am A";
    public A()
    {
    }
}

class B
{
    public A A { get; init; }
    public B(A a)
    { 
        this.A = a;
    }

    public string GetDescription()
    {
        return A.Description;
    }
}
```

如果我们需要 一个B对象，需要 怎么做呢？肯定是先创建一个A对象，因为B的创建需要A作为参数，然后才可以正常的使用B处理我们的业务：

```csharp
B b = new B(new A());
```

但是是只需要一个依赖的情况，如果B还需要另一个依赖C呢，甚至在A的创建过程中还需要一个依赖D呢，这个链延续下去是不是就会很长，在每次创建B之前都需要一系列的前期准备工作，无疑是十分不便的。如果有一个办法可以简化这个繁琐的前期工作，那编码的过程必将是十分愉快的，依赖注入正是达到了这个效果。在依赖注入的使用中，创建B只需要一步。但是B仍然不能脱离对A、C、D等单独存在，只不过是将对它们的创建改为了“注册”，这种注册是不需要知道先后顺序以及依赖关系的。最终对于B的创建，所有的依赖顺序和创建都由IOC容器来负责，用户需要做的是告诉容器“我需要一个B”。后面会涉及具体写法，此处不举例。

回看“依赖注入”这个名字来看，到底什么叫控制反转呢？什么被控制了？什么被反转了？

很显然，B对象的创建被控制了，实例的出现并不经new得到而是交由IOC容器给出，这个过程是为被控制。而创建依赖对象的权利反转了。在常规的写法中依赖的创建过程由用户使用者所有，而引入IOC容器后，依赖对象的创建则完全脱离变为由容器管理，使用者变为被动等待，是为反转。

依赖注入的使用，主要解决了依赖对象间高度耦合的问题，使类可以更专注于自己的功能而不用关注依赖。

IOC容器的角色很像一个秘书。如果我们没有秘书的情况下，做一件事，从头到尾所有的细节都需要自己梳理好，然后按顺序主次分别处理，直到完成。这个过程中，无论是主要的环节还是次要的步骤都是需要亲自实施的，必然繁琐。但是如果有了秘书，次要部分自然由秘书提前处理好，而决策者只需要处理关键部分。这样工作的效率无疑会大大提高。

## .Net开发中常用的IOC容器框架


|   框架名称   |                   开源地址                    |                描述                |
| :----------: | :-------------------------------------------: | :--------------------------------: |
|    Unity     |    https://github.com/unitycontainer/unity    |     微软官方开发的依赖注入框架     |
|     MEF      |    https://github.com/microsoftarchive/mef    |             已停止维护             |
|  Spring.Net  | https://github.com/spring-projects/spring-net |                                    |
|   AutoFac    |      https://github.com/autofac/Autofac       | 最流行的依赖注入框架，轻量且性能高 |
|   Ninject    |      https://github.com/ninject/Ninject       |                                    |
|    DryIOC    |        https://github.com/dadhi/DryIoc        |            本文介绍框架            |
| StructureMap | https://github.com/structuremap/structuremap  |                                    |


以上列举的只是比较常用的一些.Net IOC容器框架，具体的使用方法大同小异。实现的最主要功能无非是依赖注入，个别可能包含了一些高级用法，或者更特殊的支持。但是在常规项目中，我们使用到的部分绝大多数还是基础功能，所以选择框架的时候，个人认为主要考虑性能和市场占有量。基于着这种考虑，按性能首推DryIOC、AutoFac、Ninject、Grace这类轻量高性能为主要特色的框架，如果按热度，则首推AutoFac、Unity这类用户数比较多的种类。至于复杂的使用场景就需要根据实际情况，具体选择适用自己项目的产品了。

考虑到使用方法相似，在这里对这些框架的具体细节不做过多介绍。下面以DryIOC为例，介绍下它的常规用法。如果将来用到的并不是DryIOC，对于别的框架的使用也是具有很大的参考意义的。重要的是思想，写法只是细节。

## DryIOC使用方法

假设当前存在以下两个接口：

```csharp
public interface IService
{
}
public interface IClient
{
    IService Service { get; }
}
```

两个对应的实现类：

```c#
public class SomeService : IService
{
    public SomeService()
    {
        System.Console.WriteLine(nameof(SomeService) + " is created.");
    }
}
public class SomeClient : IClient
{
    public IService Service { get; }
    public SomeClient(IService service)
    {
        Service = service;
        System.Console.WriteLine(nameof(SomeClient) + " is created.");
    }
    public override string ToString()
    {
        return "Hello World";
    }
}
```

在这种情况下，我们想创建一个SomeClient对象，一定是要这样写：

```csharp
var service = new SomeService();
var client = new SomeClient(service);
```

在这个过程中，创建SomeClient对象，首先需要获取一个继承于IService的实例，也就是说要知道SomeService的存在，并将其创建出来，如果有更多的依赖，还需要继续向上获取。整个链路的情况下，在创建SomeClient都需要完全了解。这已然是我们前面第一部分讲到的那种情况了。

同样是对于这种情况，DryIOC的写法是这样的：

```csharp
// 创建容器
var container = new Container();
// 注册 -> 当获取ICliecnt接口的实例时，返回SomeClient对象
container.Register<IClient, SomeClient>();
container.Register<IService, SomeService>();
// 获取对象
var client2 = container.Resolve<IClient>();
```

对比两种写法，似乎以IOC创建对象的形式更加繁琐复杂。但实际上，仔细看代码，在总共四行代码中，有三条的作用都是用来创建容器和注册接口的，真正获取实例的只有最后一条。接口一旦注册，在整个容器的生命周期内都是有效的，无论后面获取多少个对象，无论对象的依赖有多少，都是只需要一条语句就可以获取（前提是依赖已经注册）。

除了上面使用到的只是最简单的注册、创建对象的写法，还有一些指定规则、注销等功能，可以在实践中再做了解。

