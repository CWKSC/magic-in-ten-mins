# 十分钟魔法练习：代数数据类型

### By 「玩火」，改写「CWKSC」

> 前置技能：C# 基础

## Product type 积类型

积类型是指同时包括多个值的类型，例如 C# 中的 class 包括多个字段：


```csharp
public sealed class Student
{
    string name;
    int id;
}
```

 `Student` 的类型中既有 `string` 类型的值也有 `int` 类型的值，可以表示为 `string` 和 `int` 的「积」，即 `string * int` 

## Sum type 和类型

和类型是指可以是某一些类型之一的类型，在 C# 中可以用继承来表示：

```csharp
public interface ISchoolPerson { }
public sealed class Student : ISchoolPerson
{
    string name;
    int id;
}
public sealed class Teacher : ISchoolPerson
{
    string name;
    string office;
}
```

`SchoolPerson` 可能是 `Student` 也可能是 `Teacher` ，可以表示为 `Student` 和 `Teacher` 的「和」，即 `String * int + String * String` 。而使用时只需要用 C# 中的 `is` 就能知道当前的 `StudentPerson` 具体是 `Student` 还是 `Teacher` 

```csharp
ISchoolPerson student = new Student();
ISchoolPerson teacher = new Teacher();

(student is ISchoolPerson) // True
(student is Student)       // True
(student is Teacher)       // False

(teacher is ISchoolPerson) // True
(teacher is Student)       // False
(teacher is Teacher)       // True
```

## ADT, Algebraic Data Type 代数数据类型

由和类型与积类型组合构造出的类型就是代数数据类型，其中代数指的就是和与积的操作

### Bool 布尔 

利用和类型的枚举特性与积类型的组合特性，我们可以构造出 C# 中本来很基础的基础类型，比如枚举布尔的两个量来构造布尔类型：

```csharp
public interface Bool { }
public sealed class True : Bool { }
public sealed class False : Bool { }
```

然后用 `t is True` 就可以用来判定 `t` 作为 `Bool` 的值是不是 `True` 

`enum` 也有相同的效果：

```csharp
public enum Bool { True, False }
```

### Natural number 自然数

比如利用 S 的数量表示的自然数：

```csharp
public interface Nat { }
public sealed class Z : Nat { }
public sealed class S : Nat
{
    public Nat value;
    public S(Nat v) { value = v; }
}
```

这里提一下自然数的皮亚诺构造，一个自然数要么是 0 (也就是上面的 `Z` ) 要么是比它小一的自然数 +1 (也就是上面的 `S` ) ，例如 3 可以用 `new S(new S(new S(new Z))` 来表示

```csharp
public static int CountNat(Nat number)
{
    int count = 0;
    while(!(number is Z))
    {
        number = ((S)number).value;
        count++;
    }
    return count;
}

Nat number = new S(new S(new S(new Z())));
CountNat(number) // 3
```

### Linked List 链表

再比如链表：

```csharp
public interface List<T> { }
public sealed class Nil<T> : List<T> { }
public sealed class Cons<T> : List<T> {
    public T value;
    public List<T> next;
    public Cons(T v, List<T> n) {
        value = v;
        next = n;
    }
}
```

`[1, 3, 4]` 就表示为 `new Cons<int>(1, new Cons<int>(3, new Cons<int>(4, new Nil<int>())))`

```csharp
public static void PrintList<T>(List<T> list)
{
    Console.Write('[');
    while (!(list is Nil<T>))
    {
        Cons<T> cons = (Cons<T>)list;
        Console.Write(cons.value + ", ");
        list = cons.next;
    }
    Console.Write("Nil]");
}

List<int> list = 
    new Cons<int>(1, new Cons<int>(3, new Cons<int>(4, new Nil<int>())));
PrintList(list); // [1, 3, 4, Nil]
```

更奇妙的是代数数据类型对应着数据类型可能的实例数量

很显然积类型的实例数量来自各个字段可能情况的组合也就是各字段实例数量相乘，而和类型的实例数量就是各种可能类型的实例数量之和

比如 `Bool` 的类型是 `1 + 1 ` 而其实例只有 `True` 和 `False` ，而 `Nat` 的类型是 `1 + 1 + 1 + ...` 其中每一个 1 都代表一个自然数，至于 `List` 的类型就是 `1 + x(1 + x(...))` 也就是 `1 + x^2 + x^3 ...` 其中 x 就是 `List` 所存对象的实例数量

## 实际运用

ADT 最适合构造树状的结构，比如解析 JSON 出的结果需要一个聚合数据结构。

```csharp
public interface JsonValue { }
public sealed class JsonBool   : JsonValue { bool value; }
public sealed class JsonInt    : JsonValue { int value; }
public sealed class JsonString : JsonValue { string value; }
public sealed class JsonArray  : JsonValue { List<JsonValue> value; }
public sealed class JsonMap    : JsonValue { Dictionary<string, JsonValue> value; }
```
