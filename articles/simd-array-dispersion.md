---
title: "C 言語 SIMD 入門：配列の分散を求める"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["c", "simd", "avx"]
published: true
---

# はじめに

この記事では C 言語で SIMD の一つである x64（amd64、x86-64）の AVX2 を使用して、配列の分散を求める方法を紹介します。

# SIMD とは

SIMD は Single Instruction Multiple Data の略で、一度の命令で複数のデータを処理することができます。
プログラムを高速化する上で非常に重要な技術の一つで、圧倒的な高速化が見込めます。

SIMD は CPU によって命令セットが異なります。
x86 では SSE、AVX、Arm では NEON などがあります。
SSE では 128 ビットのデータを、AVX（AVX2）では 256 ビットのデータを 1 つの命令で処理できます。

今回は Haswell 世代以降の CPU で使用できる AVX2 を使用します。
C 言語からは intrin.h ヘッダーファイルをインクルードすることで、SIMD 命令と一対一に結び付いた関数を使用することができます。

# 配列の分散を求める

いずれも定義通りではなく、分散公式を使用しています。

## 汎用命令を使用したプログラム

まずは汎用命令を使用したプログラムを紹介します。

```c
// 配列 a の分散を求める関数。
double dispersion(const int a[], int length)
{
    int sum = 0;
    int squared_sum = 0;

    for (int i = 0; i < length; i++)
    {
        sum += a[i];
        squared_sum += a[i] * a[i];
    }

    double average = (double)sum / length;
    double squred_average = (double)squared_sum / length;

    return squred_average - (average * average);
}
```

## SIMD 命令を使用したプログラム

次に SIMD 命令を使用したプログラムを紹介します。

高速化を優先するのではなく、SIMD を使ってみることに重点を置いています。
本格的に高速化したい場合はさらに工夫が必要です。

なお、1 バイトは 8 ビット、`int` 型は 4 バイトとします。コンパイラーは MSVC を想定しています。

```c
#include <immintrin.h>

// 配列 a の分散を求める関数。
double dispersion(const int a[], int length)
{
    int i = 0;

    // 合計値を 0 で初期化。
    __m256i sum256 = _mm256_setzero_si256();
    __m256i squared_sum256 = _mm256_setzero_si256();

    // 各要素を 8 個ずつ処理。
    for (; i + 7 < length; i += 8)
    {
        __m256i a256 = _mm256_loadu_si256((__m256i*)(&a[i]));

        sum256 = _mm256_add_epi32(sum256, a256);

        __m256i squared_a256 = _mm256_mullo_epi32(a256, a256);
        squared_sum256 = _mm256_add_epi32(squared_sum256, squared_a256);
    }

    // 合計値をスカラー値に変換。
    __m256i sum256_permute = _mm256_permute2x128_si256(sum256, sum256, 1);
    __m256i result256 = _mm256_hadd_epi32(sum256, sum256_permute);
    result256 = _mm256_hadd_epi32(result256, result256);
    result256 = _mm256_hadd_epi32(result256, result256);
    int sum = _mm256_extract_epi32(result256, 0);

    __m256i squared_sum256_permute = _mm256_permute2x128_si256(squared_sum256, squared_sum256, 1);
    __m256i squared_result256 = _mm256_hadd_epi32(squared_sum256, squared_sum256_permute);
    squared_result256 = _mm256_hadd_epi32(squared_result256, squared_result256);
    squared_result256 = _mm256_hadd_epi32(squared_result256, squared_result256);
    int squared_sum = _mm256_extract_epi32(squared_result256, 0);

    // 残りの要素を処理。
    // ここは汎用命令。
    for (; i < length; i++)
    {
        sum += a[i];
        squared_sum += a[i] * a[i];
    }

    double average = (double)sum / length;
    double squared_average = (double)squared_sum / length;

    return squared_average - (average * average);
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

`_mm256_storeu_si256` は今回のプログラムには登場しませんが、以降の解説で使用します。

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

## _mm256_setzero_si256

`_mm256_setzero_si256` は、`__m256i` 型の変数を 0 で初期化する関数です。

```c
int result[8];
__m256i a256 = _mm256_setzero_si256();
_mm256_storeu_si256((__m256i*)result, a256);
// result[0]=0, result[1]=0, result[2]=0, result[3]=0,
// result[4]=0, result[5]=0, result[6]=0, result[7]=0
```

汎用命令で書くと以下のようになります。

```c
int a[8];

for (int i = 0; i < 8; i++)
{
    a[i] = 0;
}
// a[0]=0, a[1]=0, a[2]=0, a[3]=0, a[4]=0, a[5]=0, a[6]=0, a[7]=0
```

## _mm256_add_epi32

`_mm256_add_epi32` は、`__m256i` 型の変数の各要素を足し合わせる関数です。
`__m256i` 型の要素は `int` 型として扱います。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int result[8];

__m256i a256 = _mm256_loadu_si256((__m256i*)a);
__m256i b256 = _mm256_loadu_si256((__m256i*)b);

__m256i result256 = _mm256_add_epi32(a256, b256);
_mm256_storeu_si256((__m256i*)result, result256);
// result[0]=2, result[1]=4, result[2]=6, result[3]=8,
// result[4]=10, result[5]=12, result[6]=14, result[7]=16
```

