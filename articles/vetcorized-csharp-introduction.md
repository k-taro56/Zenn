---
title: "C# で始める並列化：目次"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "dotnet", "simd", "assembly"]
published: true
---

# はじめに

C# で並列化によってプログラムを高速化する方法を紹介します。
基本的に複数のスレッドを使うような並列化ではなく、一度の命令で複数のデータを同時に処理できる SIMD（Single Instruction Multiple Data）を使って並列化します。
C# では `Vector256<T>` を用いることで簡単に SIMD を使うことができます。

また、JIT コンパイラーが出力するアセンブリーコードも取り扱っているため、アセンブリーコードの学習を始めたい方にもおすすめです。

# 記事一覧

- [配列の総和](https://zenn.dev/k-taro56/articles/vetcorized-csharp-array-summation)

# C 言語版

C 言語版では、SIMD 命令と一対一に結び付いた SIMD 関数を使用しています。

https://zenn.dev/k_taro56/articles/simd-introduction
