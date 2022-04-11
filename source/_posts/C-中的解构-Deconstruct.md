---
title: C#中的解构(Deconstruct)
date: 2021-12-28 00:26:02
tags: 
  - C#
  - .Net
---

**解构(Deconstruct)**是C#7.0中增加的新用法。解构的作用是拆解一个复杂的数据结构为相对原始数据简单的部分，用以后续的使用场景。简单的说，就是在满足使用的基础上删繁就简，使我们写出的代码更优雅、便捷。解构功能配合模式匹配使用效果极佳。

## 类的解构

在常见的使用场景中，面对复杂对象的比较或者其他使用时，往往我们并不需要使用整个对象，仅仅会用到其包含的一部分内容。即使出于安全考虑，很多时候我们也不希望向无关的处理暴露太多的数据。这个时候，很多东西又不得不依赖，写出的代码往往就会比较臃肿。所以给人的感觉就像我想知道一个人的身高体重，却不得不先拿到他的户口本知道所有信息。这样既不优雅又不安全。解构的出现就很好的解决了这个问题。

举个例子。

```C#
class User
{
    public string Name { get; set; }
    public int Age { get; set; }
    public string Sex { get; set; }
    public string Address { get; set; }
    ...
}
```

上面这个是一个常见的用户信息类，内容包括名字、年龄、性别、住址，假设它还包括了其他很多内容。现在假如有一段逻辑，可以通过人的年龄和性别制定他的饮食计划。这个时候可能就需要使用很多switch或者if来实现各种分支处理了，如果依赖的条件更多，那么就更没办法做到简洁易懂了。

这个时候就可以使用Deconstruct，可以这样写：

```CSharp
var user = new User()
{
    Name = "小明",
    Age = 18,
    Sex = "男",
    Address = "张家村"
};

switch (user)
{
    case ("男", 29):
        //
        break;
    case ("女", 30):
        //
        break;
    case ("男", 18):
        Console.WriteLine("制定18岁男生的饮食计划");
        break;
}
```

或者更容易理解使用方法是这样（针对“解构”这个概念来说——将复杂的数据结构拆解出需要的部分）：

```c#
(string sex,int age) = user;
Console($"sex:{sex},age:{age}");
```

这样看起来是不是非常清晰明了，很容易就看懂这段代码的意思。但是，case后的条件是怎么实现的呢，这还是需要在User类中进行相关的声明的：

```C#
class User
{
    public string Name { get; set; }
    public int Age { get; set; }
    public string Sex { get; set; }
    public string Address { get; set; }

    public void Deconstruct(out string sex, out int age)
    {
        sex = Sex;
        age = Age;
    }
}
```

只需要添加一个名称为Deconstruct的方法就可以实现解构的功能了。但条件是：

>1. 方法名称必须是Deconstruct
>2. 返回类型必须是void
>3. 参数列表必须为out

满足以上三个条件就可以实现类的定制化解构了。

同时，解构还支持重载，可以针对具体场景和不同需求重载不同参数列表的解构函数。但是 ，这里有一个特殊的点是，解构的重载只能针对不同数量的参数列表进行重载，而对于参数列表数量相同类型不同，也是不支持的。

什么意思呢？针对上面的例子，这种重载是可以的：

```c#
public void Deconstruct(out string name, out string sex, out int age)
{
    name = Name;
    sex = Sex;
    age = Age;
}
```

可以的原因是因为这个重载列表中具有三个参数，参数列表的数量发生了变化。但是假如重载的数量还是两个没变，但是参数的类型发生了变化，这样写就是错误的：

```C#
public void Deconstruct(out bool isBoy, out int age)
{
    isBoy = Sex.Equals("男");
    age = Age;
}
```

## 元组的解构

元组因为没有参数名称，并且也不需要预先定义内容，所提它的解构就不用像类的解构一样预处理一部分逻辑了。

```C#
var tuple = new Tuple<string, int, bool>("xiaoming", 9, true);
(string name, int age,bool isBoy) = tuple;
```

仅仅需要按照元组的元素列表组装解构就可以了。但是，也有注意事项，那就是解构必须包含所解构元素的按顺序的全部元素。即便有的参数是不需要的，也应以弃元_形式占位处理。

如此看来，元组的解构处理命名之外似乎意义不大，这种情况还不如直接使用ValueTuple满足需求。

## 记录的解构

C#9.0之后又添加了新的数据类型记录(Record)，可以将其理解成一种介于元组和类之间的一种数据结构。

对于记录的解构，也与其自身的定位类似，既可以使用类的解构方法又可以使用元组的解构方法。实际生产中，如果使用类的解构形式，那么与类解构一样，需预先写好解构函数；如采用元组的解构形式，那么也需在解构过程中保证元素列表的顺序和数量与记录本身一一对应，好处是不用在记录中写解构函数。