汎用命令で書くと以下のようになります。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int result[8];

for (int i = 0; i < 8; i++)
{
    result[i] = a[i] + b[i];
}
// result[0]=2, result[1]=4, result[2]=6, result[3]=8,
// result[4]=10, result[5]=12, result[6]=14, result[7]=16
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

## _mm256_permute2x128_si256

`_mm256_permute2x128_si256` は、`__m256i` 型の変数の上位 128 ビットと下位 128 ビットを組み合わせて新しい `__m256i` 型の変数を作る関数です。
組み合わせ方は、第 3 引数で指定します。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {9, 10, 11, 12, 13, 14, 15, 16};
int result[8];

__m256i a256 = _mm256_loadu_si256((__m256i*)a);
__m256i b256 = _mm256_loadu_si256((__m256i*)b);

__m256i result256 = _mm256_permute2x128_si256(a256, b256, 1);
_mm256_storeu_si256((__m256i*)result, result256);
// result[0]=5, result[1]=6, result[2]=7, result[3]=8,
// result[4]=1, result[5]=2, result[6]=3, result[7]=4
```

汎用命令で書くと以下のようになります。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {9, 10, 11, 12, 13, 14, 15, 16};
int result[8];

// _mm256_permute2x128_si256 の 第 3 引数が 1 の場合
for (int i = 0; i < 4; i++)
{
    result[i] = a[i + 4];
    result[i + 4] = a[i];
}
// result[0]=5, result[1]=6, result[2]=7, result[3]=8,
// result[4]=1, result[5]=2, result[6]=3, result[7]=4
```

## _mm256_hadd_epi32

`_mm256_hadd_epi32` は、`__m256i` 型の変数の隣り合う 2 つの要素を足し合わせる関数です。
`__m256i` 型の要素は `int` 型として扱います。

`__m256i` 型には横のつながりだと 128 ビットの壁があり、128 ビットごとに処理が行われます。
思った通りの振る舞いではない可能性があるので注意してください。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {9, 10, 11, 12, 13, 14, 15, 16};
int result[8];

__m256i a256 = _mm256_loadu_si256((__m256i*)a);
__m256i b256 = _mm256_loadu_si256((__m256i*)b);

__m256i result256 = _mm256_hadd_epi32(a256, b256);
_mm256_storeu_si256((__m256i*)result, result256);
// result[0]=3, result[1]=7, result[2]=19, result[3]=23,
// result[4]=11, result[5]=15, result[6]=27, result[7]=31
```

汎用命令で書くと以下のようになります。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {9, 10, 11, 12, 13, 14, 15, 16};
int result[8];

result[0] = a[0] + a[1];
result[1] = a[2] + a[3];

result[2] = b[0] + b[1];
result[3] = b[2] + b[3];

// 128 ビットの壁。

result[4] = a[4] + a[5];
result[5] = a[6] + a[7];

result[6] = b[4] + b[5];
result[7] = b[6] + b[7];
// result[0]=3, result[1]=7, result[2]=19, result[3]=23,
// result[4]=11, result[5]=15, result[6]=27, result[7]=31
```

## _mm256_extract_epi32

`_mm256_extract_epi32` は、`__m256i` 型の変数の指定した要素を取り出す関数です。
`__m256i` 型の要素は `int` 型として扱います。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int value = 0;

__m256i a256 = _mm256_loadu_si256((__m256i*)a);

int result = _mm256_extract_epi32(a256, value);
// result=1
```

汎用命令で書くと以下のようになります。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int value = 0;

int result = a[value];
// result=1
```

# ソースコード

https://github.com/k-taro56/ZennSimdSample/tree/main/ArrayDispersion

# SIMD 解説記事一覧

https://zenn.dev/k_taro56/articles/simd-introduction

# 参考になるサイト

- [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) インテルの公式ドキュメントで、関数が一覧表になっていています。パフォーマンスに関する情報が記載されています。
- [インテル AVX2 命令の組み込み関数](https://jp.xlsoft.com/documents/intel/compiler/18/cpp_18_win_lin/index.htm#GUID-9E84F9C5-1711-4F59-8742-8F9DF283A472.html) こちらもインテルの公式ドキュメントです。日本語ですが、パフォーマンスについての情報はありません。
- [AVX/AVX2/AVX512 アドベントカレンダー2021イントロダクション](https://qiita.com/fukushima1981/items/66bc7265f3b678903dba) 日本語で各関数を詳細に解説しています。とてもわかりやすいですが、一部の関数だけです。
