---
title: MvvmLight使用梳理
date: 2021-02-07 09:09:15
tags:
  - C#
  - WPF
  - MvvmLight
  - IOC
---

## MVVM模式

MVVM模式是一种将视图与业务高度解耦的基于数据绑定的设计模式。配合WPF的Binding用法，可以进行轻松地使用MVVM模式开发WPF应用程序。关于MVVM模式的讲解网上有很多，此处不做赘述。只列举下各模块之间的对应关系方便理解：

> Model：负责数据实体的结构处理，与ViewModel交互。
>
> View：负责UI界面的展示，与ViewModel交互。
>
> ViewModel：负责界面视图业务的逻辑处理并将其反映给界面，是View和Model的中间连接部分。

## MvvmLight

### MvvmLight的构成

####   ViewModelBase   

所有ViewModel继承的基类，其继承自ObservableObject，同时实现了ICleanup接口，ObservableObject实现了INotifyPropertyChanged接口，用于通知属性的改变。

可以通过下面的写法提供用于绑定的属性：

```csharp
private string nickName = "未知账号";
/// <summary>
/// 昵称
/// </summary>
public string NickName
{
    get =>this.nickName; 
    set
    {
        this.nickName = value;
        base.RaisePropertyChanged(nameof(NickName), value);
    }
}
```

####  ViewModelLocator

ViewModel的定位器，全部ViewModel集中创建、分发的地方，是View连接View的唯一来源。理论上在MvvmLight下不允许自行创建ViewModel实例，这样可以避免View层与ViewModel的过度耦合和对象实例的滥用。每个ViewModel都应在ViewModelLocator中体现，且应依照Nuget安装MvvmLight模板样式以依赖注入显示创建对应实例通过只读属性对外暴露。在引入MvvmLight之后，已通过模板自动创建了ViewModellocator，且将其对象实例加入到了全局资源中，这样可以在任何地方以资源方式使用 。

但是需要强调的是，MvvmLocator并不是必须使用的。不使用或直接删除也一样可以应用Mvvm模式，只是这种结构相当于MvvmLight提供了一种使用建议。

以下是安装引入MvvmLight之后自动生成的标准ViewModelLocator：

```c#
public class ViewModelLocator
{
    /// <summary>
    /// Initializes a new instance of the ViewModelLocator class.
    /// </summary>
    public ViewModelLocator()
    {
        ServiceLocator.SetLocatorProvider(() => SimpleIoc.Default);

        SimpleIoc.Default.Register<AppViewModel>();
    }

    public AppViewModel Main
    {
        get
        {
            return ServiceLocator.Current.GetInstance<AppViewModel>();
        }
    }

    public static void Cleanup()
    {
        // TODO Clear the ViewModels
    }
}

```

以上案例可以看出，ViewModelLocator通过依赖注入注册来创建AppViewModel实例对象，并通过一个只读的Main属性向外传递，这样在外界就可以通过ViewModelLocator来使用AppViewModel。并且现在这种写法，所创建出来的对象是单例的，确保只有一个实例存在。

创建ViewModelLocator的同时还会将其实例添加到App.xaml中，并命名为Locator：

```xml
<Application x:Class="MvvmDemo.App" xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" 
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml" 
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008" 
             d1p1:Ignorable="d" 
             xmlns:d1p1="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:mvvmDemo="clr-namespace:MvvmDemo" StartupUri="Views/AppView.xaml">
    <Application.Resources>
        <mvvmDemo:ViewModelLocator x:Key="Locator" d:IsDataSource="True" />
    </Application.Resources>
</Application>    
```

这样，在View中就可以进行直接引用：

```xaml
<Window x:Class="MvvmDemo.Views.AppView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:validationRules="clr-namespace:MvvmDemo.ValidationRules"
        xmlns:converter="clr-namespace:MvvmDemo.Converter"
        xmlns:i="http://schemas.microsoft.com/expression/2010/interactivity"
        xmlns:command="http://www.galasoft.ch/mvvmlight"
        Title="AppView" 
        Height="300" Width="300" 
        DataContext="{Binding Source={StaticResource Locator},Path=Main}">
</Window>
```

以上写法即为将DataContext设置为Locator(ViewModelLocator资源对象)中的Main(AppViewModel依赖注入实例)。这种写法与cs文件中写法效果一样：

```csharp
 public AppView()
 {
 	this.DataContext=new AppViewModel();
 }
```

但是，写在.xaml中和写在.cs中还是有区别的，那就是写在.xaml文件中时，可以通过设计界面看到实时绑定数据，VS的即时提示也对编码十分友好，而写在.cs中时，则不会实时提示。所以正常来说，写在.xaml中则更为优雅舒畅。

