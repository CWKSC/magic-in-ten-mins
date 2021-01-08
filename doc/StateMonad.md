# 十分钟魔法练习：状态单子

### By 「玩火」

> 前置技能：C# (Func, Tuple)，[HKT](HKT.md)，[Monad](Monad.md)

## 函数容器

很显然 C# 标准库中的各类容器都是可以看成是单子的， `Linq` 也给出了这些类的 `flatMap` 实现。不过在函数式的理论中单子不仅仅可以是实例意义上的容器，也可以是其他抽象意义上的容器，比如函数。

对于一个形如 ` Func<S, Complex<A>>` 形式的函数来说，我们可以把它看成包含了一个 `A` 的惰性容器，只有在给出 `S` 的时候才能知道 `A` 的值。对于这样形式的函数我们同样能写出对应的 `flatMap` ，这里就拿状态单子举例子。

## State Monad 状态单子

状态单子（State Monad）是一种可以包含一个“可变”状态的单子，我这里用了引号是因为尽管状态随着逻辑流在变化但是在内存里面实际上都是不变量。

其本质就是在每次状态变化的时候将新状态作为代表接下来逻辑的函数的输入。比如对于：

```csharp
i = i + 1;
Console.WriteLine(i);
```

可以用状态单子的思路改写成：

```csharp
((Action<int>)(v => Console.WriteLine(v)))(i + 1);
```

最简单的理解就是这样的一个包含函数的对象：

```csharp
public interface State<S> { }
public class State<S, V> : HKT<State<S>, V>, State<S>
{
    public Func<S, (S state, V value)> runState;
    public State(Func<S, (S state, V value)> f) => runState = f;
    public static State<S, V> Narrow(HKT<State<S>, V> v) => (State<S, V>)v;
}
```

最核心是 `runState` 函数对象，通过组合这个函数对象来使变化的状态在逻辑间传递。

`State` 是一个 Monad （注释中是简化的伪代码）：

```csharp
public class StateM<S> : Monad<State<S>>
{
    public HKT<State<S>, V> Pure<V>(V value) => 
        new State<S, V>(state => (state, value));

    public HKT<State<S>, B> FlatMap<A, B>(HKT<State<S>, A> a, Func<A, HKT<State<S>, B>> f) =>
        new State<S, B>(s =>
        {
            (S state, A value) = State<S, A>.Narrow(a).runState(s);
            return State<S, B>.Narrow(f(value)).runState(state);
        });
}
```

`pure` 操作直接返回当前状态和给定的值， `flatMap` 操作只需要把 `ma` 中的 `A` 取出来然后传给 `f` ，并处理好 `state` 。

仅仅这样的话 `State` 使用起来并不方便，还需要定义一些常用的操作来读取写入状态：

```csharp
// 读取 //
public static HKT<State<S>, S> Get = 
    new State<S, S>(s => (s, s));

// 写入 //
public static HKT<State<S>, S> Put(S s) => 
    new State<S, S>(any => (s, any));

// 修改 //
public static HKT<State<S>, S> Modify(Func<S, S> f) => 
    new State<S, S>(s => (f(s), s));
```

使用的话这里举个求斐波那契数列的例子：

```csharp
public static StateM<(int, int)> m = new StateM<(int, int)>();

public static State<(int, int), int> Fib(int n) => 
    n == 0 ? 
        State<(int, int), int>.Narrow(
            m.FlatMap(StateM<(int, int)>.Get,
            x => m.Pure(x.Item1))) : 
        State<(int, int), int>.Narrow(
            m.FlatMap(StateM<(int, int)>.Modify(x => (x.Item2, x.Item1 + x.Item2)),
            v => Fib(n - 1)));
```

```csharp
Console.WriteLine(Fib(0).runState((0, 1)).value); // 0
Console.WriteLine(Fib(1).runState((0, 1)).value); // 1
Console.WriteLine(Fib(2).runState((0, 1)).value); // 1
Console.WriteLine(Fib(3).runState((0, 1)).value); // 2
Console.WriteLine(Fib(4).runState((0, 1)).value); // 3
Console.WriteLine(Fib(5).runState((0, 1)).value); // 5
Console.WriteLine(Fib(6).runState((0, 1)).value); // 8
Console.WriteLine(Fib(7).runState((0, 1)).value); // 13
Console.WriteLine(Fib(8).runState((0, 1)).value); // 21
Console.WriteLine(Fib(9).runState((0, 1)).value); // 34
Console.WriteLine(Fib(10).runState((0, 1)).value); // 55
```

`Fib` 函数对应的 Haskell 代码是：

```haskell
fib :: Int -> State (Int, Int) Int
fib 0 = do
  (_, x) <- get
  pure x
fib n = do
  modify (\(a, b) -> (b, a + b))
  fib (n - 1)
```

~~看上去比 C# 版简单很多~~

## 有啥用

看到这里肯定有人会拍桌而起：求斐波那契数列我有更简单的写法！

```csharp
public static int Fib(int n) {
    int[] a = {0, 1, 1};
    for (int i = 0; i < n; i++)
        a[(i + 2) % 3] = a[(i + 1) % 3] + 
                         a[i % 3];
    return a[(n + 1) % 3];
}
```

但问题是你用变量了啊， `State Monad` 最妙的一点就是全程都是常量而模拟出了变量的感觉。

更何况你这里用了数组而不是在递归，如果你递归就会需要在 `fib` 上加一个状态参数， `State Monad` 可以做到在不添加任何函数参数的情况下在函数之间传递参数。

同时它还是纯的，也就是说是**可组合**的，把任意两个状态类型相同的 `State Monad` 组合起来并不会有任何问题，比全局变量的解决方案不知道高到哪里去。



