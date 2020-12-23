# 十分鐘魔法練習：單位半群

### By 「玩火」，改寫「CWKSC」

> 前置技能：C# (IEnumerable, Aggregate(), Extension Methods, IComparable)

## Semigroup 半群

半群是一種代數結構，在集合 `A` 上包含一個將兩個 `A` 的元素映射到 `A` 上的運算即 `<> : (A, A) -> A` ，同時該運算滿足**結合律**即 `(a <> b) <> c == a <> (b <> c)` ，那麼代數結構 `{<>, A}` 就是一個半群。

比如在自然數集上的加法或者乘法可以構成一個半群，再比如字符串集上字符串的連接構成一個半群。

```csharp
// 二元運算
public interface BinaryOperation<T> { public Func<T, T, T> BinaryOperation { get; set; } }
// 半群
public interface Semigroup<T> : BinaryOperation<T> { }
```

## Monoid 單位半群

單位半群是一種帶單位元的半群，對於集合`A` 上的半群`{<>, A}` ， `A` 中的元素 `a` 使 `A` 中的所有元素 `x ` 滿足 `x <> a` 和 `a <> x` 都等於 `x`，則 `a` 就是 `{<>, A}` 上的單位元。

舉個例子， `{+, 自然數集}` 的單位元就是 0 ， `{*, 自然數集}` 的單位元就是 1 ， `{+, 字符串集}` 的單位元就是空串 `"" ` 。

```csharp
// 單位元
public interface Unital<T> { public Func<T> Identity { set; get; } }
// 單位半群
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

對於 `Monoid` 這種代數結構，可以折疊 `fold` / `reduce`：

```csharp
public static T Appends<T>(this Monoid<T> monoid, IEnumerable<T> x) =>
    x.Aggregate(monoid.Identity(), monoid.BinaryOperation);

// 下面這個是為了方便使用
public static T Appends<T>(this Monoid<T> monoid, params T[] x) =>
    monoid.Appends((IEnumerable<T>)x);
```

對於其他在 `Monoid` 上使用的運算，可以用 C# 中的 Extension Methods 添加。

## 應用：Optional

在 C# 中可以用 `T?` 或者 `Nullable<T>` 可以用來表示可能有值的類型，我們可以對它定義個 `Monoid` ：

```csharp
public static Monoid<T?> OptionalM<T>() where T : struct =>
    new Monoid<T?>(() => null, (a, b) => a ?? b);
```

這樣 `appends` 將獲得一串 `T?` 中第一個不為空的值，對於需要進行一連串嘗試操作可以這樣寫：

```csharp
OptionalM<int>().Appends(null, 2, 3) // 2
```

## 應用：Ordering

存在一個 `Student` 類

```csharp
public class Student : IComparable<Student>
{
    public string name;
    public string sex;
    public DateTime birthday;
    public string from;
}
```

如果想對其實現 `IComparable<Student>` 接口，正常情況下：

```csharp
public int CompareTo(Student s) =>
    name.CompareTo(s.name) != 0 ? name.CompareTo(s.name) :
    sex.CompareTo(s.sex) != 0 ? sex.CompareTo(s.sex) :
    birthday.CompareTo(s.birthday) != 0 ? birthday.CompareTo(s.birthday) :
    from.CompareTo(s.from) != 0 ? from.CompareTo(s.from) : 0;
```

對於 `IComparable` ，可以構造出一個 Monoid：

```csharp
public static Monoid<int> OrderingM() =>
    new Monoid<int>(() => 0, (a, b) => a == 0 ? b : a);
```

同樣如果有一串帶有優先級的比較操作就可以用 appends 串起來，比如：

```csharp
public int CompareTo(Student student) =>
    OrderingM().Appends(
        name.CompareTo(student.name),
        sex.CompareTo(student.sex),
        birthday.CompareTo(student.birthday),
        from.CompareTo(student.from)
    );
```

這樣的寫法比一連串 `if-else` 或者 `?:` 優雅太多。

## 擴展

用 Extension Methods 對 `Monoid` 添加方法，支持更多方便的操作：

```csharp
public static T When<T>(this Monoid<T> monoid, bool c, T then) =>
    c ? then : monoid.Identity();
public static T Cond<T>(this Monoid<T> monoid, bool c, T then, T els) =>
    c ? then : els;
```

存在一個 `Todo` 類，

```csharp
public static Monoid<Action> Todo() =>
    new Monoid<Action>(
        () => () => { },
        (a, b) => () => { a(); b(); });
```

然後就可以像下面這樣使用上面的定義:

```csharp
Monoid<Action> todo = Todo();
todo.Appends(
    () =>　Console.WriteLine("logic1"),
    todo.When(true, () => Console.WriteLine("logic2 When true")),
    todo.Cond(false,
        () => Console.WriteLine("logic3 Cond true"),
        () => Console.WriteLine("logic4 Cond false"))
)();
// logic1
// logic2 When true
// logic4 Cond false
```

> 注：上面的 Optional 並不是 lazy 的，實際運用中加上非空短路能提高效率。