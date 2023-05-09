---
title: "C# メソッドの戻り値にする変数は最初に宣言する"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "dotnet"]
published: true
---

# はじめに

C# で戻り値にする変数を最初に宣言することで、メソッドのパフォーマンスを改善することができます。
しかし、この差はごくわずかであり、通常は無視しても問題ありません。

# 実験

最初の変数の宣言を入れ替えた 2 つのメソッドを用意します。

```csharp
public static int SumA(int[] array)
{
    int i = 0;
    int sum = 0;

    for (; i < array.Length; i++)
    {
        sum += array[i];
    }

    return sum;
}

public static int SumB(int[] array)
{
    int sum = 0;
    int i = 0;

    for (; i < array.Length; i++)
    {
        sum += array[i];
    }

    return sum;
}
```

この例だとループ変数 `i` を `for` ステートメントの外側にすることはないと思いますが、他にいい例が思いつかなかったのでご容赦ください。

`SumA` が戻り値を最初に宣言しないパターン、 `SumB` が戻り値を最初に宣言するパターンです。

これらを [SharpLab](https://sharplab.io/) を使ってアセンブリーコードを出力します。

```nasm
SumA(Int32[])
    L0000: xor eax, eax
    L0002: xor edx, edx
    L0004: mov r8d, [rcx+8]
    L0008: test r8d, r8d
    L000b: jle short L001c
    L000d: mov r9d, eax
    L0010: add edx, [rcx+r9*4+0x10]
    L0015: inc eax
    L0017: cmp r8d, eax
    L001a: jg short L000d
    L001c: mov eax, edx
    L001e: ret

SumB(Int32[])
    L0000: xor eax, eax
    L0002: xor edx, edx
    L0004: mov r8d, [rcx+8]
    L0008: test r8d, r8d
    L000b: jle short L001c
    L000d: mov r9d, edx
    L0010: add eax, [rcx+r9*4+0x10]
    L0015: inc edx
    L0017: cmp r8d, edx
    L001a: jg short L000d
    L001c: ret
```

`SumA` のループ変数 `i`が `eax レジスター` に割り当てられてられているので、リターンの直前（`L001c` のところ）で `sum （edx レジスター）` を `eax　レジスター` に移動する必要があります。

対して `SumB` は `sum` が `eax` に割り当てられるため、リターンの直前で `eax` を移動する必要がありません。

つまり、戻り値にする変数が最初に宣言されていると、`mov` 命令が 1 つ削減されることがわかります。

ただし、`mov` 命令は非常に高速（レイテンシー 0.5）で、命令が存在はするので差がまったくないとは言えませんが、体感できるようなことはないと考えられます。（ピコ秒オーダーの世界）
