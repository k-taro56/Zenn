---
title: "C 言語 SIMD 入門：行列のスカラー倍を計算する"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["c", "simd", "avx"]
published: true
---

# はじめに

この記事では C 言語で SIMD の一つである x64（amd64、x86-64）の AVX2 を使用して、行列のスカラー倍を計算する方法を紹介します。
AVX2 を使用することで従来の方法に比べて計算速度が向上し、効率的に行列のスカラー倍を計算できます。

# SIMD とは

SIMD は Single Instruction Multiple Data の略で、一度の命令で複数のデータを処理することができます。
プログラムを高速化する上で非常に重要な技術の一つで、圧倒的な高速化が見込めます。

SIMD は CPU によって命令セットが異なります。
x86 では SSE、AVX、Arm では NEON などがあります。
SSE では 128 ビットのデータを、AVX（AVX2）では 256 ビットのデータを 1 つの命令で処理できます。

今回は Haswell 世代（2013 年発売）以降の CPU で使用できる AVX2 を使用します。
C 言語からは immintrin.h ヘッダーファイルをインクルードすることで、SIMD 命令と一対一に結び付いた関数を使用することができます。

# 行列のスカラー倍を計算する

## 汎用命令を使用したプログラム

まずは汎用命令を使用したプログラムを紹介します。

```c
// 行列のスカラー倍を計算する関数。
void scalar_multiplication(int* a, int row, int column, int scalar)
{
    for (int i = 0; i < row * column; i++)
    {
        a[i] *= scalar;
    }
}
```

## SIMD 命令を使用したプログラム

次に SIMD 命令を使用したプログラムを紹介します。

高速化を優先するのではなく、SIMD を使ってみることに重点を置いています。
本格的に高速化したい場合はさらに工夫が必要です。

なお、1 バイトは 8 ビット、`int` 型は 4 バイトとします。コンパイラーは MSVC を想定しています。

```c
#include <immintrin.h>

// 行列のスカラー倍を計算する関数。
void scalar_multiplication(int* a, int row, int column, int scalar)
{
    int i = 0;

    __m256i scalar256 = _mm256_set1_epi32(scalar);

    // 8 要素ずつ計算する。
    for (; i < row * column; i += 8)
    {
        __m256i a256 = _mm256_loadu_si256((__m256i*)(&a[i]));
        __m256i product256 = _mm256_mullo_epi32(a256, scalar256);
        _mm256_storeu_si256((__m256i*)(&a[i]), product256);
    }

    // 残りの要素を処理。
    // ここは汎用命令。
    for (; i < row * column; i++)
    {
        a[i] *= scalar;
    }
}
```

# 各関数解説

この記事で取り上げている関数の簡単な説明と、それらがどのように動作するかを示す簡単なコード例を提供します。

## __m256i

初めに関数ではありませんが、`__m256i` について説明します。
`__m256i` は大きさが 256 ビットで、複数の整数を格納するための型です。
`__m256i` の中に入っている整数型は決まっておらず、どの関数を使用するかによって中身（要素）の整数型が決まります。
要素が `int` 型であれば 8 個格納することができます。

## _mm256_loadu_si256 _mm256_storeu_si256

`_mm256_loadu_si256` と `_mm256_storeu_si256` は `__m256i` 型のデータをメモリーに読み込む・書き込むための関数です。

`__mm256i` の各要素を直接調べることはできません。
そこでこれらの関数を使用して、`__m256i` 型のデータを配列に変換して各要素がどうなっているか確認します。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8];
// 配列 a を __m256i 型の vector に書き込む。
__m256i vector = _mm256_loadu_si256((__m256i*)a);
// vector の内容を配列 b に書き込む。
_mm256_storeu_si256((__m256i*)b, vector);
// これで配列 b に vector の各要素が格納され、その値を確認できる。
// b[0]=1, b[1]=2, b[2]=3, b[3]=4, b[4]=5, b[5]=6, b[6]=7, b[7]=8
```

## _mm256_set1_epi32

`_mm256_set1_epi32` は指定した 1 つの `int` 型の値を 8 個コピーして `__m256i` 型のデータを作成します。

```c
int value = 1;
int result[8];

__m256i a256 = _mm256_set1_epi32(value);
_mm256_storeu_si256((__m256i*)result, a256);
// result[0]=1, result[1]=1, result[2]=1, result[3]=1,
// result[4]=1, result[5]=1, result[6]=1, result[7]=1
```

汎用命令で書くと以下のようになります。

```c
int value = 1;
int a[8];

for (int i = 0; i < 8; i++)
{
    a[i] = value;
}
// a[0]=1, a[1]=1, a[2]=1, a[3]=1, a[4]=1, a[5]=1, a[6]=1, a[7]=1
```

## _mm256_mullo_epi32

`_mm256_mullo_epi32` は、`__m256i` 型の変数の各要素を掛け合わせる関数です。
`__m256i` 型の要素は `int` 型として扱います。

`_mm256_mul_epi32` という関数もありますが、こちらは 32 ビットの整数型を桁あふれをしにくいように 64 ビットの整数型に拡張してから掛け合わせる関数です。
今回は 32 ビットの整数型を掛け合わせるので、`_mm256_mullo_epi32` を使用します。
なぜ 64 ビットの方にシンプルな名前が付けられているかというと、汎用命令の `mul` も同様に 2 倍の大きさの結果を返すためです。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int result[8];

__m256i a256 = _mm256_loadu_si256((__m256i*)a);
__m256i b256 = _mm256_loadu_si256((__m256i*)b);

__m256i result256 = _mm256_mullo_epi32(a256, b256);
_mm256_storeu_si256((__m256i*)result, result256);
// result[0]=1, result[1]=4, result[2]=9, result[3]=16,
// result[4]=25, result[5]=36, result[6]=49, result[7]=64
```

汎用命令で書くと以下のようになります。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int result[8];

for (int i = 0; i < 8; i++)
{
    result[i] = a[i] * b[i];
}
// result[0]=1, result[1]=4, result[2]=9, result[3]=16,
// result[4]=25, result[5]=36, result[6]=49, result[7]=64
```

# ソースコード

https://github.com/k-taro56/ZennSimdSample/tree/main/ScalarMultiplication

# SIMD 解説記事一覧

https://zenn.dev/k_taro56/articles/simd-introduction

# 参考になるサイト

- [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) インテルの公式ドキュメントで、関数が一覧表になっていています。パフォーマンスに関する情報が記載されています。
- [インテル AVX2 命令の組み込み関数](https://jp.xlsoft.com/documents/intel/compiler/18/cpp_18_win_lin/index.htm#GUID-9E84F9C5-1711-4F59-8742-8F9DF283A472.html) こちらもインテルの公式ドキュメントです。日本語ですが、パフォーマンスについての情報はありません。
- [AVX/AVX2/AVX512 アドベントカレンダー2021イントロダクション](https://qiita.com/fukushima1981/items/66bc7265f3b678903dba) 日本語で各関数を詳細に解説しています。とてもわかりやすいですが、一部の関数だけです。
