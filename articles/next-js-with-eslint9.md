---
title: "Next.js 15 で ESLint 9（Flat config）を使う"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "eslint", "typescript"]
published: true
---

## はじめに

先日リリースされた Next.js 15 では ESLint 9 に[対応しました](https://nextjs.org/blog/next-15#eslint-9-support)。
互換性はまだ維持されているため、従来の設定ファイルのままでも動作しますが、せっかくなので Flat config を使ってみましょう。
（`create-next-app` で作成したプロジェクトでは、依然として ESLint 8 のままです。）

なお、`next/core-web-vitals`、`next/typescript` は ESLint 9 未対応のため、`fixupConfigRules` を使っています。
そのため内部的には ESLint 8 が使われています。

## インストール

以下の追加の依存関係が必要です。

```bash
pnpm install -D @eslint/compat
```

## 設定ファイル

`create-next-app --eslint --typescript` で作成したプロジェクトの `.eslintrc.json` を以下のファイルに置き換えます。

```js:eslint.config.cjs
const { FlatCompat } = require("@eslint/eslintrc");
const { fixupConfigRules } = require("@eslint/compat");

const flatCompat = new FlatCompat();

module.exports = fixupConfigRules(
  flatCompat.extends('next/core-web-vitals'),
  flatCompat.extends('next/typescript')
);
```

## Prettier と使う

[公式ドキュメント](https://nextjs.org/docs/app/building-your-application/configuring/eslint#prettier) に記載されている、Prettier と共存できる Flat config の設定は以下の通りです。

```js:eslint.config.cjs
const { FlatCompat } = require('@eslint/eslintrc');
const { fixupConfigRules } = require('@eslint/compat');
const prettier = require('eslint-config-prettier');

const flatCompat = new FlatCompat();

module.exports = [
  ...fixupConfigRules(
    flatCompat.extends('next/core-web-vitals'),
    flatCompat.extends('next/typescript'),
  ),
  prettier,
];
```

## おわりに

従来の設定ファイルはいずれ使えなくなってしまいます。
この記事が移行の一助になれば幸いです。
