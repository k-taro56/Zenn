---
title: "C# で本当の非同期 IO"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "dotnet", "async", "io"]
published: true
---

# 非同期メソッドだからといって非同期 IO ではない

`FileStream` クラスにファイルパスとファイルモードを指定しただけで `ReadAsync`、`WriteAsync` を使っても UI スレッドはブロックされませんが、**非同期 IO にはなりません**。
同期メソッドを `Task.Run` でラップしているイメージと同じになります。

`StreamReader`、`StreamWriter` クラスも上記の `FileStream` をラップしているので同様です。

非同期 IO をしているつもりでもできていなかったと残念な気持ちになりますよね。

# 非同期 IO にする

非同期 IO をするにはどうしたらよいかというと、以下の `FileStream` クラスのコンストラクターで `useAsync` を `true` にしなければいけません。

```cs
public FileStream (string path, FileMode mode, FileAccess access, FileShare share, int bufferSize, bool useAsync);
```

ちょっと呼び出しが長くなって嫌ですね。`useAsync` を切り替えられるだけのオーバーロードが欲しいですね。

というわけで自分で書くとこんな感じになります。`useAsync` のところ以外はデフォルトのコンストラクターと同じように動作します。

```cs
static FileStream FileStreamNewHelper(string path, FileMode mode, bool useAsync)
{
    return new FileStream(path, mode, mode == FileMode.Append ? FileAccess.Write : FileAccess.ReadWrite, FileShare.Read, 4096, useAsync);
}
```

`StreamReader`、`StreamWriter` クラスで非同期 IO にしたいときは上記の方法で作成したインスタンスをコンストラクターに渡すと良いです。

また `FileStream` が開かれたときの `useAsync` の状態は `IsAsync` プロパティーで確認することができます。

# 注意事項

非同期 IO が適さない場合があるということです。

https://learn.microsoft.com/ja-jp/dotnet/api/system.io.filestream.-ctor?view=net-7.0#system-io-filestream-ctor(system-string-system-io-filemode-system-io-fileaccess-system-io-fileshare-system-int32-system-boolean)

>非同期 I/O または同期 I/O のどちらを使用するかを指定します。 ただし、基になるオペレーティング システムが非同期 I/O をサポートしていないことがあります。したがって、`true` を指定しても、プラットフォームによってはハンドルが同期的に開かれることがあります。 非同期的に開いた場合、`BeginRead(Byte[], Int32, Int32, AsyncCallback, Object)` メソッドと `BeginWrite(Byte[], Int32, Int32, AsyncCallback, Object)` メソッドは、大量の読み取りまたは書き込み時にはパフォーマンスがより高くなりますが、少量の読み取りまたは書き込み時にはパフォーマンスが非常に低くなることがあります。 アプリケーションが非同期 I/O を利用するように設計されている場合は、`useAsync` パラメーターを `true` に設定します。 非同期 I/O を正しく使用すると、アプリケーションが 10 倍ほど高速化することがあります。ただし、非同期 I/O 用にアプリケーションを再設計せずに非同期 I/O を使用すると、パフォーマンスが 10 分の 1 ほど低下することがあります。

選択を誤ると最悪 100 倍差があるのはつらいですね。

また同期・非同期の選択の具体的な基準が書いてないのも困ったところです。（読み書きのサイズがバッファーサイズより大きければ非同期にするなど。）
わかる方はコメントお願いします。

# おまけ

ところで、なぜデフォルトで `useAsync` が `true` ではないのかということについては以下に書かれています。

https://stackoverflow.com/questions/37041844/async-await-and-opening-a-filestream

歴史的経緯だそうです。
