---
title: "C 言語 SIMD 入門：目次"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["c", "simd", "avx"]
published: true
---

# はじめに

この記事では AVX2 を使用したプログラミングを解説した記事の一覧を紹介します。
言語はすべて C 言語です。

# SIMD とは

SIMD は Single Instruction Multiple Data の略で、一度の命令で複数のデータを処理することができます。
プログラムを高速化する上で非常に重要な技術の一つで、圧倒的な高速化が見込めます。

SIMD は CPU によって命令セットが異なります。
x86 では SSE、AVX、Arm では NEON などがあります。
SSE では 128 ビットのデータを、AVX（AVX2）では 256 ビットのデータを 1 つの命令で処理できます。

このシリーズの記事では Haswell 世代以降の CPU で使用できる AVX2 を使用しています。
C言語からは immintrin.h ヘッダーファイルをインクルードすることで、SIMD 命令と一対一に結び付いた関数を使用することができます。
MSVC では intrin.h ヘッダーファイルをインクルードすることで SSE などにも対応できますが、MSVC でしか使用できません。

コンパイラーは MSVC を対象にして解説しています。記事によっては他のコンパイラーで動かないことがあります。

# 記事一覧

- [配列内の最小値と最大値](https://zenn.dev/k-taro56/articles/simd-min-of-max-of-array)
- [配列内の要素のインデックスを求める](https://zenn.dev/k-taro56/articles/simd-index-of-array)

# 関数一覧

今までに解説した関数の一覧です。

## 比較操作

### _mm256_cmpeq_epi32

- [配列内の要素のインデックスを求める](https://zenn.dev/k-taro56/articles/simd-index-of-array)

### _mm256_min_epi32

- [配列内の最小値と最大値](https://zenn.dev/k-taro56/articles/simd-min-of-max-of-array)

### _mm256_max_epi32

- [配列内の最小値と最大値](https://zenn.dev/k-taro56/articles/simd-min-of-max-of-array)

## 読み取り・書き込み操作

### _mm256_loadu_si256 _mm256_storeu_si256

- [配列内の最小値と最大値](https://zenn.dev/k-taro56/articles/simd-min-of-max-of-array)
- [配列内の要素のインデックスを求める](https://zenn.dev/k-taro56/articles/simd-index-of-array)

## その他の操作

### _mm256_movemask_ps

- [配列内の要素のインデックスを求める](https://zenn.dev/k-taro56/articles/simd-index-of-array)

### _mm256_set1_epi32

- [配列内の最小値と最大値](https://zenn.dev/k-taro56/articles/simd-min-of-max-of-array)
- [配列内の要素のインデックスを求める](https://zenn.dev/k-taro56/articles/simd-index-of-array)

## パックドテスト操作

### _mm256_testz_si256

- [配列内の要素のインデックスを求める](https://zenn.dev/k-taro56/articles/simd-index-of-array)

## ベクトルの型キャスト操作

### _mm256_castsi256_ps

- [配列内の要素のインデックスを求める](https://zenn.dev/k-taro56/articles/simd-index-of-array)

# 参考になるサイト

- [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) インテルの公式ドキュメントで、関数が一覧表になっていています。パフォーマンスに関する情報が記載されています。
- [インテル AVX2 命令の組み込み関数](https://jp.xlsoft.com/documents/intel/compiler/18/cpp_18_win_lin/index.htm#GUID-9E84F9C5-1711-4F59-8742-8F9DF283A472.html) こちらもインテルの公式ドキュメントです。日本語ですが、パフォーマンスについての情報はありません。
- [AVX/AVX2/AVX512 アドベントカレンダー2021イントロダクション](https://qiita.com/fukushima1981/items/66bc7265f3b678903dba) 日本語で各関数を詳細に解説しています。とてもわかりやすいですが、一部の関数だけです。

