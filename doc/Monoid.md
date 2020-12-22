# 十分钟魔法练习：单位半群

### By 「玩火」，改写「CWKSC」

> 前置技能：C#（IEnumerable, Aggregate(), IComparable）

## Semigroup 半群

半群是一种代数结构，在集合 `A` 上包含一个将两个 `A` 的元素映射到 `A` 上的运算即 `<> : (A, A) -> A​` ，同时该运算满足**结合律**即 `(a <> b) <> c == a <> (b <> c)` ，那么代数结构 `{<>, A}` 就是一个半群。

比如在自然数集上的加法或者乘法可以构成一个半群，再比如字符串集上字符串的连接构成一个半群。

## Monoid 单位半群

单位半群是一种带单位元的半群，对于集合 `A` 上的半群 `{<>, A}` ， `A` 中的元素 `a` 使 `A` 中的所有元素 `x` 满足 `x <> a` 和 `a <> x` 都等于 `x`，则 `a` 就是 `{<>, A}` 上的单位元。

举个例子， `{+, 自然数集}` 的单位元就是 0 ， `{*, 自然数集}` 的单位元就是 1 ， `{+, 字符串集}` 的单位元就是空串 `""` 。

C# 中可表示为：

```csharp
public abstract class Monoid<T>
{
    public abstract T Empty();
    public abstract T Append(T a, T b);
    public T Appends(IEnumerable<T> x) => x.Aggregate(Empty(), Append);
}
```

不使用 interface 是因为 C# 中继承者不能直接使用默认方法。

## 应用：Optional

在 C# 中可以用 `T?` 或者 `Nullable<T>` 可以用来表示可能有值的类型，我们可以对它定义个 Monoid ：

```csharp
public class OptionalM<T> : Monoid<T?> where T : struct
{
    public override T? Empty() => null;
    public override T? Append(T? a, T? b) => a ?? b;
}
```

这样 `appends` 将获得一串 `T?` 中第一个不为空的值，对于需要进行一连串尝试操作可以这样写：

```csharp
new OptionalM<int>().Appends(new int?[] { null, 2, 3 })) // 2
```

## 应用：Ordering

存在一個 `Student` 类

```csharp
public class Student : IComparable<Student>
{
    public string name;
    public string sex;
    public DateTime birthday;
    public string from;
}
```

如果我们想对其实现 `IComparable<Student>` 接口，正常情况下：

```csharp
public int CompareTo(Student s) =>
    name.CompareTo(s.name) != 0 ? name.CompareTo(s.name) : 
    sex.CompareTo(s.sex) != 0 ? sex.CompareTo(s.sex) :
    birthday.CompareTo(s.birthday) != 0 ? birthday.CompareTo(s.birthday) :
    from.CompareTo(s.from) != 0 ? from.CompareTo(s.from) : 0;
```

对于 `IComparable` ，可以构造出一個 Monoid：

```csharp
public class OrderingM : Monoid<int> {
    public override int Empty() => 0;
    public override int Append(int a, int b) => a == 0 ? b : a;
}
```

同样如果有一串带有优先级的比较操作就可以用 appends 串起来，比如：

```csharp
public int CompareTo(Student student) => 
    new OrderingM_pre().Appends(new[] {
        name.CompareTo(student.name),
        sex.CompareTo(student.sex),
        birthday.CompareTo(student.birthday),
        from.CompareTo(student.from)
    });
```

这样的写法比一连串 `if-else` 或者 `?:` 优雅太多。

## C# 中 Monoid 另一种实现：

个人感觉对于 Monoid 的定义和用法上有点复杂和麻烦，这里提供另一种实现：

### Monoid

```csharp
public static Func<IEnumerable<T>, T> Monoid<T>(Func<T> Empty, Func<T, T, T> Append) => 
    list => list.Aggregate(Empty(), Append);
```

### Optional

```csharp
public static T? OptionalM<T>(params T?[] list) where T : struct =>
    Monoid<T?>(() => null, (a, b) => a ?? b)(list);
```

```csharp
OptionalM(null, 2, 3) // 2
```

### OrderingM

```csharp
public static int OrderingM(params int[] list) =>
    Monoid(() => 0, (a, b) => a == 0 ? b : a)(list);
```

```csharp
public int CompareTo(Student_pre student) =>
    OrderingM(
        name.CompareTo(student.name),
        sex.CompareTo(student.sex),
        birthday.CompareTo(student.birthday),
        from.CompareTo(student.from)
    );
```

## 扩展

在 Monoid 接口里面加 default 方法可以支持更多方便的操作：

```csharp
interface Monoid<T> {
    //...
    default T when(boolean c, T then) {
        if (c) return then;
        else return empty();
    }
    default T cond(boolean c, T then, T els) {
        if (c) return then;
        else return els;
    }
}

class Todo implements Monoid<Runnable> {
    public Runnable empty() {
        return () -> {};
    }
    public Runnable 
    append(Runnable a, Runnable b) {
        return () -> { a(); b(); };
    }
}
```

然后就可以像下面这样使用上面的定义:

```java
new Todo.appends(Stream.of(
    logic1,
    () -> { logic2(); },
    Todo.when(condition1, logic3)
))
```

> 注：上面的 Optional 并不是 lazy 的，实际运用中加上非空短路能提高效率。