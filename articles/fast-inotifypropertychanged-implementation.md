---
title: "最速の INotifyPropertyChanged 実装"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wpf", "uwp", "winui","csharp","dotnet"]
published: true
---

最速かつ無駄のない `INotifyPropertyChanged` 実装を紹介します。

まあ `Prism` や `ReactiveProperty` などの MVVM フレームワークを使用していれば関係ないのですが。
ただコードの生産効率を犠牲にしてでも、最大のパフォーマンスを出したい場合はこちらの実装が最適になります。

# 一般的な実装

インターネットで検索するとだいたい以下のような例が出てくるかと思います。

```cs
using System.ComponentModel;
using System.Runtime.CompilerServices;

public class Class : INotifyPropertyChanged
{
    private int _value;

    public int Value
    {
        get => _value;
        set
        {
            _value = value;
            NotifyPropertyChanged(nameof(Value));
        }
    }
    
    // Notify ではなく On/Raise のような名前のときもある。
    protected void NotifyPropertyChanged([CallerMemberName] string? caller = null)
    {
        PropertyChanged?.Invoke(this, new(caller));
    }

    public event PropertyChangedEventHandler? PropertyChanged;
}
```

:::message
SetProperty メソッドや BindableBase クラスなどがあると書きやすくなりますがこの記事では触れません。
:::

`nameof` 演算子や `CallerMemberName` 属性を使っていて便利ですね。

しかしパフォーマンスの面ではさらに突き詰めることができます。
通知を出すたびに、まったく同じインスタンスを作成していますね。
そこで次のように書き換えます。

# 最速の実装

```cs
using System.ComponentModel;

public class Class : INotifyPropertyChanged
{
    private int _value;

    public int Value
    {
        get => _value;
        set
        {
            _value = value;
            NotifyPropertyChanged(EventArgsCache.ValuePropertyChanged);
        }
    }
    
    // string ではなく PropertyChangedEventArgs を受け取る。
    protected void NotifyPropertyChanged(PropertyChangedEventArgs e)
    {
        PropertyChanged?.Invoke(this, e);
    }

    public event PropertyChangedEventHandler? PropertyChanged;

    internal static class EventArgsCache
    {
        // 静的フィールドにすることでインスタンスの作成を 1 度限りにする。
        internal static readonly PropertyChangedEventArgs ValuePropertyChanged = new(nameof(Value));
    }
}
```

`PropertyChangedEventArgs` 型のインスタンスを `static readonly` としてキャッシュしているのがミソですね。

こうすることで `new` する時間を省略し、わずかですがヒープアロケーションを減らすことができます。
ヒープアロケーションが減るということはメモリーの消費量も減り、`GC` が走ったときの負荷も減ります。

コーディングの手間としてはキャッシュを作成するくらいで、それほど増えていないのではないでしょうか。

またこのような実装は .NET の `ObservableCollection` でも見ることができます。

https://source.dot.net/#System.ObjectModel/System/Collections/ObjectModel/ObservableCollection.cs

314 行目に `PropertyChangedEventArgs` 型のキャッシュを集めたクラスがあります。

# ベンチマーク

`BenchmarkDotNet` の結果を載せておきます。

|     Method |     Mean |     Error |    StdDev | Ratio | RatioSD |   Gen0 | Allocated | Alloc Ratio |
|----------- |---------:|----------:|----------:|------:|--------:|-------:|----------:|------------:|
| NotUseCach | 6.468 ns | 0.1510 ns | 0.3844 ns |  4.68 |    0.26 | 0.0076 |      24 B |          NA |
|    UseCach | 1.482 ns | 0.0042 ns | 0.0035 ns |  1.00 |    0.00 |      - |         - |          NA |

そもそも実行時間が一瞬なのでアプリケーション全体でどれだけ影響が出るかはわかりませんが、キャッシュを使用した方が 4 倍以上高速化されていることがわかります。

ソースコードは Github にあります。

https://github.com/k-taro56/NotifyPropertyChangedBenchmark

# おまけ

ViewModel で文字列として比較するのではなく、インスタンスの比較ができるのでこの部分でも高速化できます。

```cs
public class ViewModel
{
    private Class _model;
    
    public ViewModel(Class model)
    {
        _model = model;

        // プロパティーごとに必要な処理を登録。
        _model.PropertyChanged += (sender, e) =>
        {
            // switch 文が使えなくなるのが残念。
            if(e == Class.EventArgsCache.ValuePropertyChanged)
            {
                // 処理。
            }
        };
    }
}
```
