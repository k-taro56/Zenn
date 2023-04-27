---
title: "C 言語 SIMD 入門：配列内の要素のインデックスを求める [線形探索]"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["c", "simd", "avx"]
published: true
---

# はじめに

この記事では C 言語で SIMD の一つである x64（amd64、x86-64）の AVX2 を使用して、配列内の要素のインデックスを求める方法を紹介します。

# SIMD とは

SIMD は Single Instruction Multiple Data の略で、一度の命令で複数のデータを処理することができます。
プログラムを高速化する上で非常に重要な技術の一つで、圧倒的な高速化が見込めます。

SIMD は CPU によって命令セットが異なります。
x86 では SSE、AVX、Arm では NEON などがあります。
SSE では 128 ビットのデータを、AVX（AVX2）では 256 ビットのデータを 1 つの命令で処理できます。

今回は Haswell 世代以降の CPU で使用できる AVX2 を使用します。
C 言語からは intrin.h ヘッダーファイルをインクルードすることで、SIMD 命令と一対一に結び付いた関数を使用することができます。

# 配列内の要素のインデックスを求める

## 汎用命令を使用したプログラム

まずは汎用命令を使用したプログラムを紹介します。

```c
// 配列 a の中から key と等しい要素のインデックスを求める関数。
int index_of(const int a[], int length, int key)
{
    for (int i = 0; i < length; i++)
    {
        if (key == a[i])
        {
            return i;
        }
    }

    return -1;
}
```

## SIMD 命令を使用したプログラム

次に SIMD 命令を使用したプログラムを紹介します。

高速化を優先するのではなく、SIMD を使ってみることに重点を置いています。
本格的に高速化したい場合はさらに工夫が必要です。

なお、1 バイトは 8 ビット、int 型は 4 バイトとします。MSVC 以外ではコンパイルできません。

```c
#include <intrin.h>

// 32 ビット整数の 8 個の要素を持つベクトルの中から、最初に 0 以外の要素が見つかったインデックスを求める関数。
unsigned long find_first_non_zero_index_epi32(__m256i a)
{
    unsigned long index;
    __m256 floating_point_a = _mm256_castsi256_ps(a);
    int mask = _mm256_movemask_ps(floating_point_a);
    _BitScanForward(&index, mask);
    return index;
}

// 配列 a の中から key と等しい要素のインデックスを求める関数。
int index_of(const int a[], int length, int key)
{
    if (length < 0)
    {
        return -1;
    }

    int i = 0;

    __m256i key256 = _mm256_set1_epi32(key);

    // 各要素を 8 個ずつ処理。
    for (; i + 7 < length; i += 8)
    {
        __m256i a256 = _mm256_loadu_si256((__m256i*)(&a[i]));
        __m256i equals256 = _mm256_cmpeq_epi32(a256, key256);

        // 8 個の要素の中に key と等しい要素があるかどうかを判定。
        if (!_mm256_testz_si256(equals256, equals256))
        {
            return i + find_first_non_zero_index_epi32(equals256);
        }
    }

    // 残りの要素を処理。
    // ここは汎用命令。
    for (; i < length; i++)
    {
        if (key == a[i])
        {
            return i;
        }
    }

    return -1;
}
```

# 各関数解説

この記事で取り上げている関数の簡単な説明と、それらがどのように動作するかを示す簡単なコード例を提供します。

## __m256i

初めに関数ではありませんが、__m256i について説明します。
__m256i は大きさが 256 ビットで、複数の整数を格納するための型です。
__m256i の中に入っている整数型は決まっておらず、どの関数を使用するかによって中身（要素）の整数型が決まります。
要素が int 型であれば 8 個格納することができます。

## __m256

こちらも関数ではなく型です。
__m256 は大きさが 256 ビットで、8 個の単精度浮動小数点数（IEEE754 binary32）を格納するための型です。
__m256i とは異なり中に入っている型は決まっています。

## _mm256_loadu_si256 _mm256_storeu_si256

_mm256_loadu_si256 と _mm256_storeu_si256 は __m256i 型のデータをメモリーに読み込む・書き込むための関数です。

__mm256i の各要素を直接調べることはできません。
そこでこれらの関数を使用して、__m256i 型のデータを配列に変換して各要素がどうなっているか確認します。

_mm256_storeu_si256 は今回のプログラムには登場しませんが、以降の解説で使用します。

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

## _mm256_movemask_epi32

この関数は実際には存在しないため、代わりに _mm256_castsi256_ps と _mm256_movemask_ps を組み合わせて使用します。
もし存在していたら、 __m256i 型中の 32 ビット符号付整数型の符号だけを取り出し、int 型に変換して返すでしょう。
ここで使用している関数の詳細は後述します。

```c
int _mm256_movemask_epi32(__m256i a)
{
    __m256 floating_point_a = _mm256_castsi256_ps(a);
    return _mm256_movemask_ps(floating_point_a);
}
```

## _mm256_castsi256_ps

_mm256_castsi256_ps は __m256i 型を __m256 型に変換します。
各要素は浮動小数点数にキャストされません。型を読み替えるだけです。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
float b[8];

