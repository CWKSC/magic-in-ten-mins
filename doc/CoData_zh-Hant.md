# 十分鐘魔法練習：餘代數數據類型

### By 「玩火」，改寫「CWKSC」

> 前置技能：C# 基礎，[ADT](ADT_zh_Hant.md)

## ADT 的局限性

很顯然， ADT 可以構造任何樹形的數據結構：樹的節點內分支用和類型連接，層級間節點用積類型連接。

但是同樣很顯然 ADT 並不能搞出環形的數據結構或者說是無窮大小的數據結構。比如下面的代碼：

```csharp
using IntList = ADT.List<int>;
using IntNil = ADT.Nil<int>;
using IntCons = ADT.Cons<int>;

IntList list = new IntCons(4, new IntCons(2, new IntNil()));
ADT.PrintList(list); // [4, 2, Nil]

// CS0165 使用未指派的區域變數 'list'
IntList list = new IntCons(1, list);
                              ^^^^
```

編譯器會表示 `list` 在使用時未初始化。

為什麼會這樣呢？ ADT 是歸納構造的，也就是說它必須從非遞歸的基本元素開始組合構造成更大的元素。

如果我們去掉這些基本元素那就沒法憑空構造大的元素，也就是說如果去掉歸納的第一步那整個歸納過程毫無意義。

## 餘代數數據類型

餘代數數據類型（Coalgebraic Data Type）也就是餘歸納數據類型（Coinductive Data Type），代表了自頂向下的數據類型構造思路，思考一個類型可以如何被分解從而構造數據類型。

這樣在分解過程中再次使用自己這個數據類型本身就是一件非常自然的事情了。

不過在編程實現過程中使用自己需要加個惰性數據結構包裹，防止積極求值的語言無限遞歸生成數據。

比如一個列表可以被分解為第一項和剩餘的列表：

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

這裡的 `Func` 可以做到僅在需要 `next` 的時候才求值。使用的例子如下：

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

這裡的 `infAlt` 從某種角度來看實際上就是個長度為 2 的環形結構。

用這樣的思路可以構造出無限大的樹、帶環的圖等數據結構。

不過以上都是對余代數數據類型的一種模擬，實際上在對其支持良好的語言都會自動加上`Func` 來輔助構造，同時還能處理好對無限大（其實是環）的數據結構的無限遞歸變換（`map`, `fold` ...）的操作。

## 無限數據結構的其他實現：

在 C# 中，可以定義遞歸的函數類型

對於環形結構，可以這樣定義：

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

有向樹/圖：

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