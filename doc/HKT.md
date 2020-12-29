# 十分钟魔法练习：高阶类型

### By 「玩火」，改写「CWKSC」

> 前置技能：C# 基础

## 常常碰到的困难

写代码的时候常常会碰到语言表达能力不足的问题，比如下面这段用来给 `F` 容器中的值进行映射的代码：

```csharp
public interface Functor<F>
{
    public F<B> Map<A, B>(Func<A, B> f, F<A> a);
           ^^^^                         ^^^^
}
// CS0307 类型参数 'F' 不可搭配类型引数一起使用
```

并不能通过编译，编译器会告诉你 F 不能有泛型参数。

最简单粗暴的解决方案就是放弃类型检查，全上 `object` ，如：

```csharp
public interface Functor<F>
{
    public object Map(Func<object, object> f, object a);
}
```

## 高阶类型

假设类型的类型是 `Type` ，比如 `int` 和 `String` 类型都是 `Type` 。

而对于 `List` 这样带有一个泛型参数的类型来说，它相当于一个把类型 `T` 映射到 `List<T>` 的函数，其类型可以表示为 `Type -> Type` 。

同样的对于 `Map` 来说它有两个泛型参数，类型可以表示为 `(Type, Type) -> Type` 。

像这样把类型映射到类型的非平凡类型就叫高阶类型（HKT, Higher Kinded Type）。

虽然 C# 中存在这样的高阶类型但是我们并不能用一个泛型参数表示出来，也就不能写出如上 `F<A>` 这样的代码了，因为 `F` 是个高阶类型。

> 如果加一层解决不了问题，那就加两层。

虽然在 C# 中不能直接表示出高阶类型，但是我们可以通过加一个中间层来在保留完整信息的情况下强类型地模拟出高阶类型。

首先，我们需要一个中间层：

```csharp
public interface HKT<F, A> { }
```

然后我们就可以用 `HKT<F, A>` 来表示 `F<A>` ，这样操作完 `HKT<F, A>` 后我们仍然有完整的类型信息来还原 `F<A>` 的类型。

这样，上面 `Functor` 就可以写成：

```csharp
public interface Functor<F>
{
    public HKT<F, B> Map<A, B>(Func<A, B> f, HKT<F, A> a);
}
```

这样就可以编译通过了。而对于想实现 `Functor` 的类，需要先实现 `HKT` 这个中间层，这里拿 `List` 举例：

```csharp
public interface ListHKT { }
public class ListHKT<T> : HKT<ListHKT, T>, ListHKT
{
    public List<T> value;
    public ListHKT() => value = new List<T>();
    public ListHKT(List<T> v) => value = v;
    public static ListHKT<T> Narrow(HKT<ListHKT, T> v) => (ListHKT<T>)v;
}
```

注意 `ListHKT` 把自己作为了 `HKT` 的第一个参数来保存自己的类型信息，这样对于 `HKT<ListHKT, T>` 这个接口来说就只有自己这一个子类，而在 `Narrow` 函数中可以安全地把这个唯一子类转换回来。

这样，实现 `Functor` 类就是一件简单的事情了：

```csharp
public class ListF : Functor<ListHKT>
{
    public HKT<ListHKT, B> Map<A, B>(Func<A, B> f, HKT<ListHKT, A> a) => 
        new ListHKT<B>(new List<B>(
            ListHKT<A>.Narrow(a).value.Select(f)));
}
```

## 测试代码：

```csharp
List<int>       list          = new List<int>(){ 1, 2, 3, 4 };
ListHKT<int>    listHKT       = new ListHKT<int>(list);
ListF           listFunctor   = new ListF();
ListHKT<double> mappedListHKT = (ListHKT<double>)listFunctor.Map(x => x + 1.5, listHKT);
List<double>    mappedList    = mappedListHKT.value;
foreach (var ele in mappedList)
    Console.Write(ele + " ");
// 2.5 3.5 4.5 5.5
```

压缩：

```csharp
List<double> list2 = 
    ((ListHKT<double>)new ListF().Map(x => x + 1.5,
        new ListHKT<int>(
            new List<int>() { 1, 2, 3, 4 }))).value;
```

为了方便使用，可以在 `ListHKT<T>` 中加入隐含转换

```csharp
public static implicit operator ListHKT<T>(List<T> list) => new ListHKT<T>(list);
public static implicit operator List<T>(ListHKT<T> list) => list.value;
```

```csharp
ListHKT<int> listHKT = new List<int>() { 1, 2, 3, 4 };
List<double> mappedList = (ListHKT<double>)new ListF().Map(x => x + 1.5, listHKT);
```

