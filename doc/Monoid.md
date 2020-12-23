# 十分钟魔法练习：单位半群

### By 「玩火」，改写「CWKSC」

> 前置技能：C#（IEnumerable, Aggregate(), IComparable）

## Semigroup 半群

半群是一种代数结构，在集合 `A` 上包含一个将两个 `A` 的元素映射到 `A` 上的运算即 `<> : (A, A) -> A​` ，同时该运算满足**结合律**即 `(a <> b) <> c == a <> (b <> c)` ，那么代数结构 `{<>, A}` 就是一个半群。

比如在自然数集上的加法或者乘法可以构成一个半群，再比如字符串集上字符串的连接构成一个半群。

```csharp
// 二元运算
public interface BinaryOperation<T> { public Func<T, T, T> BinaryOperation { get; set; } }
// 半群
public interface Semigroup<T> : BinaryOperation<T> { }
```

## Monoid 单位半群

单位半群是一种带单位元的半群，对于集合 `A` 上的半群 `{<>, A}` ， `A` 中的元素 `a` 使 `A` 中的所有元素 `x` 满足 `x <> a` 和 `a <> x` 都等于 `x`，则 `a` 就是 `{<>, A}` 上的单位元。

举个例子， `{+, 自然数集}` 的单位元就是 0 ， `{*, 自然数集}` 的单位元就是 1 ， `{+, 字符串集}` 的单位元就是空串 `""` 。

```csharp
// 单位元
public interface Unital<T> { public Func<T> Identity { set; get; } }
// 单位半群
public class Monoid<T> : Unital<T>, Semigroup<T> {
    public Monoid(Func<T> Identity, Func<T, T, T> BinaryOperation)
    {
        this.Identity = Identity;
        this.BinaryOperation = BinaryOperation;
    }
    public Func<T> Identity { get; set; }
    public Func<T, T, T> BinaryOperation { get; set; }
}
```

对于 `Monoid` 这种代数结构，可以折叠 `fold` / `reduce`：

```csharp
public static T Appends<T>(this Monoid<T> monoid, IEnumerable<T> x) => 
    x.Aggregate(monoid.Identity(), monoid.BinaryOperation);
// 下面这个是为了方便使用
public static T Appends<T>(this Monoid<T> monoid, params T[] x) =>
    monoid.Appends((IEnumerable<T>)x);
```

对于其他在 `Monoid` 上使用的运算，可以用 C# 中的 Extension Methods 添加。

## 应用：Optional

在 C# 中可以用 `T?` 或者 `Nullable<T>` 可以用来表示可能有值的类型，我们可以对它定义个 `Monoid` ：

```csharp
public static Monoid<T?> OptionalM<T>() where T : struct => 
    new Monoid<T?>(() => null, (a, b) => a ?? b);
```

这样 `appends` 将获得一串 `T?` 中第一个不为空的值，对于需要进行一连串尝试操作可以这样写：

```csharp
OptionalM<int>().Appends(null, 2, 3) // 2
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

如果想对其实现 `IComparable<Student>` 接口，正常情况下：

```csharp
public int CompareTo(Student s) =>
    name.CompareTo(s.name) != 0 ? name.CompareTo(s.name) : 
    sex.CompareTo(s.sex) != 0 ? sex.CompareTo(s.sex) :
    birthday.CompareTo(s.birthday) != 0 ? birthday.CompareTo(s.birthday) :
    from.CompareTo(s.from) != 0 ? from.CompareTo(s.from) : 0;
```

对于 `IComparable` ，可以构造出一個 Monoid：

```csharp
public static Monoid<int> OrderingM() =>
    new Monoid<int>(() => 0, (a, b) => a == 0 ? b : a);
```

同样如果有一串带有优先级的比较操作就可以用 appends 串起来，比如：

```csharp
public int CompareTo(Student student) => 
    OrderingM().Appends(
        name.CompareTo(student.name),
        sex.CompareTo(student.sex),
        birthday.CompareTo(student.birthday),
        from.CompareTo(student.from)
    );
```

这样的写法比一连串 `if-else` 或者 `?:` 优雅太多。

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