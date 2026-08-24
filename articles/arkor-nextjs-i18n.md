---
title: "ライブラリーを使わずに Next.js App Router で多言語対応する"
emoji: "🌐"
type: "tech"
topics: ["nextjs", "i18n", "typescript", "react", "zod", "arkor"]
published: true
publication_name: "romanark"
---

[Arkor](https://www.arkor.ai) のマーケティングサイトは英語と日本語の 2 言語に対応しています。next-intl も next-i18next も negotiator も使っていません。追加したランタイム依存はゼロで、使っているのは Next.js App Router の機能（動的セグメントと proxy）、TypeScript の `satisfies`、そしてもともと依存に入っていた Zod だけです。

この記事では、その実装を「ルーティング」「辞書」「長文コンテンツ」の 3 層に分けて紹介します。構成は Next.js 16 + Vercel へのデプロイが前提ですが、考え方自体は App Router が動く環境なら使えるはずです。

## なぜライブラリーを使わなかったか

一番大きい理由は、翻訳をエンジニア自身が管理したかったからです。

多言語対応ライブラリーの典型的な構成は `en.json` / `ja.json` のような言語別 JSON ファイルです。この形は翻訳を外部サービスや翻訳者に渡すワークフローには向いていますが、エンジニアが自分で両言語を書くチームでは逆にコストになります。コピーを 1 か所変えるたびに複数ファイルを行き来することになり、キーの追加漏れは実行時まで気づけず、画面に `home.hero.title` という生のキーが表示されて初めて発覚します。

私たちの辞書は「schema と英語と日本語が同じ TypeScript ファイルに同居するただのオブジェクト」です。これで次の 3 つが手に入ります。

- コピー変更が単一ファイルの diff になる。両言語の変更が並んで見えるので、レビューで「日本語側の直し忘れ」が一目で分かる
- キーの漏れがコンパイルエラーになる。`satisfies` で型を固定しているので、英語にだけキーを足せばその場で赤くなる
- 使う側で未定義キー参照が起きない。`t("home.hero.title")` のような文字列キーがそもそも存在せず、辞書はただの型付きオブジェクトなので、存在しないプロパティーへのアクセスは型エラー

もうひとつの理由は、要件が既成ライブラリーの前提と合わなかったことです。多言語対応するのはマーケティングページだけで、ログイン後のアプリは英語のみ。さらに英語の URL にはプレフィックスを付けたくない（`/en/pricing` ではなく `/pricing`）が、日本語は `/ja/pricing` にしたい。この非対称な要件をライブラリーの設定でねじ伏せるより、素直に自分で書くほうが短くなりました。

規模感を先に書いておくと、ロケールは 2 つ、辞書のスライスは 12 ファイル、リーフの文字列キーは約 290 個（英日それぞれ）です。全体像は次のとおりです。

```
リクエスト
   │
   ▼
src/proxy.ts ─── URL の形を [locale] セグメントに写像する
   │              /pricing    → /en/pricing に内部リライト
   │              /ja/pricing → そのまま配信
   ▼
src/app/[locale]/ ─── 全ページがこの下に置かれる
   │                   layout.tsx が <html lang={locale}> を描画
   ▼
getDictionary(locale) ─── ただのオブジェクトを返し、props で下ろす
```

## ルーティング層: [locale] セグメント + proxy

### 語彙を 1 ファイルに集める

ロケールに関する語彙はすべて `src/lib/i18n/locales.ts` にあります。依存なしの 70 行程度です。

```ts:src/lib/i18n/locales.ts
export const LOCALES = ["en", "ja"] as const;
export type Locale = (typeof LOCALES)[number];
export const DEFAULT_LOCALE = "en" satisfies Locale;

// 明示的な言語選択を記憶する cookie。
// 「存在するだけ」で Accept-Language の自動リダイレクトを無効化する。
export const LOCALE_COOKIE = "arkor_locale";

export function hasLocale(value: string): value is Locale {
  return (LOCALES as readonly string[]).includes(value);
}

// 「英語はプレフィックスなし」というルールを符号化する唯一の場所
export function localePath(locale: Locale, href: string): string {
  if (locale === DEFAULT_LOCALE || !href.startsWith("/")) return href;
  return href === "/" ? `/${locale}` : `/${locale}${href}`;
}

export async function resolveLocale(
  params: Promise<{ locale: string }>,
): Promise<Locale> {
  const { locale } = await params;
  return hasLocale(locale) ? locale : DEFAULT_LOCALE;
}
```

`hasLocale` を型述語にしているのが地味に効いていて、ルートレイアウトの `<html lang={locale}>` がキャストなしで `Locale` に絞り込まれます。

```tsx:src/app/[locale]/layout.tsx
// proxy がベアパスを /en/* にリライトするので、ユーザーに見える URL は
// プレフィックスなしのまま。両ロケールの静的シェルをプリレンダーする。
export function generateStaticParams() {
  return LOCALES.map((locale) => ({ locale }));
}

export default async function RootLayout({ children, params }: LayoutProps<"/[locale]">) {
  const { locale } = await params;
  if (!hasLocale(locale)) notFound();
  return <html lang={locale}>{/* ... */}</html>;
}
```

### proxy の 4 分岐

ページはすべて `src/app/[locale]/` の下にありますが、ユーザーに見せたい URL の形はファイルシステムと一致しません。そのギャップを埋めるのが `src/proxy.ts`（Next.js 16 における middleware の後継）です。多言語対応に関わる分岐は 4 つだけです。

```ts:src/proxy.ts（抜粋・簡略化）
const { pathname } = request.nextUrl;
const { locale, rest } = stripLocalePrefix(pathname);
const marketing = isMarketingPath(rest);

// (1) /en/* は内部リライト先であって公開 URL ではない。
//     直撃はベアパスへ 308 して 1 ページ 1 URL を保つ。
if (pathname === "/en" || pathname.startsWith("/en/")) {
  const target = redirectUrl(request);
  target.pathname = pathname.slice("/en".length) || "/";
  return NextResponse.redirect(target, 308);
}

// (2) 多言語対応しているのはマーケティングページだけ。
//     /ja/<アプリのパス> はベアパスへ 307。
if (locale === "ja" && !marketing) {
  const target = redirectUrl(request);
  target.pathname = rest;
  return NextResponse.redirect(target, 307);
}

// (3) 日本語を好むブラウザーの「初回訪問のドキュメント GET」だけ /ja へ。
if (
  locale === DEFAULT_LOCALE &&
  marketing &&
  isDocumentGet(request) &&
  !request.cookies.has(LOCALE_COOKIE) &&
  prefersJapanese(request.headers.get("accept-language"))
) {
  const target = redirectUrl(request);
  target.pathname = localePath("ja", pathname);
  return NextResponse.redirect(target, 307);
}

// (4a) /ja のマーケティングページは実在のルートなのでリライトせず配信。
if (locale === "ja") {
  return documentResponse(response);
}

// (4b) それ以外のベアパスはすべて /en/* から配信する。
//      ドキュメントも RSC フェッチも Server Action の POST も同様。
const rewriteTo = rewriteUrl(request);
rewriteTo.pathname = `/en${pathname === "/" ? "" : pathname}`;
return documentResponse(NextResponse.rewrite(rewriteTo));
```

分岐 (3) の 5 条件はどれも外せません。特に cookie の条件は「値」ではなく「存在」を見ています。ユーザーが言語切替で明示的に英語へ戻したら `arkor_locale=en` が立ち、以後 Accept-Language が日本語でも二度とリダイレクトされない、という挙動がこの 1 行で出ます。

cookie を立てるのはマーケティングページのドキュメントレスポンスを返すときで、値が変わったときだけ書きます。

```ts:src/proxy.ts（抜粋）
if (marketing && isDocumentGet(request) && request.cookies.get(LOCALE_COOKIE)?.value !== locale) {
  response.cookies.set(LOCALE_COOKIE, locale, {
    path: "/",
    maxAge: 60 * 60 * 24 * 365,
    sameSite: "lax",
    httpOnly: true, // 読むのは proxy だけなのでクライアントに晒さない
  });
}
```

### Accept-Language は 29 行で足りる

言語ネゴシエーションのために negotiator を入れる必要はありませんでした。ロケールが 2 つなら、RFC 9110 の q 値パースは手書きでファイル全体が 29 行です。

```ts:src/lib/i18n/negotiation.ts
export function prefersJapanese(header: string | null): boolean {
  if (!header) return false;
  let ja: number | null = null;
  let en: number | null = null;
  let wildcard: number | null = null;
  for (const part of header.split(",")) {
    // RFC 9110 は ";" の前後に空白を許すので、タグは split 後にも trim する
    const [rawTag = "", ...params] = part.trim().toLowerCase().split(";");
    const tag = rawTag.trim();
    const qParam = params.map((p) => p.trim()).find((p) => p.startsWith("q="));
    const q = qParam ? Number(qParam.slice(2)) : 1;
    if (Number.isNaN(q)) continue;
    if (tag === "ja" || tag.startsWith("ja-")) ja = Math.max(ja ?? 0, q);
    else if (tag === "en" || tag.startsWith("en-")) en = Math.max(en ?? 0, q);
    else if (tag === "*") wildcard = Math.max(wildcard ?? 0, q);
  }
  // "*" は「明示されなかったタグすべて」に適用される。
  // "*,ja;q=0.5" は英語を q=1 で受けるので日本語へ飛ばしてはいけない。
  // 同点はデフォルトの英語が勝つ。
  return (ja ?? 0) > (en ?? wildcard ?? 0);
}
```

判定は最後の 1 式に集約されています。厳密な `>` なので同点は英語（安全側）、`en ?? wildcard` なので明示された `en` はワイルドカードに勝ち、`Math.max` が `ja-JP,ja;q=0.9` のような複数タグを吸収します。壊れた q 値（`q=abc`）はそのエントリーごと捨てるので、既定値 1 に化けて誤リダイレクトすることもありません。この 3 性質はそれぞれユニットテストで固定しています。

### 言語切替リンクは素の `<a>`

言語切替は `<Link>` ではなく素の `<a>` にしています。ソフトナビゲーションではドキュメントレスポンスが発生しないため、proxy が cookie を立てる機会がなく、`<html lang>` もサーバー側で再描画されないからです。切替だけはフルページロードにする、というのが正解でした。

## ハマりどころ（ルーティング編）

素朴に見える proxy ですが、実際にはいくつか踏み抜きました。特に痛かった 3 つを紹介します。

**1. rewrite の URL は直してはいけない**

リダイレクト用の URL はプロキシ配下でも正しい Location を返すために `Host` ヘッダーと `X-Forwarded-Proto` で origin を補正します。ところが rewrite 用の URL に同じ補正をかけると壊れます。Next.js は rewrite 先の origin が自分のベース URL と一致するときだけ内部リライトとして扱い、一致しないと外部へのサブリクエストに silent に化けるからです。外部リクエストになった `/en/pricing` は分岐 (1) の 308 に捕まり、自己リダイレクトループが完成します。リダイレクト用（補正する）とリライト用（`request.url` の origin を保つ）で URL ビルダーを分けるのが正解です。

**2. `NextResponse.cookies.set()` は set-cookie を作り直す**

認証ライブラリー（Auth0）のローリングセッション cookie を `headers.append("set-cookie", ...)` で引き継いだ後に `response.cookies.set(LOCALE_COOKIE, ...)` を呼ぶと、`cookies.set` が自分のスナップショットから set-cookie ヘッダー全体を再構築するため、自分が作っていない値を消します。つまりセッション更新の cookie が落ち、アクティブユーザーが静かにログアウトされます。cookie の書き込みは必ず raw append より前。この順序はテストで固定しています。

**3. プリレンダーは `/en/*` のパスで走る**

ブラウザーには `/pricing` しか見えなくても、静的シェルのプリレンダーはファイルシステム上の `/en/pricing` で実行されるので、`usePathname()` は `/en/pricing` を返します。これを正規化せずに言語切替リンクを組み立てると、英語シェルに `/ja/en/pricing` が焼き込まれます（実際に起きました）。`stripLocalePrefix` がデフォルトロケールのプレフィックスまで剥がすのはこのためです。

:::message
細かい罠は他にもあります。typedRoutes を使うとファイルシステムに存在しない `/pricing` は型エラーになるので、キャストを `bareHref()` という 1 関数に隔離して「この嘘を成立させているのは proxy のリライトである」というコメントを付けています。また分岐 (3) が「ドキュメント GET」に限定されているのは、RSC ペイロードのフェッチをロケール跨ぎでリダイレクトするとルーターと URL バーが食い違うのと、HEAD で来る死活監視に cookie を立てないためです。
:::

## 辞書層: Zod スキーマ + satisfies + props

### スライス: schema + en + ja を 1 ファイルに

辞書はページ（サーフェス）ごとの「スライス」に分かれていて、各ファイルに Zod スキーマと英語と日本語が同居します。小さいスライスを丸ごと見るのが早いです。

```ts:src/lib/i18n/slices/playground.ts
import { z } from "zod";
import { metaCopySchema } from "./meta";

export const playgroundCopySchema = z.object({
  meta: metaCopySchema, // title / description の共有サブスキーマ
  heading: z.string().min(1),
  sub: z.string().min(1),
});

export type PlaygroundCopy = z.infer<typeof playgroundCopySchema>;

export const playgroundEn = {
  meta: {
    title: "Playground",
    description: "Chat with any OpenAI-compatible endpoint straight from the browser.",
  },
  heading: "Playground",
  sub: "Chat with any OpenAI-compatible endpoint straight from the browser.",
} satisfies PlaygroundCopy;

export const playgroundJa = {
  meta: {
    title: "Playground",
    description: "ブラウザから直接、OpenAI 互換エンドポイントとチャットできます。",
  },
  heading: "Playground",
  sub: "ブラウザから直接、OpenAI 互換エンドポイントとチャットできます。",
} satisfies PlaygroundCopy;
```

組み立て側は各スライスを寄せ集めるだけです。

```ts:src/lib/i18n/index.ts（抜粋）
export const dictionarySchema = z.object({
  chrome: chromeCopySchema,
  home: homeCopySchema,
  pricing: pricingCopySchema,
  playground: playgroundCopySchema,
  // ... 全 11 スライス
});

export type Dictionary = z.infer<typeof dictionarySchema>;

export const en = { chrome: chromeEn, /* ... */ } satisfies Dictionary;
export const ja = { chrome: chromeJa, /* ... */ } satisfies Dictionary;

export function getDictionary(locale: Locale): Dictionary {
  return locale === "ja" ? ja : en;
}
```

保証は二段構えです。**型**（`satisfies Dictionary`）がキーの欠落・余剰・型違いをコンパイル時に捕まえ、**Zod** が型では見えない制約を捕まえます。空文字列（`.min(1)`）や「カードは必ず 4 枚」（`.length(4)`）のような制約です。後者は CI のテストで両ロケールをまとめて parse するだけです。

```ts:src/lib/i18n/dictionaries.test.ts
describe("i18n dictionaries", () => {
  it.each([
    ["en", en],
    ["ja", ja],
  ] as const)("%s parses against the dictionary schema", (_locale, dict) => {
    expect(() => dictionarySchema.parse(dict)).not.toThrow();
  });
});
```

### Provider も t() もない

コンポーネントへの受け渡しは Server Component の props だけです。コンテキストプロバイダーも `t()` ランタイムもありません。

```tsx:src/app/[locale]/(marketing)/layout.tsx
export default async function MarketingLayout({ children, params }: LayoutProps<"/[locale]">) {
  const locale = await resolveLocale(params);
  const dict = getDictionary(locale);
  return (
    <div className="arkor-theme">
      <SiteHeader locale={locale} chrome={dict.chrome} />
      {children}
      <SiteFooter locale={locale} chrome={dict.chrome} />
    </div>
  );
}
```

ページも同じ形で、`getDictionary(locale).pricing` のようにスライスを取って JSX に流し込みます。リーフのコンポーネントは `copy: ProductPagesCopy["cta"]` のようにサブスライスの型を props に取るだけで、クライアントコンポーネントにも文字列が props で渡るため、辞書全体がクライアントバンドルに乗ることはありません。

### インライン強調は 3 マーカーだけ

見出しの一部だけ色を変えたい、というよくある要件のために ICU MessageFormat を再発明はせず、辞書の文字列に `<kw>`（強調スパン）と `<br/>` だけを許すミニマーカーにしました。レンダラーは 50 行程度で、区切りを残す capturing split でパースします。

```tsx:src/components/marketing/rich-copy.tsx（抜粋）
const parts = text.split(/(<kw>.*?<\/kw>|<br\s*\/>)/g);
// <kw>...</kw> は <span className="arkor-kw">、<br/> は <br />、
// それ以外の山かっこはそのまま素通しする
```

これで済むのは、翻訳者に渡す汎用フォーマットではなく自分たちしか書かない辞書だからです。しかも強調位置は言語ごとに自由です。英語では "Fine-tune and deploy your model, `<kw>`all in TypeScript`</kw>`."、日本語では「ファインチューニングもデプロイも、`<kw>`すべて TypeScript で`</kw>`。」のように、係り受けの違いに合わせてマーカーを別の句に置けます。コンポーネント差し込み型の interpolation ではかえって面倒になるところです。

もうひとつ運用ルールとして「コピーとデータの境界」を決めています。コマンド（`pnpm create ...`）、コード片、アナリティクスのイベント名、ブランド名は辞書に入れません。翻訳対象はあくまで人間向けのコピーだけです。

### 触りたくないデータはオーバーレイで訳す

例外がひとつあります。プロダクトカタログ（ナビゲーションに並ぶ製品一覧）は英語のデータファイルが source of truth で、プロダクト側の作業で頻繁に増えます。これを辞書に取り込むと翻訳必須になり、カタログ変更のたびに多言語対応がブロッカーになってしまう。そこでカタログだけは slug をキーにした部分オーバーレイにしました。

```ts:src/lib/i18n/slices/products.ts（抜粋）
export const catalogCopySchema = z.object({
  // record の値を optional にして「未知の slug は本当に miss する」型にする
  products: z.record(z.string(), productOverlaySchema.optional()),
});

export function localizeProductCopy<T extends Product>(product: T, catalog: CatalogCopy): T {
  const overlay = catalog.products[product.slug];
  return overlay ? { ...product, ...overlay } : product;
}
```

オーバーレイにない slug は英語のまま表示されます。サイレントな英語フォールバックを許しているのはコードベース全体でここだけで、「新しいカタログエントリーは翻訳されるまで英語で出る」を公認の状態として扱っています。

## コンテンツ層: MDX 記事は「ミラー」で訳す

ブログと changelog のような長文 MDX は辞書に入れる粒度ではないので、コンテンツディレクトリーの中に日本語版をミラーとして置きます。`content/blog/<slug>.mdx` が英語、`content/blog/ja/<slug>.mdx` が日本語で、URL の言語はサイト共通の `/[locale]` セグメントが決めます（`/ja/blog/<slug>`）。

解決ロジックは fumadocs に依存しない純関数です。

```ts:src/lib/content-locale.ts（抜粋）
export function resolveArticle<TPage>(getPage: GetPage<TPage>, locale: Locale, slug: string) {
  if (locale === "ja") {
    const mirror = getPage(["ja", slug]);
    if (mirror) return { page: mirror, slug, isFallback: false };
  }
  const english = getPage([slug]);
  if (!english) return undefined;
  return { page: english, slug, isFallback: locale === "ja" };
}
```

ポイントは `isFallback` です。日本語ミラーがなければ 404 にせず英語記事を出しますが、その事実を呼び出し側に伝えます。ページはこれを受けて記事本文に `lang="en"` を付け、JSON-LD の言語表記も揃えます。`/ja` 配下でスクリーンリーダーが英語本文を日本語規則で読み上げたり、構造化データと実際の言語が食い違ったりしないようにするためです。

ただし、このフォールバックは安全網であってワークフローではありません。CI にミラーを 1:1 で固定するテストがあり、次の 3 点を検査します。

1. 英語記事すべてに日本語ミラーが存在する（訳し忘れは CI が落ちる）
2. 英語の原文がない日本語記事が存在しない（到達不能な記事の検出）
3. 日付の frontmatter がペアで一致する（sitemap の lastModified と JSON-LD の datePublished は配信されたロケール側から読むので、ズレると同じ記事が 2 つの公開日で出てしまう）

つまりフォールバック方針は層ごとに意図的に逆です。カタログは「サイレントに英語で出てよい」（変更頻度が高く、翻訳をブロッカーにしない）、長文コンテンツは「実行時は英語で凌ぐが、その状態の存在自体を CI が禁止する」。どちらも「フォールバックするか」ではなく「フォールバックをいつ誰が検知するか」の設計です。

## まとめ

ライブラリーなしの多言語対応は、条件が揃えば十分に実用的でした。今回の条件は次の 3 つです。

- ロケールが 2 つで、翻訳サーフェスがマーケティングページに限定されている
- 翻訳をエンジニア自身が書くので、JSON 納品や翻訳管理サービスとの連携が要らない
- 型システム（`satisfies`）と CI テスト（Zod parse、ミラー 1:1、ファイルシステム diff）で運用ルールを機械的に固定できる

得られたものは、追加ランタイム依存ゼロ、キー欠落のコンパイル時検出、単一ファイル diff で完結するコピー変更、そして「未定義キーが画面に漏れる」クラスのバグの構造的な排除です。

逆に、ロケールが多数ある、翻訳を外部に出すので交換フォーマットが必要、ICU の複数形・性数一致が要る、といった状況ではライブラリーの土俵です。手書きが勝つのは要件が非対称で小さいときで、そのときのコード量は今回の実測でネゴシエーション 29 行、マーカーレンダラー 55 行、辞書アセンブリー 75 行でした。この規模なら、ライブラリーの設定ファイルより自分のコードのほうが読みやすい、というのが実装してみての結論です。
---

Arkor の取り組みに興味を持っていただけたら、[GitHub リポジトリー](https://github.com/arkorlab/arkor)にスターをいただけるとうれしいです。
