# 十分鐘魔法練習：代數數據類型

### By 「玩火」，改寫「CWKSC」

> 前置技能：C# 基礎

## Product type 積類型

積類型是指同時包括多個值的類型，例如 C# 中的 class 包括多個字段：


```csharp
public class Student
{
    string name;
    int id;
}
```

 `Student` 的類型中既有 `string` 類型的值也有 `int` 類型的值，可以表示為 `string` 和 `int` 的「積」，即 `string * int`

## Sum type 和類型

和類型是指可以是某一些類型之一的類型，在 C# 中可以用繼承來表示：

```csharp
public interface ISchoolPerson { }
public class Student : ISchoolPerson
{
    string name;
    int id;
}
public class Teacher : ISchoolPerson
{
    string name;
    string office;
}
```

`SchoolPerson` 可能是 `Student` 也可能是 `Teacher` ，可以表示為 `Student` 和 `Teacher` 的「和」，即 `String * int + String * String` 。而使用時只需要用 C# 中的 `is` 就能知道當前的 `StudentPerson` 具體是 `Student` 還是 `Teacher`

```csharp
ISchoolPerson student = new Student();
ISchoolPerson teacher = new Teacher();

(student is ISchoolPerson) // True
(student is Student) // True
(student is Teacher) // False

(teacher is ISchoolPerson) // True
(teacher is Student) // False
(teacher is Teacher) // True
```

## ADT, Algebraic Data Type 代數數據類型

由和類型與積類型組合構造出的類型就是代數數據類型，其中代數指的就是和與積的操作

### Bool 布爾

利用和類型的枚舉特性與積類型的組合特性，我們可以構造出 C# 中本來很基礎的基礎類型，比如枚舉布爾的兩個量來構造布爾類型：

```csharp
public interface Bool { }
public class True : Bool { }
public class False : Bool { }
```

然後用 `t is True` 就可以用來判定 `t` 作為 `Bool` 的值是不是 `True`

`enum` 也有相同的效果：

```csharp
public enum Bool { True, False }
```

### Natural number 自然數

比如利用 S 的數量表示的自然數：

```csharp
public interface Nat { }
public class Z : Nat { }
public class S : Nat
{
    public Nat value;
    public S(Nat v) { value = v; }
}
```

這裡提一下自然數的皮亞諾構造，一個自然數要么是 0 (也就是上面的 `Z` ) 要么是比它小一的自然數 +1 (也就是上面的 `S` ) ，例如 3 可以用 `new S(new S(new S(new Z))` 來表示

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

### Linked List 鏈表 / 連結串列

再比如鏈表：

```csharp
public interface List<T> { }
public class Nil<T> : List<T> { }
public class Cons<T> : List<T> {
    public T value;
    public List<T> next;
    public Cons(T v, List<T> n) {
        value = v;
        next = n;
    }
}
```

`[1, 3, 4]` 就表示為 `new Cons<int>(1, new Cons<int>(3, new Cons<int>(4, new Nil<int>())))`

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

更奇妙的是代數數據類型對應著數據類型可能的實例數量

很顯然積類型的實例數量來自各個字段可能情況的組合也就是各字段實例數量相乘，而和類型的實例數量就是各種可能類型的實例數量之和

比如 `Bool` 的類型是 `1 + 1 ` 而其實例只有 `True` 和 `False` ，而 `Nat` 的類型是 `1 + 1 + 1 + ...` 其中每一個1 都代表一個自然數，至於 `List` 的類型就是 `1 + x(1 + x(...))` 也就是 `1 + x^2 + x^3 ...` 其中 `x` 就是 `List` 所存對象的實例數量

## 實際運用

ADT 最適合構造樹狀的結構，比如解析 JSON 出的結果需要一個聚合數據結構。

```csharp
public interface JsonValue { }
public class JsonBool : JsonValue { bool value; }
public class JsonInt : JsonValue { int value; }
public class JsonString : JsonValue { string value; }
public class JsonArray : JsonValue { List<JsonValue> value; }
public class JsonMap : JsonValue { Dictionary<string, JsonValue> value; }
```