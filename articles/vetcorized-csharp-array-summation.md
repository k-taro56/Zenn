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
    int sum = 0;

    // edx
    nuint index = 0;

    // r8d = span.Length
    // r8
    nuint length = (nuint)span.Length;

    // rcx
    ref int reference = ref MemoryMarshal.GetReference(span);

    if (Vector256.IsHardwareAccelerated && (nuint)Vector256<int>.Count <= length)
    {
        // rax
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
; int sum = 0;
L0003: xor eax, eax
; nuint index = 0;
L0005: xor edx, edx
; int length = span.Length;
L0007: mov r8d, [rcx+8]
; nuint length = (nuint)length;
L000b: movsxd r8, r8d
; ref int reference = ref MemoryMarshal.GetReference(span);
L000e: mov rcx, [rcx]
; if (Vector256.IsHardwareAccelerated && (nuint)Vector256<int>.Count <= length)
L0011: cmp r8, 8
L0015: jb L009b
; nuint oneVectorAwayFromLast = length - (nuint)Vector256<int>.Count;
L001b: lea rax, [r8-8]
; Vector256<int> sum256 = Vector256<int>.Zero;
L001f: vxorps ymm0, ymm0, ymm0
; do
; Vector256<int> vector = Vector256.LoadUnsafe(ref reference, index);
; sum256 += vector;
L0023: vpaddd ymm0, ymm0, [rcx+rdx*4]
; index += (nuint)Vector256<int>.Count;
L0028: add rdx, 8
; while (index <= oneVectorAwayFromLast);
L002c: cmp rdx, rax
L002f: jbe short L0023
L0031: vmovaps xmm1, xmm0
; スカラー値に変換。
; 組み込み関数を使うよりも高速。
L0035: vmovd eax, xmm1
L0039: vmovaps xmm1, xmm0
L003d: vpextrd r9d, xmm1, 1
L0043: add eax, r9d
L0046: vmovaps xmm1, xmm0
L004a: vpextrd r9d, xmm1, 2
L0050: add eax, r9d
L0053: vmovaps xmm1, xmm0
L0057: vpextrd r9d, xmm1, 3
L005d: add eax, r9d
L0060: vextractf128 xmm1, ymm0, 1
L0066: vmovd r9d, xmm1
L006b: add eax, r9d
L006e: vextractf128 xmm1, ymm0, 1
L0074: vpextrd r9d, xmm1, 1
L007a: add eax, r9d
L007d: vextractf128 xmm1, ymm0, 1
L0083: vpextrd r9d, xmm1, 2
L0089: add eax, r9d
L008c: vextractf128 xmm0, ymm0, 1
L0092: vpextrd r9d, xmm0, 3
L0098: add eax, r9d
; do ステートメントに展開されている。
; index < length;
L009b: cmp rdx, r8
L009e: jae short L00ab
; sum += Unsafe.Add(ref reference, index);
L00a0: add eax, [rcx+rdx*4]
; index++;
L00a3: inc rdx
L00a6: cmp rdx, r8
L00a9: jb short L00a0
L00ab: vzeroupper
L00ae: ret
```

# ベンチマーク結果

[BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) を使用してベンチマークを行いました。

`Vector256<int>` を使用した方がすべての長さで高速で、要素数が増えるほど差が開きます。
最大 9 倍以上高速化することができました。

|     Method | Length |         Mean |     Error |    StdDev | Ratio |
|----------- |------- |-------------:|----------:|----------:|------:|
|    General |      1 |     1.683 ns | 0.0201 ns | 0.0178 ns |  1.00 |
| Vectorized |      1 |     1.648 ns | 0.0059 ns | 0.0055 ns |  0.98 |
|            |        |              |           |           |       |
|    General |     10 |     4.236 ns | 0.0203 ns | 0.0190 ns |  1.00 |
| Vectorized |     10 |     3.756 ns | 0.0139 ns | 0.0130 ns |  0.89 |
|            |        |              |           |           |       |
|    General |    100 |    43.808 ns | 0.0279 ns | 0.0261 ns |  1.00 |
| Vectorized |    100 |     6.958 ns | 0.0161 ns | 0.0151 ns |  0.16 |
|            |        |              |           |           |       |
|    General |   1000 |   349.722 ns | 1.3974 ns | 1.3071 ns |  1.00 |
| Vectorized |   1000 |    41.432 ns | 0.0199 ns | 0.0166 ns |  0.12 |
|            |        |              |           |           |       |
|    General |  10000 | 3,516.904 ns | 1.1052 ns | 0.9797 ns |  1.00 |
| Vectorized |  10000 |   390.960 ns | 0.4707 ns | 0.3675 ns |  0.11 |

# C 言語版

C 言語版では、SIMD 命令と一対一に結び付いた SIMD 関数を使用しています。

https://zenn.dev/k-taro56/articles/simd-array-summation

# C# 並列化記事一覧

その他の C# 並列化に関する記事の一覧です。

https://zenn.dev/k_taro56/articles/simd-introduction

# おまけ

アセンブリーコードの `L0023` のアドレスオフセットの計算で、`int` 型のサイズがかけられています。
初めから `byte` 型でオフセットを持っていればこのかけ算を省略できます。
そのように書き換えたのが以下のコードです。

```csharp
public static int VectorizedByteOffset(ReadOnlySpan<int> span)
{
    int sum = 0;
    nuint byteOffset = 0;
    nuint byteLength = (nuint)span.Length * sizeof(int);
    ref int intReference = ref MemoryMarshal.GetReference(span);
    ref byte byteReference = ref Unsafe.As<int, byte>(ref intReference);

    if (Vector256.IsHardwareAccelerated && (nuint)Vector256<byte>.Count <= byteLength)
    {
        nuint awayFromLastElement = byteLength - (nuint)Vector256<byte>.Count;
        Vector256<int> sum256 = Vector256<int>.Zero;

        do
        {
            Vector256<byte> vector = Vector256.LoadUnsafe(ref byteReference, byteOffset);
            sum256 += Unsafe.As<Vector256<byte>, Vector256<int>>(ref vector);
            byteOffset += (nuint)Vector256<byte>.Count;
        } while (byteOffset <= awayFromLastElement);

        sum = sum256[0];
        sum += sum256[1];
        sum += sum256[2];
        sum += sum256[3];
        sum += sum256[4];
        sum += sum256[5];
        sum += sum256[6];
        sum += sum256[7];
    }

    for (; byteOffset < byteLength; byteOffset += sizeof(int))
    {
        sum += Unsafe.AddByteOffset(ref intReference, byteOffset);
    }

    return sum;
}
```

ただ、ベンチマークをとったところ特に結果は変わりませんでした。
机上では速くなるはずですが、実際にはそうならず、コードも読みにくくなっているのでおまけとしました。

# ソースコード

https://github.com/k-taro56/ZennSampleVectorizedCSharp/blob/main/ArraySummation

## ベンチマーク

https://github.com/k-taro56/ZennSampleVectorizedCSharp/blob/main/Benchmark/ArraySummationBenchmark/Program.cs
