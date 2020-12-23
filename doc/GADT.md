# 十分钟魔法练习：广义代数数据类型

### By 「玩火」，改写「CWKSC」

> 前置技能：C# 基础，[ADT](ADT.md)

在 ADT 中可以构造出如下类型：

```csharp
// 构造函数已省去
public interface Expr { }
public class IntVal  : Expr { int value; }
public class BoolVal : Expr { bool value; }
public class Add     : Expr { Expr e1, e2; }
public class Eq      : Expr { Expr e1, e2; }
```

但是这样构造有个问题，很显然 `BoolVal` 是不能相加，而这样的构造并不能防止构造出这样的东西。实际上在这种情况下 ADT 的表达能力是不足的

一个比较显然的解决办法是给 `Expr` 添加一个类型参数用于标记表达式的类型：

```csharp
// 构造函数已省去
public interface Expr { }
public interface Expr<T> : Expr { }
public class IntVal  : Expr<int>  { int value; }
public class BoolVal : Expr<bool> { bool value; }
public class Add     : Expr<int>  { Expr<int> e1, e2; }
public class Eq      : Expr<bool> { Expr<bool> e1, e2; }
```

这样就可以避免构造出两个类型为 `bool` 的表达式相加，能构造出的表达式都是类型安全的

注意到四个 `class` 的父类都不是 `Expr<T>` 而是包含参数的 `Expr` ，这和 ADT 并不一样。而这就是广义代数数据类型（Generalized Algebraic Data Type, GADT）

### Example 完整例子：

```csharp
public interface Expr { }
public interface Expr<T> : Expr { }
public class IntVal : Expr<int>
{
    public int value;
    public IntVal(int v) => value = v;
}
public class BoolVal : Expr<bool>
{
    public bool value;
    public BoolVal(bool b) => value = b;
}
public class Add : Expr<int>
{
    public Expr<int> e1, e2;
    public Add(Expr<int> e1, Expr<int> e2)
    {
        this.e1 = e1;
        this.e2 = e2;
    }
}
public class Eq : Expr<bool>
{
    public Expr<bool> e1, e2;
    public Eq(Expr<bool> e1, Expr<bool> e2)
    {
        this.e1 = e1;
        this.e2 = e2;
    }
}

// 1 + 2
Expr OnePlusTwo = new Add(new IntVal(1), new IntVal(2));

// true == false
Expr TrueEqFalse = new Eq(new BoolVal(true), new BoolVal(false));

// 1 + true
// CS1503 引数 2: 无法从 'GADT.BoolVal' 转换成 'GADT.Expr<int>'
Expr OnePlusTrue = new Add(new IntVal(1), new BoolVal(true));
                                          ^^^^^^^^^^^^^^^^^

// true == 42
// CS1503 引数 2: 无法从 'GADT.IntVal' 转换成 'GADT.Expr<bool>'
Expr TrueEq42 = new Eq(new BoolVal(true), new IntVal(42));
                                          ^^^^^^^^^^^^^^
```

### See other 参看其他：

[什么是 Haskell 中的 GADT（广义代数数据类型）？ | 黃河青山](https://colliot.org/zh/2017/11/what-is-gadt-in-haskell/)

[Generalized algebraic data type - Wikipedia](https://en.wikipedia.org/wiki/Generalized_algebraic_data_type)