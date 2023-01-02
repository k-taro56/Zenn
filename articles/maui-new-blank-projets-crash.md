---
title: "MAUI の新規プロジェクトがクラッシュする"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["maui", "blazor"]
published: true
---

# 症状

Visual Studio の新規 .NET MAUI App プロジェクトテンプレートを変更せず、そのままビルドしただけにもかかわらず起動しません。ビルドは問題なく成功します。

デバッグコンソールの最後には以下の出力がありました。

```
The program '[00000] MauiApp1.exe' has exited with code 2147942405 (0x80070005).
```

`Platforms\Windows\App.xaml`を開くと以下のエラーが出ました。（ファイルを開かないとエラーは出力されません。）

```
The type 'maui:MauiWinUIApplication' was not found. Verify that you are not missing an assembly reference and that all referenced assemblies have been built.
```

MAUI Blazor の新規プロジェクトでも同じ症状です。

# 解決策

すでに Github に issue があります。

https://github.com/dotnet/maui/issues/12080

一度でよいので管理者権限で MAUI App を起動するそうです。
私はこれで解決しました。

または`WindowsApp Runtime`を再インストールするといいようです。

# 環境

 - Windows 10 Home 22H2 build 19045.2364
 - VisualStudio 17.4.3
 - .NET 7.0.101
