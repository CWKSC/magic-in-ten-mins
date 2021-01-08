# 十分钟魔法练习：单子

### By 「玩火」

> 前置技能：C# 基础，[HKT](HKT.md)

## Monad 单子

单子(Monad)是指一种有一个类型参数的数据结构，拥有 `pure` （也叫 `unit` 或者 `return` ）和 `flatMap` （也叫 `bind` 或者 `>>=` ）两种操作：

```csharp
public interface Monad<T>
{
    public HKT<T, V> Pure<V>(V value);
    public HKT<T, B> FlatMap<A, B>(HKT<T, A> a, Func<A, HKT<T, B>> f);
}
```

其中 `Pure` 要求返回一个包含参数类型内容的数据结构， `FlatMap` 要求把 `a` 的值经过 `f` 以后再串起来。

举个最经典的例子：

## List Monad

```csharp
public class ListM : Monad<ListHKT>
{
    public HKT<ListHKT, V> Pure<V>(V value) =>
        new ListHKT<V>(new List<V>() { value });

    // 方便使用 
    public HKT<ListHKT, V> Pure<V>(params V[] values) =>
        new ListHKT<V>(new List<V>(values));

    public HKT<ListHKT, B> FlatMap<A, B>(HKT<ListHKT, A> a, Func<A, HKT<ListHKT, B>> f) => 
        ListHKT<A>.Narrow(a).value.Aggregate(new ListHKT<B>(), (result, ele) =>
        {
            result.value.AddRange(ListHKT<B>.Narrow(f(ele)).value);
            return result;
        });
}
```

简单来说 `Pure(v)` 将得到 `{v}` ，而 `FlatMap({1, 2, 3}, v -> {v + 1, v + 2})` 将得到 `{2, 3, 3, 4, 4, 5}` 。

```csharp
ListM listM = new ListM();

ListHKT<int> pureHKT = (ListHKT<int>)listM.Pure(42, 10007);
List<int> pure = pureHKT.value;
foreach (var ele in pure)
    Console.Write(ele + " ");
// 42 10007
Console.WriteLine();

List<int> list = new List<int>() { 1, 2, 3, 4 };
ListHKT<int> listHKT = new ListHKT<int>(list);

ListHKT<double> flatMappedListHKT = 
    (ListHKT<double>)listM.FlatMap(listHKT, v => 
                                   listM.Pure(v + 0.5, v + 1.5));

List<double> flatMappedList = flatMappedListHKT.value;
foreach (var ele in flatMappedList)
    Console.Write(ele + " ");
// 1.5 2.5 2.5 3.5 3.5 4.5 4.5 5.5
```

## Maybe Monad

C# 不是一个空安全的语言，也就是说任何对象类型的变量都有可能为 `null` 。对于一串可能出现空值的逻辑来说，判空常常是件麻烦事：

```csharp
public static Maybe<int?> AddI(Maybe<int?> ma, Maybe<int?> mb)
{
    if (ma is Nothing<int?> || mb is Nothing<int?>)
        return new Nothing<int?>();
    return new Just<int?>(
        ((Just<int?>)ma).value +
        ((Just<int?>)mb).value);
}
```

`Maybe` 的实现：

```csharp
public interface Maybe { }
public interface Maybe<T> : HKT<Maybe, T>, Maybe
{
    public static Maybe<T> Narrow(HKT<Maybe, T> v) => (Maybe<T>)v;
}

public class Nothing<T> : Maybe<T> { }
public class Just<T> : Maybe<T>
{
    public T value;
    public Just() { }
    public Just(T value) => this.value = value;
}
```

可以像这样定义 `Maybe Monad` ：

```csharp
public class MaybeM : Monad<Maybe>
{
    public HKT<Maybe, V> Pure<V>(V value) => new Just<V>(value);
    public HKT<Maybe, B> FlatMap<A, B>(HKT<Maybe, A> a, Func<A, HKT<Maybe, B>> f)
    {
        A value = ((Just<A>)Maybe<A>.Narrow(a)).value;
        return value == null ? new Nothing<B>() : f(value);
    }
}
```

上面 `AddI` 的代码就可以改成：

```csharp
public static Maybe<int?> AddI(Maybe<int?> ma, Maybe<int?> mb)
{
    MaybeM m = new MaybeM();
    return Maybe<int?>.Narrow(
        m.FlatMap(ma, a =>
        m.FlatMap(mb, b =>
        m.Pure(a + b)))
    );
}
```

这样看上去就比上面的连续 `if-return` 优雅很多。在一些有语法糖的语言 (`Haskell`) 里面 Monad 的逻辑甚至可以像上面右边的注释一样简单明了。

> 我知道会有人说，啊，我有更简单的写法：
>
> ```java
> public static Maybe<int?> AddE(Maybe<int?> ma, Maybe<int?> mb)
> {
>        try
>        {
>            return new Just<int?>(((Just<int?>)ma).value + ((Just<int?>)mb).value);
>        }
>        catch (Exception)
>        {
>            return new Nothing<int?>();
>        }
> }
> ```
>
> 确实，这样写也挺简洁直观的， `Maybe Monad` 在有异常的 C# 里面确实不是一个很好的例子，不过 `Maybe Monad` 确实是在其他没有异常的函数式语言里面最为常见的 Monad 用法之一。而之后我也会介绍一些异常也无能为力的 Monad 用法。

