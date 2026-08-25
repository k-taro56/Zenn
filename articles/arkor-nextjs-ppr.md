---
title: "Next.js 16.3 の PPR で、動的ページを待たせたない。Arkor の実践"
emoji: "⚡"
type: "tech"
topics: ["nextjs", "react", "typescript", "playwright", "arkor"]
published: true
publication_name: "romanark"
---

オープンモデルのファインチューニングとデプロイを TypeScript で扱う [Arkor](https://arkor.ai/) の Web アプリには、性質の異なる画面が同居しています。

- 誰にでも同じ内容を返せるマーケティングサイトと Docs
- `cookies()` からログイン状態を読むヘッダー
- 組織やプロジェクトごとに内容が変わるダッシュボード
- 外部 API から最新情報を取得するステータスページ

静的なページだけなら事前生成し、動的なページだけならリクエストごとにレンダリングすれば済みます。しかし、実際の画面はその中間です。ヘッダーや見出しは全員に共通でも、右上の認証ボタンや一覧データだけはリクエストが来るまで確定しません。（Arkor ではすべてのページで、何かしらのコンポーネントが、リクエスト時にレンダリングする必要があります）

そこで Arkor では Next.js 16 の Cache Components を有効にし、そのレンダリングモデルである Partial Prerendering（PPR）を使っています。執筆時の構成は Next.js 16.3.1 です。認証や API の応答を待つ前に、マーケティング画面ではロゴと固定ナビゲーションを、認証付きダッシュボードでは見出しとページ枠（シェル）をレンダリングできるようにしました。

もう一つ、本番運用で無視できなかったのが Vercel との組み合わせです。Arkor では Proxy rewrite 後の 404 とページ内 `redirect()` で、ローカルの `next start` と Vercel 本番の結果が一致しませんでした。静的シェルの改善と、デプロイ先での HTTP ステータス検証は分けて考える必要があります。

この記事では、App Router で Cache Components を使う人に向けて、フラグを有効にするだけでは終わらなかった実装と、運用して分かった実装・配信上の注意点を紹介します。掲載するコードは要点が分かるように簡略化しています。

## まず押さえること

### PPR は「ページを静的か動的かに分類する機能」ではない

Cache Components を有効にすると、Next.js はビルド時にコンポーネントツリーを可能なところまでレンダリングします。リクエストに依存しない部分は静的シェルに入ります。`cookies()`、`headers()`、`searchParams`、未確定の `params`、キャッシュしていないデータ取得などに依存する部分は、明示的に `<Suspense>` で囲むことでリクエスト時まで描画を遅らせられます。必要な境界がなければ、開発時の Instant Insights が blocking route として指摘し、構成によってはエラーになります。

```text
ページ
├─ 事前生成: ロゴ、固定ナビゲーション、見出し
├─ リクエスト時に描画: ログイン状態      → 後からストリーム
└─ リクエスト時に描画: プロジェクト一覧  → 後からストリーム
```

つまり、境界はルート単位ではなく、コンポーネントツリー内の `<Suspense>` とキャッシュスコープ単位です。同じレスポンスの中で、事前生成した HTML とリクエスト時の出力を組み合わせられます。

Next.js 16 では `cacheComponents` を有効にすると PPR が標準のレンダリングモデルになります。以前の `experimental.ppr` やルート単位の `experimental_ppr` は使いません。（おそらく、Next.js 17 でいずれも廃止されます）

Arkor の設定は次の形です。

```ts:next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
};

export default nextConfig;
```

名前が似ていますが、ここには別の機能が 2 つあります。

- **Partial Prerendering（PPR）** は、事前生成した静的シェルと、リクエスト時にレンダリングする部分を組み合わせるレンダリングモデルです。
- **Partial Prefetching** は Next.js 16.3 で導入された、`<Link>` が遷移先のどこまでを先読みするかというナビゲーションの仕組みです。

まず PPR で「すぐレンダリングできる UI」を作り、Partial Prefetching でそのシェルをクリック前に取得する、という関係です。

ここでいう「すぐ」は、動的データをゼロ時間で取得するという意味ではありません。初回表示なら静的シェルの HTML、クライアント遷移なら事前取得済みの App Shell をすぐレンダリングし、動的な部分を後からストリームする、という意味です。即時表示できるのはキャッシュ済みの場合であり、キャッシュミス時の処理まで消えるわけではありません。

### 最初に決めたのは、何をキャッシュするかではなく何を待たせないか

Arkor では、データを次のように分類しました。

| データ | 扱い | 理由 |
| --- | --- | --- |
| 固定のナビゲーション、説明文、見出し | 静的シェル | リクエスト情報を必要としない |
| セッション、ユーザー別一覧 | `<Suspense>` の内側 | 常に現在のユーザーに対して取得したい |
| 公開の稼働状況 | `"use cache: remote"` | 全ユーザーで共有でき、外部 API を保護したい |
| 既知の Docs・ブログ記事 | `generateStaticParams` で事前生成 | URL ごとの本文を静的に配信できる |

特に重要だったのは、認証付き画面を速くするためにユーザーデータまで共有キャッシュへ入れないことです。ダッシュボードのデータは新鮮なままにし、その手前にある見出しやページ枠だけを静的シェルへ移しました。

## 静的シェルを増やした実装

### ヘッダーでは、認証部分だけをリクエスト時にレンダリングする

導入前のヘッダーは Server Component 全体が `async` で、先頭でセッションを取得していました。この構造では、右上の「Log in」か「Dashboard」かを決めるために、ロゴやナビゲーションまで待たされます。

現在はヘッダー本体を同期コンポーネントにし、認証に依存する部分だけを分離しています。

```tsx:site-header.tsx
import { Suspense } from "react";

export function SiteHeader() {
  return (
    <header>
      <Logo />
      <StaticNavigation />

      <Suspense fallback={<AuthNavFallback />}>
        <AuthNavActions />
      </Suspense>
    </header>
  );
}

async function AuthNavActions() {
  const session = await getSession();

  return session ? <DashboardAndLogout /> : <LoginAndSignup />;
}
```

これでロゴとナビゲーションは静的シェルに入り、認証ボタンだけが後からストリームされます。

`AuthNavFallback` は単なるスピナーではありません。ログアウト時のボタンと同じ幅の、見えないプレースホルダーを置いています。内容が解決した瞬間にヘッダー全体が横へずれないようにするためです。

次の画像は、後述する `instant()` で認証処理を止めたマーケティング画面のヘッダーです。ロゴと固定ナビゲーションはすでにレンダリングされ、右端では認証ボタンと同じ幅だけが確保されています。

![instant() で認証処理を停止した Arkor のヘッダー。ロゴと固定ナビゲーションが表示され、認証ボタン用の幅が確保されている](/images/arkor-nextjs-ppr/site-header-static-shell.png =900x)
*青は事前生成済みの部分、橙は認証結果を待つ間も幅を維持する部分。*

デスクトップ用とモバイル用のセッション取得は React の `cache()` で、同一リクエスト内の重複だけを避けています。これは、後で登場する Next.js の `"use cache"` によるリクエスト間キャッシュとは別物です。

### `await` を、それを必要とする子コンポーネントへ移す

ダッシュボードのジョブ一覧も、導入前はページの先頭で認証、`params`、API を順に待っていました。

```tsx
// ページ全体が動的処理を待つ
type JobsPageProps = PageProps<"/[orgSlug]/[projSlug]/jobs">;

export default async function JobsPage({ params }: JobsPageProps) {
  await requireSession();
  const { orgSlug, projSlug } = await params;
  const jobs = await listJobs(orgSlug, projSlug);

  return <JobsScreen jobs={jobs} />;
}
```

この形を、同期のページ枠と非同期の子コンポーネントへ分けました。

```tsx:app/[orgSlug]/[projSlug]/jobs/page.tsx
import Link from "next/link";
import { Suspense } from "react";

type JobsPageProps = PageProps<"/[orgSlug]/[projSlug]/jobs">;
type Params = JobsPageProps["params"];

export default function JobsPage({ params }: JobsPageProps) {
  return (
    <div>
      <div className="page-heading">
        <h1>Training Jobs</h1>
        <Suspense fallback={<NewJobPlaceholder />}>
          <NewJobLink params={params} />
        </Suspense>
      </div>

      <Suspense fallback={<CardListSkeleton />}>
        <JobsList params={params} />
      </Suspense>
    </div>
  );
}

async function NewJobLink({ params }: { params: Params }) {
  const { orgSlug, projSlug } = await params;
  return <Link href={`/${orgSlug}/${projSlug}/jobs/new`}>New Job</Link>;
}

async function JobsList({ params }: { params: Params }) {
  await requireSession();
  const { orgSlug, projSlug } = await params;
  const jobs = await listJobs(orgSlug, projSlug);
  return <JobCards jobs={jobs} />;
}
```

見出しは URL にもユーザーにも依存しないため、すぐレンダリングできます。一方、リンクの `href` は `params`、一覧は認証と API に依存するため、それぞれ必要最小限の境界に残します。

Next.js 15 以降の `params` は Promise です。さらに、テナントのように値を列挙できず `generateStaticParams` を持たない動的ルートでは、ビルド時の App Shell に特定 URL の値を入れられません。ページの先頭で `await params` するのではなく、Promise のまま子へ渡すのがポイントです。

この形は Jobs だけでなく、Endpoints、Usage、設定画面にも適用しました。ただし、静的シェルへ出す範囲は画面ごとに変えています。たとえば Usage の期間ラベルはデータ取得に使う時間窓と一致する必要があるため、見出しだけを静的にして、期間ラベルはグラフと一緒にストリームさせています。

### 静的シェルを妨げていたのは親レイアウトだった

各ページを分割しても、最初はダッシュボードの静的シェルがほとんど増えませんでした。原因は上位の組織レイアウトです。

次のように、セッションを読む非同期ガードが `children` を包んでいました。

```tsx
// 子ページまでセッション取得の完了を待つ
<Suspense fallback={<OrgNavSkeleton />}>
  <OrgGuard params={params}>{children}</OrgGuard>
</Suspense>
```

`OrgGuard` が結果を返すまで、その内側にある子ページも出てきません。子ページ側で見出しを同期化しても、親レイアウトの処理完了を待っていたのです。

修正は、認証に依存するナビゲーションと子ページを兄弟にすることでした。ただし、この変更が成立するのは、`children` 配下のメンバー専用ページと API が、それぞれセッションと組織メンバーシップを検証している場合だけです。

:::message alert
PPR はアクセス制御ではありません。静的シェルに保護データを置かず、認可はページや Route Handler、データアクセス層で独立して行います。レイアウトの表示分岐だけを認可として扱ってはいけません。
:::

表示用コンポーネントであることが分かるよう、ここでは `OrgGuard` を `AuthenticatedOrgChrome` と呼び換えます。

```tsx
export default function OrgLayout({ children, params }: LayoutProps) {
  return (
    <div>
      <Suspense fallback={<OrgNavSkeleton />}>
        <AuthenticatedOrgChrome params={params} />
      </Suspense>

      {children}
    </div>
  );
}
```

これでナビゲーションはセッションを待ちながら、子ページは自分自身の `<Suspense>` 境界まで先にレンダリングできます。

同じ URL が匿名向けの公開画面とメンバー向け画面を持つ場合は、フォールバックにも注意が必要です。Arkor の公開プロジェクト画面では、セッションが判明するまで外側の認証境界を `fallback={null}` にし、ログイン済みと分かってからメンバー向けナビゲーションのスケルトンを出します。匿名訪問者へ一瞬だけ会員用 UI を見せないためです。

## 運用で分かった注意点

### `use cache` は共有してよいデータにだけ使う

PPR を使うと、何でも `"use cache"` で静的シェルへ入れたくなります。しかし、Arkor のユーザー別ダッシュボードデータはキャッシュせず、毎回認証して取得しています。

一方、公開ステータスページの稼働情報は全ユーザーで共有できます。取得先の外部 API をリクエスト数から守る価値もあるため、ここではリモートキャッシュを使っています。

```tsx:app/status/page.tsx
import { Suspense } from "react";
import { cacheLife } from "next/cache";

async function getInferenceStatus() {
  "use cache: remote";
  cacheLife({ stale: 60, revalidate: 60, expire: 240 });

  return fetchUptimeStatus();
}

async function StatusPanel() {
  const status = await getInferenceStatus();
  return <StatusCard status={status} />;
}

export default function StatusPage() {
  return (
    <main>
      <StatusHeading />
      <Suspense fallback={<StatusPanelFallback />}>
        <StatusPanel />
      </Suspense>
    </main>
  );
}
```

この設定では `expire: 240` が prerender 対象の最小値である 5 分を下回るため、ステータスは事前生成せず、リクエスト時にレンダリングする部分に残ります。それでも Arkor の Vercel 環境では remote cache handler がサーバーレスインスタンスをまたいで値を共有します。キャッシュヒットが続く通常時なら、複数の外部 API を呼ぶ更新処理をアクセスごとには繰り返さず、おおむね 1 分単位に集約できます。キャッシュミス、デプロイ、取得失敗後の再試行ではこの限りではありません。

`"use cache"` と `"use cache: remote"` の中から `cookies()` や `headers()` を直接読むことはできません。必要なら外で値を読み、引数として渡す設計になります。その値はキャッシュキーの一部になるため、「共有して本当によい単位か」を先に考える必要があります。

### Partial Prefetching では「1 ルートに 1 つの App Shell」を意識する

`partialPrefetching: true` を有効にすると、通常の `<Link>` は URL ごとの完全な出力ではなく、ルートごとに再利用できる App Shell を先読みします。

これはリンクが多い画面で効きます。たとえば `/blog/a` と `/blog/b` は同じ `/blog/[slug]` ルートなので、既定では共有の App Shell を 1 つ取得すれば済みます。

ただし、共有シェルは `slug` に依存できません。Arkor の記事ページでは `params` の読み取りを `<Suspense>` の内側へ置いているため、先読み済みデータがない状態でのクライアント遷移では、最初に記事スケルトンが見えます。

既知の記事は `generateStaticParams` で完全に事前生成しています。さらに、クリック時点で本文まで用意したいリンクには `prefetch={true}` を指定し、per-link prefetching で URL 固有の RSC を先読みします。Arkor が採用した 16.3.1 のドキュメントでは、この仕組みを Runtime Prefetching と呼んでいました。

```tsx:article-link.tsx
import Link from "next/link";
import type { ComponentProps } from "react";

export function ArticleLink(props: ComponentProps<typeof Link>) {
  return <Link {...props} prefetch={true} />;
}
```

ここではインターネット帯域とのトレードオフがあります。Next.js 16.3.1 の本番ビルドで計測すると、Arkor の Docs のサイドバーと本文内リンクは 1 ページあたり約 11 宛先、各 RSC payload は非圧縮でおよそ 45〜100 KB でした。CDN から返る静的データとはいえ、すべてのリンクで本文を先読みすれば転送量は増えます。

そこで使い分けています。

- Docs のサイドバーや CTA のように、情報設計上リンク数が上限を持つ場所は常時 `prefetch={true}`
- 記事数とともに増え続けるブログ・変更履歴の一覧は、ホバーまたはフォーカスされたリンクだけ `prefetch={true}` へ切り替える
- タッチ操作では共有 App Shell とスケルトンを使い、一覧全件の本文を先読みしない

増え続ける一覧では、次のように利用意図を検出して一方向に先読みを有効化しています。

```tsx:article-intent-link.tsx
"use client";

import Link from "next/link";
import { useState, type ComponentProps } from "react";

export function ArticleIntentLink({
  onMouseEnter,
  onFocus,
  ...props
}: ComponentProps<typeof Link>) {
  const [intent, setIntent] = useState(false);

  return (
    <Link
      {...props}
      prefetch={intent ? true : undefined}
      onMouseEnter={(event) => {
        setIntent(true);
        onMouseEnter?.(event);
      }}
      onFocus={(event) => {
        setIntent(true);
        onFocus?.(event);
      }}
    />
  );
}
```

「先読みできるか」ではなく「そのリンク数でも先読みすべきか」で決めるのが大切でした。

### PPR の HTTP ステータスは Vercel 本番でも確認する

PPR 導入後、特に慎重な確認が必要だったのが HTTP ステータスです。Arkor の Next.js 16.3.1 / 2026 年 8 月時点の Vercel 本番調査では、Proxy の rewrite 先によって `next start` と Vercel の結果が一致しませんでした。

| Proxy の rewrite 先 | `next start` | Vercel 本番 | 初期 HTML |
| --- | --- | --- | --- |
| `/en/_404`（ページ内で `notFound()`） | 404 | 404（既存の本番運用で確認） | `html#__next_error__` となり、404 カードは含まれない |
| `/_not-found` | 404 | 200（Proxy rewrite で到達した場合） | 404 カードが SSR される |

JavaScript が有効なブラウザでは、`/en/_404` も RSC ペイロードの適用後に 404 カードを表示します。画面だけを確認すると、両者の違いに気づけません。

`/_not-found` への rewrite を Vercel で観測したときのレスポンスは次のとおりでした。

```text
HTTP/2 200
x-matched-path: /_not-found
x-vercel-cache: MISS
x-robots-tag: noindex, nofollow
```

`x-vercel-cache: MISS` なので、CDN に残った古い 200 レスポンスではありません。ただし、なぜ Vercel でステータスが変わるのかは分かっていません。`/_not-found` は `prerender-manifest.json` と `.meta` の両方が 404 を示す一方、`/en/_404` は manifest が 200、`.meta` が 404 を示していました。今回のケースでは、この 2 つの生成物だけから Vercel の実際の応答を予測できなかったため、内部の仕組みを推測せず、本番の観測結果を判断材料にしています。

Arkor は、404 カードを初期 HTML に含めることより、正しい HTTP 404 を返すことを優先し、rewrite 先を `/en/_404` にしました。代わりに、JavaScript を実行しないクローラーや利用者が受け取る初期 HTML には 404 カードが入りません。

なお、上の 200 は **Proxy の rewrite で `/_not-found` へ到達したケース**です。Next.js が通常のルート照合で処理する未知 URL まで、Vercel が常に 200 を返すという意味ではありません。

#### `notFound()` より前に未知の URL を判定する

Arkor の構成では、`generateStaticParams` に含まれない Docs・ブログ・変更履歴の URL で、ページ内の `notFound()` より先にフォールバック用の静的シェルが送信され、HTTP 200 が確定するケースがありました。

そこで、公開済み URL の軽量な slug manifest をビルドへ含めています。Proxy はリクエストされた slug を manifest と照合し、未知ならページを描画する前に `/en/_404` へ rewrite します。遅れて実行される `notFound()` に HTTP ステータスを委ねないためです。Proxy の単体テストでは判定条件や rewrite 先を確認できますが、Proxy が返す rewrite レスポンス自体のステータスだけでは、最終的に配信されるステータスを証明できません。

#### `redirect()` もデプロイ先で結果が変わった

`generateStaticParams` の対象外になる `/docs` でページ内の `redirect()` を実行すると、ローカルの `next start` では HTTP 200 とクライアント向けリダイレクト payload が返りました。同じ構成を Vercel へデプロイすると、応答は 500 になりました。

Arkor では、この恒久リダイレクトを `next.config.ts` へ移し、HTTP 308 としてシェルより前に確定させています。ページ内の `redirect()` は防御的な確認として残しました。オンボーディングのようにユーザーごとの判定が必要なものは Proxy で処理し、ページ側でも認可を再確認しています。

#### 検証を 3 層に分ける

ここから得た教訓は、ローカルの本番ビルドと Vercel 本番を同じ検証として扱わないことです。

1. **本番ビルドと `instant()`**：動的処理を止めても、意味のある静的シェルがレンダリングされるかを確認する
2. **ローカルのリクエストテスト / E2E**：`next start` に対する最終ステータス、初期 HTML、hydration 後の画面を確認する
3. **デプロイ後の HTTP smoke test**：Vercel 上で 200、404、rewrite、redirect を直接確認する

:::message alert
`instant()` が成功しても、Vercel の HTTP ステータスや rewrite、redirect まで保証したことにはなりません。`next start` の結果も Vercel 本番の代用にはなりません。
:::

Arkor ではビルドコストの都合で Preview Deployment を常時有効にしていないため、Vercel 固有の差をマージ前に再現できません。`/onboarding`、存在しない Docs URL、既知の Docs URL、sitemap 内の URL などを対象にしたデプロイ後 smoke test は、今後自動化したい確認として残っています。

### `catch` で prerender の中断を飲み込まない

Cache Components の事前生成中には、`redirect()`、`notFound()`、`cookies()` や `headers()` などの Request-time API、特定の uncached fetch が、Next.js 内部の制御フローとして例外経路を通ることがあります。これを通常の障害と同じように `catch` すると、たとえば「セッション取得に失敗したのでログアウト表示にする」という分岐を静的シェルへ誤って事前生成する可能性があります。

Arkor では、動的 API を含む処理を捕捉するとき、最初に `unstable_rethrow()` へ渡しています。

```tsx
import { unstable_rethrow } from "next/navigation";

try {
  return await getSession();
} catch (error) {
  unstable_rethrow(error);

  // ここから下はアプリケーション側で扱うエラー
  console.warn("session lookup failed", error);
  return null;
}
```

`redirect()` や prerender interruption のような Next.js の制御フローは再送出し、残ったエラーをアプリケーション側で処理します。残るのは外部障害だけとは限らないため、実際のコードでは必要に応じて型やエラーコードでも対象を絞ります。

:::message alert
`unstable_rethrow()` は名前のとおり不安定な API で、公式ドキュメントでは変更の可能性があり、production での利用は推奨されていません。Arkor では Next.js と関連パッケージを 16.3.1 に固定し、必要な箇所へ限定して使っています。新しいコードでは、まず `try/catch` の範囲を狭め、採用する場合は Next.js の更新時に挙動を監査してください。
:::

PPR を有効にした後は、「広い `try/catch` が動的処理を囲んでいないか」も監査対象になりました。

### フォールバックは、表示するだけでなく形を守る

静的シェルが速くても、解決後に大きくレイアウトが動けば体験は良くなりません。一方、可変長のコンテンツをピクセル単位で再現する必要もありません。Arkor では次をレビューの観点にしています。

- 高さや構造が決まる UI は、最終 UI と同じ外枠とブレークポイントを使う
- 可変長の一覧や記事は、代表的な非空状態のスケルトンを使う
- ページ見出しなど、確定している本物のコンテンツはスケルトンへ置き換えない
- 読み上げ用の `role="status"` と `aria-live="polite"` を境界側に置く
- 装飾用のスケルトン要素は `aria-hidden="true"` にする
- アニメーションを付ける場合は、可能なら `motion-safe:` で利用者の設定を尊重する
- `loading.tsx` とページ内 `<Suspense>` で同じスケルトンコンポーネントを共有する

高さが決定的な 2 つのフォーム画面では、E2E テストでスケルトンと解決後のフォームの高さも比較しています。フィールドが追加されたのにスケルトンが古いまま、というドリフトをレビュー時の目視だけに任せないためです。

## `instant()` で「本当にすぐ見えるもの」を固定する

通常の E2E テストで最終状態だけを確認しても、静的シェルが空に戻ったことは検出できません。データが数百ミリ秒後に表示されればテストは通るからです。

Next.js 16.3 の `@next/playwright` には `instant()` があります。コールバック内ではリクエスト時に描画する部分を止めた状態で、静的シェルだけを検証できます。

```bash
pnpm add -D @next/playwright@16.3.1 @playwright/test
```

`@next/playwright` は Next.js のリリース系列に追従するため、Arkor では `next` と同じ 16.3.1 に固定しています。

### `next-cache-components-optimizer` スキルで検証を進める

今回の改善と検証には、Next.js 公式リポジトリで公開されている [`next-cache-components-optimizer`](https://github.com/vercel/next.js/tree/canary/skills/next-cache-components-optimizer) スキルを使いました。これはアプリへ組み込むライブラリではなく、Cache Components / PPR の改善をテスト駆動で進める作業手順です。[Next.js の AI coding agents ガイド](https://github.com/vercel/next.js/blob/canary/docs/01-app/02-guides/ai-agents.mdx)から、次のように追加できます。

```bash
npx skills add vercel/next.js --skill next-cache-components-optimizer
```

対象ルートごとに、次の流れを繰り返します。

1. ロックを使わない通常の遷移で、検証対象の UI がテストユーザーにも表示されることを確認する
2. `instant()` の内側では静的シェルが表示されず、失敗するテストを作る
3. 親レイアウトやページ先頭の `await`、範囲が広すぎる `<Suspense>` を特定する
4. 非同期処理を、それを必要とする最小のコンポーネントへ移し、既存のスケルトンをフォールバックとして再利用する
5. 修正だけを戻すと失敗し、再適用すると成功することを確かめ、テストを退行防止として残す

ここで `instant()` は所要時間を測るストップウォッチではありません。リクエスト時の処理を止めても、意味のある静的シェルが表示されるかを判定するための定規です。空の `fallback={null}` だけが描画される状態は、テストが成功しても有用なシェルとは見なしません。

次の 2 枚は、同じ Jobs 画面を同じビューポートで撮影したものです。注釈も Playwright が実要素の位置から生成しているため、画面変更後に同じ手順で作り直せます。

![instant() でリクエスト時のレンダリングを停止した Jobs 画面。見出しと New Job の代替表示、一覧スケルトンが表示されている](/images/arkor-nextjs-ppr/jobs-static-shell.png =900x)
*`instant()` の内側。青は事前生成済み、橙はリクエスト時の処理を待つ部分。*

![停止解除後の同じ Jobs 画面。見出しを維持したまま New Job リンクと 3 件のジョブが表示されている](/images/arkor-nextjs-ppr/jobs-streamed-content.png =900x)
*通常のレンダリング。見出しはそのまま、緑の部分がリクエスト時に届く。*

### 初回表示（hard navigation）とクライアント遷移（soft navigation）を別々に検証する

Arkor のテストは、ロック中に静的な見出しが見え、ジョブデータはまだ表示されないことを同時に確認します。後者があることで、テスト用 API の有効化を忘れて `instant()` が何もしなかった場合も検出できます。

次の例では、`authenticated` fixture がログイン Cookie とテスト用の組織・プロジェクト、`triage-v3` というジョブを用意済みとします。初回表示では `page.goto()` 自体を `instant()` の内側に置き、まだ遷移しておらず `about:blank` の `page` にもロック用 Cookie を設定できるよう `baseURL` を渡します。

```ts:e2e/instant-navigation.spec.ts
import { instant } from "@next/playwright";
import { test, expect } from "./fixtures/authenticated";

const ORG = "acme-team";
const PROJECT = "training";
const JOB = "triage-v3";

test("Jobs の初回表示で静的シェルがレンダリングされる", async ({
  page,
  baseURL,
}) => {
  if (!baseURL) throw new Error("Playwright の baseURL が必要です");

  await instant(
    page,
    async () => {
      await page.goto(`/${ORG}/${PROJECT}/jobs`);

      await expect(
        page.getByRole("heading", { name: "Training Jobs" }),
      ).toBeVisible();
      await expect(page.getByText(JOB)).toHaveCount(0);
    },
    { baseURL },
  );

  // 最初のドキュメントはロック中に返っているため、通常の応答を取り直す
  await page.reload();
  await expect(page.getByText(JOB)).toBeVisible();
});

test("Jobs へのクライアント遷移で App Shell がレンダリングされる", async ({
  page,
}) => {
  await page.goto(`/${ORG}/${PROJECT}`);
  const jobsLink = page.getByRole("link", { name: "Jobs", exact: true });
  await expect(jobsLink).toBeVisible();

  await instant(page, async () => {
    await jobsLink.click();
    await page.waitForURL(`**/${ORG}/${PROJECT}/jobs`);

    await expect(
      page.getByRole("heading", { name: "Training Jobs" }),
    ).toBeVisible();
    await expect(page.getByText(JOB)).toHaveCount(0);
  });

  await expect(page.getByText(JOB)).toBeVisible();
});
```

初回表示とクライアント遷移は、同じ URL でも静的シェルが異なることがあります。初回表示はルートからすべてのレイアウトを通りますが、クライアント遷移は共有レイアウトより下だけを再レンダリングするためです。クライアント遷移は `page.goto()` で代用せず、実際の `<Link>` をクリックします。遷移先 URL を待ってから見出しを検証するのは、遷移元にある同名要素を拾う誤判定を防ぐためです。

判定には `next dev` を使いません。開発時は通常の自動 prefetch が無効で、ロックや配信条件も本番ビルドと異なるためです。`next build` と `next start` で検証するときは `experimental.exposeTestingApiInProductionBuild` が必要なので、テスト専用の環境変数がある場合だけ有効にし、本番デプロイのビルドへ混ざらないようにしています。

```ts:next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
  experimental: {
    ...(process.env.NEXT_E2E_TESTING_API === "1"
      ? { exposeTestingApiInProductionBuild: true }
      : {}),
  },
};

export default nextConfig;
```

その環境変数はビルド時と `next start` の起動時の両方へ渡します。Arkor のテスト基盤では `.next/required-server-files.json` も読み、測定対象のビルドに API が含まれることをテスト開始前に検査しています。さらに、上のテストはロック中に `JOB` が表示されないことも確認するため、ロックが効かずに通常レンダリングを見た場合は失敗します。

この本番ビルドによる `instant()` の成功が証明するのは、静的シェルの構成です。前節で見たとおり、Vercel の HTTP ステータスや rewrite、redirect の挙動は別途デプロイ後に確認します。

## 導入して分かったこと

PPR 導入で一番変わったのは設定ではなく、コンポーネントツリーの見方でした。

1. **ページ先頭と親レイアウトの `await` を疑う。** 認証、`params`、データ取得を、それを本当に必要とする最小の子コンポーネントへ移します。上位の非同期コンポーネントが `children` を包んでいないかも確認します。
2. **新鮮さが必要なデータは無理にキャッシュしない。** `<Suspense>` でストリームし、共通 UI だけを先に見せれば十分なことがあります。
3. **フォールバックを UI として設計し、`instant()` で固定する。** 初回表示とクライアント遷移の両方で、「あるべきもの」と「まだないべきもの」を検証します。
4. **先読みには予算を持つ。** 共有 App Shell、常時の per-link prefetching、利用意図に応じた prefetching をリンク数に応じて使い分けます。
5. **HTML 配信前に確定すべき制御を監査する。** データ認可はページと API に残しつつ、リダイレクトや HTTP ステータスは Proxy や設定層で確定させます。

PPR は、動的なアプリを静的サイトへ変える魔法ではありません。ユーザーが待たなくてよい部分をコンポーネントツリーから見つけ、残りだけを正しい境界で待たせる仕組みです。

Arkor では、マーケティングページの認証ボタン、テナント別ダッシュボード、記事ページ、外部 API を使うステータス画面まで、同じ考え方を適用できました。その状態を `instant()` で UI の契約として残したことで、今後のリファクタリングでも「いつの間にかページ全体が待つ」退行を検出できます。

## 参考資料

- [Next.js: Cache Components](https://nextjs.org/docs/app/getting-started/caching)
- [Next.js: `cacheComponents`](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents)
- [Next.js: Ensuring instant navigations](https://nextjs.org/docs/app/guides/instant-navigation)
- [Next.js: `next-cache-components-optimizer`](https://github.com/vercel/next.js/tree/canary/skills/next-cache-components-optimizer)
- [Next.js: AI Coding Agents](https://github.com/vercel/next.js/blob/canary/docs/01-app/02-guides/ai-agents.mdx)
- [Next.js: Adopting Partial Prefetching](https://nextjs.org/docs/app/guides/adopting-partial-prefetching)
- [Next.js: Optimizing prefetching](https://nextjs.org/docs/app/guides/optimizing-prefetching)
- [Next.js 16.3.1: Runtime Prefetching（当時の名称）](https://nextjs.org/docs/app/guides/runtime-prefetching)
- [Next.js: `use cache`](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [Next.js: `use cache: remote`](https://nextjs.org/docs/app/api-reference/directives/use-cache-remote)
- [Next.js: `cacheLife`](https://nextjs.org/docs/app/api-reference/functions/cacheLife)
- [Next.js: `notFound`](https://nextjs.org/docs/app/api-reference/functions/not-found)
- [Next.js: `unstable_rethrow`](https://nextjs.org/docs/app/api-reference/functions/unstable_rethrow)
- [Next.js 16.3](https://nextjs.org/blog/next-16-3)
- [Next.js 16.3: Instant Navigations（背景資料）](https://nextjs.org/blog/next-16-3-instant-navigations)

---

Arkor の取り組みに興味を持っていただけたら、[GitHub リポジトリー](https://github.com/arkorlab/arkor)にスターをいただけるとうれしいです。