// 配列 a を __m256i 型の vector に書き込む。
__m256i vector = _mm256_loadu_si256((__m256i*)a);
//  __m256i 型 を __m256 型に読み替える。（キャストはしない。）
__m256 floating_point_vector = _mm256_castsi256_ps(vector);
// floating_point_vector の内容を配列 b に書き込む。
_mm256_storeu_ps(b, floating_point_vector);
// 配列 b は意味をなさず、以下のようには**ならない**。
// b[0]=1.0, b[1]=2.0, b[2]=3.0, b[3]=4.0, b[4]=5.0, b[5]=6.0, b[6]=7.0, b[7]=8.0
```

汎用命令で書くと以下のようになります。
```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
float b[8];

for (int i = 0; i < 8; i++)
{
    b[i] = *(float*)(int*)&a[i];
}
// 配列 b の各要素は意味をなさない。
```

## _mm256_movemask_ps

_mm256_movemask_ps は __m256 型のデータの各要素の最上位ビット（符号ビット）を取り出して 8 ビットのデータを作成します。

```c
float a[8] = {1.0, -2.0, 3.0, 4.0, 5.0, -6.0, 7.0, -8.0};

// _mm256_loadu_si256 の浮動小数点数版。
__m256 a256 = _mm256_loadu_ps(a);
int　result = _mm256_movemask_ps(a256);
// result=0b10100010
```

汎用命令で書くと以下のようになります。

```c
float a[8] = {1.0, -2.0, 3.0, 4.0, 5.0, -6.0, 7.0, -8.0};
int result = 0;

for (int i = 0; i < 8; i++)
{
    if (a[i] < 0.0)
    {
        result |= (1 << i);
    }
}
// result=0b10100010
```

## _mm256_set1_epi32

_mm256_set1_epi32 は指定した 1 つの int 型の値を 8 個コピーして __m256i 型のデータを作成します。

```c
int value = 1;
int result[8];

__m256i a256 = _mm256_set1_epi32(value);
_mm256_storeu_si256((__m256i*)result, a256);
// result[0]=1, result[1]=1, result[2]=1, result[3]=1, result[4]=1, result[5]=1, result[6]=1, result[7]=1
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

## _mm256_cmpeq_epi32

_mm256_cmpeq_epi32 は 2 つの __m256i 型のデータの各要素を比較して、等しい場合は 1、等しくない場合は 0 を返します。
__m256i 型の要素は int 型として扱います。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {1, 0, 3, 6, 5, 10, 15, 8};
int result[8];

__m256i a256 = _mm256_loadu_si256((__m256i*)a);
__m256i b256 = _mm256_loadu_si256((__m256i*)b);

__m256i equals256 = _mm256_cmpeq_epi32(a256, b256);
_mm256_storeu_si256((__m256i*)result, equals256);
// result[0]=1, result[1]=0, result[2]=1, result[3]=0, result[4]=1, result[5]=0, result[6]=0, result[7]=1
```

汎用命令で書くと以下のようになります。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {1, 0, 3, 6, 5, 10, 15, 8};
int result[8];

for (int i = 0; i < 8; i++)
{
    if (a[i] == b[i])
    {
        result[i] = 1;
    }
    else
    {
        result[i] = 0;
    }
}
// result[0]=1, result[1]=0, result[2]=1, result[3]=0, result[4]=1, result[5]=0, result[6]=0, result[7]=1
```

## _mm256_testz_si256

_mm256_testz_si256 は 2 つの __m256i 型のデータをビット単位で論理積を計算して、すべてのビットが 0 になったかを返します。
すべてのビットが 0 になった場合は 1、それ以外の場合は 0 を返します。

2 つの __m256i 型が等しいかどうかを判定する場合に使用できます。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {1, 0, 3, 6, 5, 10, 15, 8};

__m256i a256 = _mm256_loadu_si256((__m256i*)a);
__m256i b256 = _mm256_loadu_si256((__m256i*)b);

int result = _mm256_testz_si256(a256, b256);
// result=0
result = _mm256_testz_si256(a256, a256);
// result=1
```

汎用命令で書くと以下のようになります。

```c
int a[8] = {1, 2, 3, 4, 5, 6, 7, 8};
int b[8] = {1, 0, 3, 6, 5, 10, 15, 8};

int result = 1;

for (int i = 0; i < 8; i++)
{
    if ((a[i] & b[i]) != 0)
    {
        result = 0;
        break;
    }
}
// result=0

result = 1;

for (int i = 0; i < 8; i++)
{
    if ((a[i] & a[i]) != 0)
    {
        result = 0;
        break;
    }
}
// result=1
```

# ソースコード

https://github.com/k-taro56/ZennSimdSample/tree/main/IndexOf

# 参考になるサイト

- [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) インテルの公式ドキュメントで、関数が一覧表になっていています。パフォーマンスに関する情報が記載されています。
- [概要: インテル AVX 命令の組み込み関数](https://www.xlsoft.com/jp/products/intel/compilers/ccl/12/ug/hh_goto.htm#intref_cls/common/intref_avx_overview.htm) こちらもインテルの公式ドキュメントです。日本語ですがパフォーマンスについての情報はありません。
- [AVX/AVX2/AVX512 アドベントカレンダー2021イントロダクション](https://qiita.com/fukushima1981/items/66bc7265f3b678903dba) 日本語で各関数を詳細に解説しています。とてもわかりやすいですが、一部の関数だけです。
