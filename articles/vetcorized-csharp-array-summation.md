---
title: "C# で始める並列化：配列の総和"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "dotnet", "simd", "assembly"]
published: true
---

# はじめに

C# で並列化によってプログラムを高速化する方法を紹介します。
複数のスレッドを使うような並列化ではなく、一度の命令で複数のデータを同時に処理できる SIMD（Single Instruction Multiple Data）を使って並列化します。
C# では `Vector256<T>` を用いることで簡単に SIMD を使うことができます。

また、JIT コンパイラーが出力するアセンブリーコードも取り扱っているため、アセンブリーコードの学習を始めたい方にもおすすめです。

# 配列の総和を求める

以下のプログラムは MIT ライセンスで使用できます。
詳細は [GitHub](https://github.com/k-taro56/ZennSampleVectorizedCSharp/blob/main/LICENSE.txt) を確認してください。

## 通常版

```csharp
public static int Sum(ReadOnlySpan<int> span)
{
    int sum = 0;

    for (int i = 0; i < span.Length; i++)
    {
        sum += span[i];
    }

    return sum;
}
```

## 並列化版

最適なコードを探すために、コードの書き方を変えて、ベンチマークの結果からどのような書き方が高速に動作するか調べるのが理想です。
しかし、ベンチマークを試す回数には限度があるので、JIT が出力するアセンブリーコードを見ながらある程度絞り込んでいます。

コメントでは変数がどのレジスターに割り当てられているかを書いています。

`Unsafe` クラスを使用しているので、実行は自己責任でお願いします。

```csharp
using System.Runtime.CompilerServices;
using System.Runtime.InteropServices;
using System.Runtime.Intrinsics;

public static int VectorizedSum(ReadOnlySpan<int> span)
{
    // eax
    nuint index = 0;

    // edx = span.Length
    // rdx = (nuint)span.Length
    nuint length = (nuint)span.Length;

    // r8d
    int sum = 0;
    // rcx
    ref int reference = ref MemoryMarshal.GetReference(span);

    if (Vector256.IsHardwareAccelerated && (nuint)Vector256<int>.Count <= length)
    {
        // r8
        nuint oneVectorAwayFromLast = length - (nuint)Vector256<int>.Count;
        // ymm0
        Vector256<int> sum256 = Vector256<int>.Zero;

        do
        {
            Vector256<int> vector = Vector256.LoadUnsafe(ref reference, index);
            sum256 += vector;
            index += (nuint)Vector256<int>.Count;
        }
        // index + (nuint)Vector256<int>.Count <= length のままだと、
        // 毎回足し算を計算してしまう。
        // span.Length - Vector256<int>.Count を変数に用意しておく必要がある。
        while (index <= oneVectorAwayFromLast);

        sum = sum256[0];
        sum += sum256[1];
        sum += sum256[2];
        sum += sum256[3];
        sum += sum256[4];
        sum += sum256[5];
        sum += sum256[6];
        sum += sum256[7];
    }

    for (; index < length; index++)
    {
        // Unsafe クラスを使用しなければ、範囲チェックが入ってしまう。
        sum += Unsafe.Add(ref reference, index);
    }

    return sum;
}
```

## アセンブリーコードの解析

[SharpLab](https://sharplab.io/) を使って出力しました。

```nasm
L0000: vzeroupper
; nuint index = 0;
L0003: xor eax, eax
; int length = span.Length;
L0005: mov edx, [rcx+8]
; nuint length = (nuint)length;
L0008: movsxd rdx, edx
; int sum = 0;
L000b: xor r8d, r8d
; ref int reference = ref MemoryMarshal.GetReference(span);
L000e: mov rcx, [rcx]
; if (Vector256.IsHardwareAccelerated && (nuint)Vector256<int>.Count <= length)
L0011: cmp rdx, 8
L0015: jb L009c
; nuint oneVectorAwayFromLast = length - (nuint)Vector256<int>.Count;
L001b: lea r8, [rdx-8]
; Vector256<int> sum256 = Vector256<int>.Zero;
L001f: vxorps ymm0, ymm0, ymm0
; do
; Vector256<int> vector = Vector256.LoadUnsafe(ref reference, index);
; sum256 += vector;
L0023: vpaddd ymm0, ymm0, [rcx+rax*4]
; index += (nuint)Vector256<int>.Count;
L0028: add rax, 8
; while (index <= oneVectorAwayFromLast);
L002c: cmp rax, r8
L002f: jbe short L0023
; スカラー値に変換。
; 組み込み関数を使うよりも高速。
L0031: vmovaps xmm1, xmm0
L0035: vmovd r8d, xmm1
L003a: vmovaps xmm1, xmm0
L003e: vpextrd r9d, xmm1, 1
L0044: add r8d, r9d
L0047: vmovaps xmm1, xmm0
L004b: vpextrd r9d, xmm1, 2
L0051: add r8d, r9d
L0054: vmovaps xmm1, xmm0
L0058: vpextrd r9d, xmm1, 3
L005e: add r8d, r9d
L0061: vextractf128 xmm1, ymm0, 1
L0067: vmovd r9d, xmm1
L006c: add r8d, r9d
L006f: vextractf128 xmm1, ymm0, 1
L0075: vpextrd r9d, xmm1, 1
L007b: add r8d, r9d
L007e: vextractf128 xmm1, ymm0, 1
L0084: vpextrd r9d, xmm1, 2
L008a: add r8d, r9d
L008d: vextractf128 xmm0, ymm0, 1
L0093: vpextrd r9d, xmm0, 3
L0099: add r8d, r9d
; do ステートメントに展開されている。
; index < length;
L009c: cmp rax, rdx
L009f: jae short L00ad
; sum += Unsafe.Add(ref reference, index);
L00a1: add r8d, [rcx+rax*4]
; index++;
L00a5: inc rax
; index < length;
L00a8: cmp rax, rdx
L00ab: jb short L00a1
L00ad: mov eax, r8d
L00b0: vzeroupper
L00b3: ret
```

# ベンチマーク結果

[BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) を使用してベンチマークを行いました。

`Vector256<int>` を使用した方がすべての長さで高速で、要素数が増えるほど差が開きます。
最大 10 倍以上高速化することができました。

|     Method | Length |         Mean |     Error |    StdDev | Ratio |
|----------- |------- |-------------:|----------:|----------:|------:|
|    General |      1 |     1.739 ns | 0.0079 ns | 0.0074 ns |  1.00 |
| Vectorized |      1 |     1.704 ns | 0.0050 ns | 0.0042 ns |  0.98 |
|            |        |              |           |           |       |
|    General |     10 |     4.030 ns | 0.0149 ns | 0.0139 ns |  1.00 |
| Vectorized |     10 |     3.825 ns | 0.0228 ns | 0.0202 ns |  0.95 |
|            |        |              |           |           |       |
|    General |    100 |    43.876 ns | 0.0805 ns | 0.0714 ns |  1.00 |
| Vectorized |    100 |     7.150 ns | 0.0405 ns | 0.0317 ns |  0.16 |
|            |        |              |           |           |       |
|    General |   1000 |   350.615 ns | 2.0672 ns | 1.9337 ns |  1.00 |
| Vectorized |   1000 |    41.376 ns | 0.0392 ns | 0.0367 ns |  0.12 |
|            |        |              |           |           |       |
|    General |  10000 | 3,519.136 ns | 1.7540 ns | 1.6407 ns |  1.00 |
| Vectorized |  10000 |   337.213 ns | 0.7954 ns | 0.6642 ns |  0.10 |

# C 言語版

C 言語版では、SIMD 命令と一対一に結び付いた SIMD 関数を使用しています。

https://zenn.dev/k-taro56/articles/simd-array-summation

# C# 並列化記事一覧

その他の C# 並列化に関する記事の一覧です。

https://zenn.dev/k_taro56/articles/simd-introduction

# ソースコード

https://github.com/k-taro56/ZennSampleVectorizedCSharp/blob/main/ArraySummation

## ベンチマーク

https://github.com/k-taro56/ZennSampleVectorizedCSharp/blob/main/Benchmark/ArraySummationBenchmark/Program.cs
