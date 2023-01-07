---
title: "C# の async メソッドで同期実行される処理"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "dotnet", "async"]
published: true
---

C# でメソッドに `async` キーワード^[`async` キーワード自体は何もしていないという話は置いておいてください。]を付けて `await` するだけでは**非同期処理にはならない**ということを聞いたので、確かめてみたいと思います。

`await` 以後の処理がコールバックに展開されるので `await` 以前は同期処理で、`await` 以後が非同期処理になります。

順を追って確認しましょう。

# await 以前の処理

```cs
using System.Diagnostics;

namespace AsyncAwaitTest;

internal class Program
{
    static async Task Main(string[] args)
    {
        Stopwatch stopwatch = Stopwatch.StartNew();

        Console.WriteLine($"Main-1 {stopwatch.ElapsedMilliseconds, 4}");

        // とりあえずメソッドを実行してあとで待てるように task によける。
        Task task = F(stopwatch);

        // まだ task と同期しているのでブロックされる。
        // await していないので、同期的に実行されているというわけでもない。
        Console.WriteLine($"Main-2 {stopwatch.ElapsedMilliseconds, 4}");

        // task の完了をここで待つ。
        await task;
        
        // いつも通り同期的に実行されるので task が完了してから実行される。
        Console.WriteLine($"Main-3 {stopwatch.ElapsedMilliseconds, 4}");
    }

    async static Task F(Stopwatch stopwatch)
    {
        Console.WriteLine($"F-1    {stopwatch.ElapsedMilliseconds, 4}");

        // 重たい処理。
        // async メソッドだが、await 以前なので同期処理になる。
        Thread.Sleep(1000);

        Console.WriteLine($"F-2    {stopwatch.ElapsedMilliseconds, 4}");

        await Task.Delay(1000);

        // これ以降のコードは非同期処理になる。

        Console.WriteLine($"F-3    {stopwatch.ElapsedMilliseconds, 4}");
    }
}
```

### 実行結果

```
Main-1    0
F-1      13
F-2    1028
Main-2 1042
F-3    2057
Main-3 2058
```

ご覧の通り `async` メソッドでも同期処理になっていますね。
では `await` 以後であれば非同期処理になるかを確認しましょう。

# await 以後の処理

```cs
using System.Diagnostics;

namespace AsyncAwaitTest;

internal class Program
{
    static async Task Main(string[] args)
    {
        Stopwatch stopwatch = Stopwatch.StartNew();

        Console.WriteLine($"Main-1 {stopwatch.ElapsedMilliseconds, 4}");

        Task task = G(stopwatch);

        // task がすぐに非同期で実行されるのでこちらもすぐに実行される。
        Console.WriteLine($"Main-2 {stopwatch.ElapsedMilliseconds, 4}");

        // task の途中になるように時間をいい感じに調整する。
        Thread.Sleep(500);

        // task とは非同期なので並列に実行される。
        Console.WriteLine($"Main-2 {stopwatch.ElapsedMilliseconds, 4}");

        await task;

        Console.WriteLine($"Main-3 {stopwatch.ElapsedMilliseconds, 4}");
    }

    static async Task G(Stopwatch stopwatch)
    {
        Console.WriteLine($"G-1    {stopwatch.ElapsedMilliseconds, 4}");

        await Task.Delay(1000);

        Console.WriteLine($"G-2    {stopwatch.ElapsedMilliseconds, 4}");
    }
}
```

### 実行結果

```
Main-1    0
G-1      13
Main-2   18
Main-2  526
G-2    1024
Main-3 1026
```

確かに非同期処理になっていますね。

# いつも非同期？

じゃあ `Task` クラスを `await` したあとは必ず非同期で実行されるの？というとそうではありません。

`await` する前に `Task.IsCompleted` プロパティーが `true` のとき、待つ必要がないので非同期で実行されません。
パフォーマンス向上のためだとは思いますが、注意が必要ですね。

```cs
using System.Diagnostics;

namespace AsyncAwaitTest;

internal class Program
{
    static async Task Main(string[] args)
    {
        Stopwatch stopwatch = Stopwatch.StartNew();

        Console.WriteLine($"Main-1 {stopwatch.ElapsedMilliseconds, 4}");

        Task task = H(stopwatch);

        Console.WriteLine($"Main-2 {stopwatch.ElapsedMilliseconds, 4}");

        await task;

        Console.WriteLine($"Main-3 {stopwatch.ElapsedMilliseconds, 4}");
    }

    static async Task H(Stopwatch stopwatch)
    {
        // すでに Task の処理が完了しているコードたちを await。
        await Task.Delay(0);
        await Task.CompletedTask;

        // 非同期処理にならない。

        Console.WriteLine($"H-1    {stopwatch.ElapsedMilliseconds, 4}");

        Thread.Sleep(1000);

        Console.WriteLine($"H-2    {stopwatch.ElapsedMilliseconds, 4}");
    }
}
```

### 実行結果

```
Main-1    0
H-1      10
H-2    1014
Main-2 1015
Main-3 1016
```

# 実際に気を付けるべきポイント

ぱっと思いつくところでは、スリープ中の HDD のファイルを開くときでしょうか。

HDD のスリープからの復帰には体感できるレベルで時間がかかりますが、私の知る限りこの部分を非同期にする手段は提供されていないと思います。

そのため読み書きを非同期にしてもファイルを開くとき（クラスを `new` するとき）に UI スレッドをブロックしてしまう可能性があります。

試しに以下のコードでスリープ中の HDD のファイルパスを指定したところ、UI スレッドがブロックされることが確認できました。

```cs
static async Task Write(string filePath, string toWrite)
{
    // この行で HDD のスリープからの復帰を待ってしまう。
    // コンストラクターなので非同期にする手段が提供されていない。
    // 一度も await していないので同期処理され、UI スレッドがブロックされてしまう。
    using StreamWriter streamWriter = new(filePath);
    
    // これ以降は問題なく非同期処理になる。
    await streamWriter.WriteAsync(toWrite);
    await streamWriter.DisposeAsync();
}
```

また複数の非同期メソッドを実行して並列実行しようとするとき、それぞれのメソッドの開始時期が想定通りにならない可能性があり、パフォーマンスが低下するおそれがあります。

# おまけ

UI スレッドがブロックされないように重たい処理を `Task.Run` でラップすることがあると思いますが、以下のコードでも目的は果たせていて、同じように非同期処理になります。

```cs
static async Task I1()
{
    await Task.Delay(1);

    Thread.Sleep(1000);
}

// 普通はこっち。
// 素直にこっちで書く方が Task をそのまま返しているのでいい。
static Task I2()
{
    return Task.Run(() =>
    {
        Thread.Sleep(1000);
    });
}
```

まあ、変なことをするのはやめましょう。