#### Messenger

顾名思义，视为信息类，提供全局的消息订阅、通知功能。可类比Event事件，但其与事件又大不相同。Messenger的使用相对于事件是极为自由的。首先，他不关心注册的对象，其次不在乎调用者，第三不关心发送的内容。也就是说任何对象都可以接收消息、任何对象都可以发送消息、任何对象都可以作为消息的内容被发送。其本质是一个灵活的全局委托，供松耦合的结构交换信息。

有了Messnger的存在，配合MVVM模式，可以更大程度的解耦ViewModel与ViewModel之间、View与ViewModel的依赖，达到调用者与使用者互不知情的情况下进行通信的效果。

常用写法如下：

```csharp
// 消息订阅，此处“token”为消息接收识别的令牌，num则为消息内容
Messenger.Default.Register<int>("token", num =>
                                {
                                    // do something
                                });
// 消息发送，参数分别为消息内容和识别令牌
Messenger.Default.Send<int>(20, "token");
// 注销消息                
Messenger.Default.Unregister<int>("token");
```

以上是全局Messenger的使方法，使用的是Messenger的Default静态对象，全局有效。如果需要在特定范围内使用，也可根据场景自行创建对象来使用。

但是使用的时候需要注意的是，在结束后记得注销消息，否则也可能会产生逻辑上的错误。可以考虑配合ViewModelBase的Cleanup()函数进行自动注销（实际上ViewModelBase中已经写了对MessengerInstance的注销逻辑）。使用Messenger的好处是减少了对象之间消息传递的强依赖，但是坏处是可能会使我们的代码不容易读懂和调试。所以在使用的时候也不可过度滥用，权衡评判之后再下手不迟。

#### SImpleIoc

SimpleIoc是MvvmLight提供的一个简单的IOC容器。ViewModelLocator中默认的实例创建方式就是通过SimmpleIoc注入实现。SimpleIoc实现了注册、注销、获取实例等基础的IOC容器功能，创建对象时IOC容器可以根据注册对各构造函数参数进行自动装配，用于解耦用户创建实例时的相互依赖。此处依赖注入是针对性的优化功能，但不是必须存在的，即便不适用IOC创建对象，直接new一个需要的实例效果并没有什么不同。

通过上面ViewModelLocator的代码可以看到，关于IOC的使用部分，其实是包含了两部分。首先是`ServiceLocator`,其次才是`SimpleIoc`。ServiceLocator是一个来自CommonServiceLocator库的IOC中继器，用于调用触发IOC框架的功能。SImpleIoc类正是继承了CommonServiceLocator中的`IServiceLocator`接口，所以其可以作为参数设置ServiceLocator的实际使用框架实例。在上面的代码中，实际上这两种写法是等效的：

```csharp
ServiceLocator.Current.GetInstance<AppViewModel>();
SimpleIoc.Default.GetInstance<AppViewModel>();
```

实际上，最终调用的功能都是SimpleIoc的GetInstance函数，但是仍然建议使用ServiceLocator下获取，需要牢记我们解耦的初衷。

CommonServiceLocator的功能十分有限，这个项目总共代码也不过三百多行。它最大的意义在于使用接口规范了IOC框架的结构。对于ServiceLocator的操作都是针对接口的操作，而具体的实例则是用户传入的。这样使用的好处是进一步解耦IOC框架与具体项目，如果自带的SimpleIoc框架不能满足实际使用时，也可在ServiceLocator中替换合适的框架，或完全自己继承IServiceLocator进行实现，也是一种可行的途径。ServiceLocator的源码可以参考[这里](https://github.com/unitycontainer/commonservicelocator)。

需要注意的是，SimpleIoc注入创建的对象都是单例的，如业务有需求需要创建非单例实例，需要在GetInstance传入一个key做区分：

```csharp
// 创建一次性实例，并将其保留在Ioc容器中，供后续使用。
var tmp1 =SimpleIoc.Default.GetInstance<Window>(Guid.NewGuid().ToString());
// 创建一次性实例，但不在IOC容器中保留实例，下次以相同key获取返回的是不同实例。
var tmp2 =SimpleIoc.Default.GetInstanceWithoutCaching<Window>(Guid.NewGuid().ToString());
```

---

以上就是本次对MvvvmLight框架的一次简单梳理，主要讲解了框架下的几个重要组成部分，方便对整体有个大致的感知。关于其他具体使用的细节还需要在实际中进行更深入的理解，以辅助我们在开发中获得更高的效率和更好的体验。