# 十分钟魔法练习：余代数数据类型

### By 「玩火」，改写「CWKSC」

> 前置技能：C# 基础，ADT

## ADT 的局限性

很显然， ADT 可以构造任何树形的数据结构：树的节点内分支用和类型连接，层级间节点用积类型连接。

但是同样很显然 ADT 并不能搞出环形的数据结构或者说是无穷大小的数据结构。比如下面的代码：

```csharp
using IntList = ADT.List<int>;
using IntNil  = ADT.Nil<int>;
using IntCons = ADT.Cons<int>;

IntList list = new IntCons(4, new IntCons(2, new IntNil()));
ADT.PrintList(list); // [4, 2, Nil]

// CS0165 使用未指派的区域变数 'list'
IntList list = new IntCons(1, list);
                              ^^^^
```

编译器会表示 `list` 在使用时未初始化。

为什么会这样呢？ ADT 是归纳构造的，也就是说它必须从非递归的基本元素开始组合构造成更大的元素。

如果我们去掉这些基本元素那就没法凭空构造大的元素，也就是说如果去掉归纳的第一步那整个归纳过程毫无意义。

## 余代数数据类型

余代数数据类型（Coalgebraic Data Type）也就是余归纳数据类型（Coinductive Data Type），代表了自顶向下的数据类型构造思路，思考一个类型可以如何被分解从而构造数据类型。

这样在分解过程中再次使用自己这个数据类型本身就是一件非常自然的事情了。

不过在编程实现过程中使用自己需要加个惰性数据结构包裹，防止积极求值的语言无限递归生成数据。

比如一个列表可以被分解为第一项和剩余的列表：

```csharp
public class InfIntList
{
    public int head;
    public Func<InfIntList> next;
    public InfIntList(int head, Func<InfIntList> next)
    {
        this.head = head;
        this.next = next;
    }
}
```

这里的 `Func` 可以做到仅在需要 `next` 的时候才求值。使用的例子如下：

```csharp
public static InfIntList InfAlt()
{
    return new InfIntList(1, 
        () => new InfIntList(2, 
            InfAlt));
}

Console.WriteLine(InfAlt().head); // 1
Console.WriteLine(InfAlt().next().head); // 2
Console.WriteLine(InfAlt().next().next().head); // 1
Console.WriteLine(InfAlt().next().next().next().head); // 2
Console.WriteLine(InfAlt().next().next().next().next().head); // 1
```

这里的 `infAlt` 从某种角度来看实际上就是个长度为 2 的环形结构。

用这样的思路可以构造出无限大的树、带环的图等数据结构。

不过以上都是对余代数数据类型的一种模拟，实际上在对其支持良好的语言都会自动加上 `Func` 来辅助构造，同时还能处理好对无限大（其实是环）的数据结构的无限递归变换（`map`, `fold` ...）的操作。

## 无限数据结构的其他实现：

在 C# 中，可以定义递归的函数类型

对于环形结构，可以这样定义：

```csharp
public delegate (T value, InfRing<T> next) InfRing<T>();
public static (int value, InfRing<int> next) threeLengthRing() =>
    (1, () => (2, () => (3, threeLengthRing)));
```

```csharp
Console.WriteLine(threeLengthRing().value); // 1
Console.WriteLine(threeLengthRing().next().value); // 2
Console.WriteLine(threeLengthRing().next().next().value); // 3
Console.WriteLine(threeLengthRing().next().next().next().value); // 1
Console.WriteLine(threeLengthRing().next().next().next().next().value); // 2
Console.WriteLine(threeLengthRing().next().next().next().next().next().value); // 3

    1 --> 2 --> 3 
    ^           |
    |           |
    +-----------+
```

有向树/图：

```csharp
public delegate (T value, List<InfTree<T>> nexts) InfTree<T>();
public static (int value, List<InfTree<int>> nexts) tree(int x) => 
    (x, new List<InfTree<int>>{
        () => tree(x += 1),
        () => tree(x += 2) });
```

```csharp
Console.WriteLine(tree(1).value); // 1
Console.WriteLine(tree(1).nexts[0]().value); // 2
Console.WriteLine(tree(1).nexts[1]().value); // 3
Console.WriteLine(tree(1).nexts[0]().nexts[0]().value); // 3
Console.WriteLine(tree(1).nexts[0]().nexts[1]().value); // 4
Console.WriteLine(tree(1).nexts[1]().nexts[0]().value); // 4
Console.WriteLine(tree(1).nexts[1]().nexts[1]().value); // 5

              1
            /   \
          /       \
        2           3
      /   \       /   \
    3       4   4       5
  /  \    /  \ /  \    /  \
...  ... ... ...  ... ... ...

              x
            /   \
          /       \
       x + 1     x + 2
       /   \     /   \
     /       \ /       \
   ...      ......      ...
```