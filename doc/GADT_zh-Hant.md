# 十分鐘魔法練習：廣義代數數據類型

### By 「玩火」，改寫「CWKSC」

> 前置技能：C# 基礎，[ADT](ADT_zh_Hant.md)

在 ADT 中可以構造出如下類型：

```csharp
// 構造函數已省去
public interface Expr { }
public class IntVal : Expr { int value; }
public class BoolVal : Expr { bool value; }
public class Add : Expr { Expr e1, e2; }
public class Eq : Expr { Expr e1, e2; }
```

但是這樣構造有個問題，很顯然 `BoolVal` 是不能相加，而這樣的構造並不能防止構造出這樣的東西。實際上在這種情況下 ADT 的表達能力是不足的

一個比較顯然的解決辦法是給 `Expr` 添加一個類型參數用於標記表達式的類型：

```csharp
// 構造函數已省去
public interface Expr { }
public interface Expr<T> : Expr { }
public class IntVal : Expr<int> { int value; }
public class BoolVal : Expr<bool> { bool value; }
public class Add : Expr<int> { Expr<int> e1, e2; }
public class Eq : Expr<bool> { Expr<bool> e1, e2; }
```

這樣就可以避免構造出兩個類型為 `bool` 的表達式相加，能構造出的表達式都是類型安全的

注意到四個 `class` 的父類都不是 `Expr<T>` 而是包含參數的 `Expr` ，這和 ADT 並不一樣。而這就是廣義代數數據類型（Generalized Algebraic Data Type, GADT）

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
// CS1503 引數 2: 無法從 'GADT.BoolVal' 轉換成 'GADT.Expr<int>'
Expr OnePlusTrue = new Add(new IntVal(1), new BoolVal(true));
                                          ^^^^^^^^^^^^^^^^^

// true == 42
// CS1503 引數 2: 無法從 'GADT.IntVal' 轉換成 'GADT.Expr<bool>'
Expr TrueEq42 = new Eq(new BoolVal(true), new IntVal(42));
                                          ^^^^^^^^^^^^^^
```

### See other 參看其他：

[什麼是 Haskell 中的 GADT（廣義代數數據類型）？ | 黃河青山](https://colliot.org/zh/2017/11/what-is-gadt-in-haskell/)

[Generalized algebraic data type - Wikipedia](https://en.wikipedia.org/wiki/Generalized_algebraic_data_type)