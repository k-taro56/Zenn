---
title: "[C 言語] SIMD 入門：配列内の最小値と最大値"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["C", "simd", "avx"]
published: true
---

# はじめに

この記事では C 言語で SIMD の一つである x64（amd64、x86-64）の AVX2 を使用して、配列内の最小値と最大値を求める方法を紹介します。

# SIMD とは

SIMD は Single Instruction Multiple Data の略で、一度の命令で複数のデータを処理することができます。
プログラムを高速化する上で非常に重要な技術の一つで、圧倒的な高速化が見込めます。

SIMD は CPU によって命令セットが異なります。
x86 では SSE、AVX、Arm では NEON などがあります。
SSE では 128 ビットのデータを、AVX（AVX2）では 256 ビットのデータを 1 つの命令で処理できます。

今回は Haswell 世代以降の CPU で使用できる AVX2 を使用します。
C 言語からは intrin.h ヘッダーファイルをインクルードすることで、SIMD 命令と一対一に結び付いた関数を使用することができます。

# 配列内の最小値と最大値を求めるプログラム

プログラムは以下のようになります。
高速化を優先するのではなく、SIMD を使ってみることに重点を置いています。
本格的に高速化したい場合はさらに工夫が必要です。

なお、1 バイトは 8 ビット、int 型は 4 バイトとします。コンパイラーは MSVC を想定しています。

```c
#include <stdio.h>
#include <limits.h>
#include <intrin.h>

// 配列 a の中から最小値を求める関数。
int min_of(const int a[], int length)
{
    int i = 0;

    // 最小値を最大の整数で初期化。
    __m256i min_value256 = _mm256_set1_epi32(INT_MAX);

    // 各要素を 8 個ずつ処理。
    for (; i + 7 < length; i += 8)
    {
        __m256i a256 = _mm256_loadu_si256((__m256i*)&a[i]);
        min_value256 = _mm256_min_epi32(min_value256, a256);
    }

    // 最小値をスカラー値に変換。
    int result[8];
    _mm256_storeu_si256((__m256i*)result, min_value256);

    int min_value = result[0];

    for (int j = 1; j < 8; j++)
    {
        if (result[j] < min_value)
        {
            min_value = result[j];
        }
    }

    // 残りの要素を処理。
    // ここは汎用命令。
    for (; i < length; i++)
    {
        if (a[i] < min_value)
        {
            min_value = a[i];
        }
    }

    // 最小値を返す。
    return min_value;
}

// 配列 a の中から最大値を求める関数。
int max_of(int a[], int length)
{
    int i = 0;

    // 最大値を最小の整数で初期化。
    __m256i max_value256 = _mm256_set1_epi32(INT_MIN);

    // 各要素を 8 個ずつ処理。
    for (; i + 7 < length; i += 8)
    {
        __m256i a256 = _mm256_loadu_si256((__m256i*)&a[i]);
        max_value256 = _mm256_max_epi32(max_value256, a256);
    }

    // 最大値をスカラー値に変換。
    int result[8];
    _mm256_storeu_si256((__m256i*)result, max_value256);

    int max_value = result[0];

    for (int j = 1; j < 8; j++)
    {
        if (result[j] > max_value)
        {
            max_value = result[j];
        }
    }

    // 残りの要素を処理。
    // ここは汎用命令。
    for (; i < length; i++)
    {
        if (a[i] > max_value)
        {
            max_value = a[i];
        }
    }

    // 最大値を返す。
    return max_value;
}
```

# 各関数解説

この記事で取り上げている関数の簡単な説明と、それらがどのように動作するかを示す簡単なコード例を提供します。

## __m256i

初めに関数ではありませんが、__m256i について説明します。
__m256i は大きさが 256 ビットで、複数の整数を格納するための型です。
__m256i の中に入っている整数型は決まっておらず、どの関数を使用するかによって中身（要素）の整数型が決まります。
要素が int 型であれば 8 個格納することができます。

## _mm256_loadu_si256 _mm256_storeu_si256

_mm256_loadu_si256 と _mm256_storeu_si256 は __m256i 型のデータをメモリーに読み込む・書き込むための関数です。

__mm256i の各要素を直接調べることはできません。
そこでこれらの関数を使用して、__m256i 型のデータを配列に変換して各要素がどうなっているか確認します。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8];
// 配列 a を __m256i 型の vector に書き込む。
__m256i vector = _mm256_loadu_si256((__m256i*)a);
// vector の内容を配列 b に書き込む。
_mm256_storeu_si256((__m256i*)b, vector);
// これで vector の各要素が何かわかる。
// b[0]=1, b[1]=2, b[2]=3, b[3]=4, b[4]=5, b[5]=6, b[6]=7, b[7]=8
```

## _mm256_min_epi32

_mm256_min_epi32 は 2 つの __m256i 型のデータの各要素ごとに最小値を求めます。
__m256i 型の要素は int 型として扱います。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {8, 7, 6, 5, 4, 3, 2, 1};
int result[8];

__m256i a256 = _mm256_loadu_si256((__m256i*)a);
__m256i b256 = _mm256_loadu_si256((__m256i*)b);
__m256i result256 = _mm256_min_epi32(a256, b256);
_mm256_storeu_si256((__m256i*)result, result256);
// result[0]=1, result[1]=2, result[2]=3, result[3]=4, result[4]=4, result[5]=3, result[6]=2, result[7]=1
```

汎用命令で書くと以下のようになります。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {8, 7, 6, 5, 4, 3, 2, 1};
int result[8];

for (int i = 0; i < 8; i++)
{
    result[i] = a[i] < b[i] ? a[i] : b[i];
}
// result[0]=1, result[1]=2, result[2]=3, result[3]=4, result[4]=4, result[5]=3, result[6]=2, result[7]=1
```

## _mm256_max_epi32

_mm256_max_epi32 は 2 つの __m256i 型のデータの各要素ごとに最大値を求めます。
__m256i 型の要素は int 型として扱います。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {8, 7, 6, 5, 4, 3, 2, 1};
int result[8];

__m256i a256 = _mm256_loadu_si256((__m256i*)a);
__m256i b256 = _mm256_loadu_si256((__m256i*)b);
__m256i result256 = _mm256_max_epi32(a256, b256);
_mm256_storeu_si256((__m256i*)result, result256);
// result[0]=8, result[1]=7, result[2]=6, result[3]=5, result[4]=5, result[5]=6, result[6]=7, result[7]=8
```

汎用命令で書くと以下のようになります。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {8, 7, 6, 5, 4, 3, 2, 1};
int result[8];

for (int i = 0; i < 8; i++)
{
    result[i] = a[i] > b[i] ? a[i] : b[i];
}
// result[0]=8, result[1]=7, result[2]=6, result[3]=5, result[4]=5, result[5]=6, result[6]=7, result[7]=8
```

# 参考になるサイト

- [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) インテルの公式ドキュメントで、関数が一覧表になっていています。パフォーマンスに関する情報が記載されています。
- [概要: インテル AVX 命令の組み込み関数](https://www.xlsoft.com/jp/products/intel/compilers/ccl/12/ug/hh_goto.htm#intref_cls/common/intref_avx_overview.htm) こちらもインテルの公式ドキュメントです。日本語ですがパフォーマンスについての情報はありません。
- [AVX/AVX2/AVX512 アドベントカレンダー2021イントロダクション](https://qiita.com/fukushima1981/items/66bc7265f3b678903dba) 日本語で各関数を詳細に解説しています。とてもわかりやすいですが、一部の関数だけです。
