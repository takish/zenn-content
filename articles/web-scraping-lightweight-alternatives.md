---
title: "Web検索・スクレイピングAPIの選び方 — 有料APIの前に検討したい軽量な選択肢"
emoji: "🕷️"
type: "tech"
topics: ["スクレイピング", "API", "JinaReader", "Firecrawl", "GoogleAppsScript"]
published: false
---

## TL;DR

Webサービスやエージェントの開発で「URLの中身を取得したい」「Web検索結果を取り込みたい」という場面があります。Firecrawl や Brave Search のような有料 API を最初に選びがちですが、ユースケースによっては無料で十分なことも多いです。実際に Google Apps Script（GAS）プロジェクトで Brave Search + Firecrawl → Jina Reader に置き換えた経験をもとに、選び方を整理しました。

## よくある場面

Webサービスやボット、AIエージェントを作っていると、外部のWeb情報を取得したい場面が出てきます。

- **検索**: クエリを投げて、関連するページの一覧を取得する
- **スクレイピング**: URL を指定して、そのページの内容をテキストやMarkdownで取得する

どちらもブラウザでやれば一瞬ですが、プログラムから実行するには API が必要です。そして API には大抵、料金がかかります。

「とりあえず有名なサービスの API キーを取得して...」と進めた結果、管理する秘密情報が増え、無料枠を超えて課金が始まり、気づけばフォールバック用の別 API も契約している——という状態に陥ったことがありました。

## 選択肢の全体像

Web情報の取得に使える主なサービス・ツールを整理しました。

### 検索 API

| サービス | 料金 | 特徴 |
|---------|------|------|
| **Google Custom Search** | 100回/日 無料、以降 $5/1,000回 | Google の検索結果を取得。**新規受付終了**（既存顧客は2027年1月まで） |
| **Brave Search API** | 月$5クレジット（～1,000回）、以降従量課金 | プライバシー重視の検索エンジン。Web/News/Imagesに対応 |
| **Jina Search** (`s.jina.ai`) | 1,000万トークン無料（以降従量課金） | APIキー必須（無料発行可）。レスポンスに本文 Markdown を含む |
| **SerpAPI** | 100回/月 無料、以降 $75/月〜 | Google/Bing 等の検索結果をJSON化。構造化データが豊富 |

### スクレイピング API / ツール

| サービス | 料金 | JS対応 | 本文抽出 | 特徴 |
|---------|------|--------|---------|------|
| **Jina Reader** (`r.jina.ai`) | トークン制（初回1,000万トークン無料） | ○ | ○ | APIキーなしでも利用可（20 RPM）。GET 一発で Markdown が返る |
| **Firecrawl** | 500クレジット無料、以降 $16/月〜 | ○ | ○ | クロール機能あり。MCP対応 |
| **trafilatura** | 無料（OSS） | × | ◎ | Python CLI/ライブラリ。本文抽出精度が高い |
| **ScrapingBee** | 1,000回 無料、以降有料 | ○ | × | プロキシ・ブラウザレンダリング特化 |
| **wget + pandoc** | 無料 | × | × | 多くの環境にプリインストール |

## 有料 API を選ぶ前に確認しておきたいこと

有料 API は高機能ですが、すべてのユースケースに必要なわけではありません。選定の前に以下の 3 点を確認しておくと、不要なコストを避けられます。

### 1. JS レンダリングが本当に必要か

SPA で構築されたサイトの内容を取得するには JS レンダリングが必要です。しかし、取得したいページが静的 HTML であれば不要です。ドキュメントサイト、技術ブログ、ニュースサイトの多くは静的に配信されています。

JS レンダリングが不要であれば、trafilatura や wget + pandoc といった無料のローカルツールで足ります。

### 2. 検索とスクレイピングの両方が必要か

「検索して上位数件の概要を取得する」のと「特定 URL の中身をスクレイピングする」のは別の機能です。それぞれ別の API を使うと、APIキーの管理が倍になります。

Jina Reader は Search API（`s.jina.ai`）と Reader API（`r.jina.ai`）の両方を提供しており、1 サービスで検索もスクレイピングも完結します。APIキーは無料で発行でき、1,000万トークンが付与されます。

### 3. 月間のリクエスト数はどれくらいか

個人開発や小規模なボットであれば、月数百〜数千リクエスト程度ということが多いです。この規模なら無料枠で収まるサービスも多くあります。

逆に、大量のページを定期クロールする場合は、レート制限や安定性の面で有料サービスのほうが安心です。

## 主なツールの使い方

### Jina Reader — `curl` 一発で Markdown が返る

```bash
curl -s "https://r.jina.ai/https://example.com"
```

これだけで、ページの本文を Markdown に変換した結果が返ってきます。インストール不要で、JS レンダリングにも対応しています。

```bash
# 出力形式の切り替え
curl -s "https://r.jina.ai/https://example.com" \
  -H "x-respond-with: text"  # markdown / html / text

# Search API で Web 検索（APIキーが必要）
curl -s "https://s.jina.ai/?q=Web+scraping+API+comparison" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Jina Reader の無料枠と注意点

Jina Reader は「月間リクエスト上限なし」ですが、**トークン制**です。APIキー発行時に 1,000万トークンが付与され、消費したぶんだけ減っていきます。

| | APIキーなし | APIキーあり（無料） | 有料 |
|---|---|---|---|
| Reader API（`r.jina.ai`） | 20 RPM | 500 RPM | 500 RPM |
| Search API（`s.jina.ai`） | **利用不可** | 100 RPM | 100 RPM |
| トークン消費 | あり | あり | あり |
| 同時接続 | — | 2 | キーランクに依存 |

:::message alert
**Search API は APIキーなしでは使えません。** Reader API（URL → Markdown 変換）は APIキーなしでも 20 RPM で利用可能ですが、Web 検索機能を使うには APIキーの発行が必要です。APIキー自体は無料で発行でき、1,000万トークンが付与されます。
:::

たとえば長めのニュース記事（約4万文字）を取得した場合、数万トークンを消費します（トークン換算はトークナイザに依存するため概算です）。1日数回の検索 + たまにスクレイピングという使い方なら、無料トークンでかなり長く持ちます。使い切ったらトークンを追加購入する従量課金制です。

#### Jina Reader の弱点

万能ではありません。把握しておきたい制限があります。

- **特定サイトで本文抽出が失敗する** — CSS 構造が特殊なサイトでは、ページの一部しか取得できないケースが [GitHub Issues](https://github.com/jina-ai/reader/issues) で報告されています
- **トークンベース課金の不透明さ** — Search API（`s.jina.ai`）は検索結果本文も含むため、1リクエストあたりのトークン消費量が予測しにくいです。想定以上にトークンが減る場合があります
- **外部サービスへの依存** — サービス障害時は代替手段がありません。プロダクション用途ではフォールバックの設計が必要です

### trafilatura — 本文抽出の精度が高い Python CLI

```bash
pip install trafilatura
trafilatura -u "https://example.com" --markdown
```

学術論文にもなっている本文抽出アルゴリズムを使っており、広告やナビゲーションを高精度に除去します。

```bash
# 複数 URL を一括処理
cat urls.txt | trafilatura --markdown > output.md
```

### wget + pandoc — 追加インストールなしで動くことが多い

```bash
wget -q -O- "https://example.com" | pandoc -f html -t markdown
```

本文抽出は行わないため、ナビゲーションやフッターも含めてページ全体が変換されます。静的ページの簡易な変換向きです。

## Jina Reader vs Firecrawl — 何が違うのか

この2つは「URL を渡して Markdown を返す」という用途が重なるため、比較されやすいサービスです。しかしアーキテクチャも料金体系も大きく異なります。

| 観点 | Jina Reader | Firecrawl |
|------|------------|-----------|
| **アーキテクチャ** | クラウド API のみ | クラウド API + セルフホスト可 |
| **取得単位** | 単一ページ | 単一ページ + サイト全体クロール |
| **Web 検索** | あり（`s.jina.ai`） | なし |
| **JS レンダリング** | 対応 | 対応（ブラウザフリート） |
| **本文抽出** | 対応 | 対応 + AI 構造化抽出 |
| **MCP 対応** | なし | あり |
| **レスポンス速度** | 平均 7.9秒（公式ページ掲載値、条件非公開） | ページ複雑度に依存 |
| **料金体系** | トークン制（1,000万トークン無料） | クレジット制（500クレジット無料、以降 $16/月〜） |
| **セルフホスト** | 不可 | 可（Docker 5コンテナ構成） |

### 使い分けの考え方

**Jina Reader が向いているケース:**
- 単一ページの取得が中心
- Web 検索も同じサービスで完結させたい
- セットアップを最小限にしたい

**Firecrawl が向いているケース:**
- サイト全体のクロールが必要（再帰クロール機能は Firecrawl のみ）
- AI による構造化データ抽出を使いたい
- MCP サーバーとして Claude Code 等に統合したい
- 自前サーバーにセルフホストしたい

両者は競合するようでいて、実際にはユースケースが異なります。「1ページ取得してMarkdownにする」だけなら Jina Reader で十分ですが、「サイト全体をクロールして構造化データを抽出する」なら Firecrawl の領域です。

なお、どちらもクラウド API を使う場合、**スクレイプ対象の URL と取得コンテンツが第三者サーバーを経由します**。社内ページや機密情報を含む URL を扱う場合は、trafilatura のようなローカル完結ツールか、Firecrawl のセルフホストを検討してください。

## 判断フローチャート

```
Q1: 月間リクエスト数は？
├── 数千以下
│   Q2: JS レンダリングが必要？
│   ├── No → trafilatura（スクレイピング）/ Jina Search（検索）
│   └── Yes → Jina Reader（無料枠で十分）
└── 数万以上
    Q3: サイト全体のクロールが必要？
    ├── No → Brave Search + Jina Reader
    └── Yes → Firecrawl or ScrapingBee
```

## 実践: GAS ボットで Brave + Firecrawl → Jina Reader に置き換えた

ここからは実際の置き換え事例です。Google Apps Script（GAS）で動く Slack ボットの Web 検索部分をリファクタしました。

### 元の構成

毎朝 AI が生成した挨拶を Slack に投稿するボットを運用しており、挨拶に「最新ニュース」を織り交ぜるために Web 検索 API を使っていました。

```
Brave Search API（メイン検索）→ Firecrawl API（URL スクレイピング + フォールバック）
```

動いてはいたのですが、不満が 3 つありました。

1. **APIキーが 2 つ必要** — Brave と Firecrawl、それぞれ取得・管理が必要
2. **Firecrawl のコスト** — 無料枠は 500クレジット（1回限り）。継続利用なら Hobby プランで $16/月〜。フォールバック用途にしては割高
3. **GAS の Script Properties が増える一方** — `BRAVE_API_KEY`、`FIRECRAWL_API_KEY`...

「検索して上位数件の概要を取得する」という単純な要件に対して、インフラが重すぎました。

### Jina Reader に一本化

Jina Reader の Search API（`s.jina.ai`）を使えば、検索もスクレイピングも 1 サービスで完結します。

```javascript
// GAS: Web 検索（Search API は APIキー必須）
function searchWithJina(query) {
  const apiKey = PropertiesService.getScriptProperties().getProperty("JINA_API_KEY");
  const url = "https://s.jina.ai/?q=" + encodeURIComponent(query);
  const response = UrlFetchApp.fetch(url, {
    headers: {
      "Accept": "application/json",
      "Authorization": "Bearer " + apiKey,
    },
    muteHttpExceptions: true,
  });

  if (response.getResponseCode() !== 200) return null;
  return JSON.parse(response.getContentText());
}
```

```javascript
// GAS: URL の中身を Markdown で取得（Reader API はキーなしでも動作）
function readUrl(targetUrl) {
  const response = UrlFetchApp.fetch("https://r.jina.ai/" + targetUrl, {
    muteHttpExceptions: true,
  });
  if (response.getResponseCode() !== 200) return null;
  return response.getContentText();
}
```

どちらも **GET リクエスト一発**です。Search API の利用には APIキーが必要ですが、無料で発行できます。Reader API は APIキーなしでも動きます。

### Before / After

| 観点 | Brave + Firecrawl | Jina Reader |
|------|-------------------|-------------|
| 検索 | Brave Search API | `s.jina.ai` |
| スクレイピング | Firecrawl `/v1/scrape` | `r.jina.ai` |
| APIキー | 2つ必要 | 1つ（無料発行） |
| HTTP メソッド | GET + POST | GET のみ |
| JS レンダリング | Firecrawl のみ | 対応 |
| 管理する秘密情報 | `BRAVE_API_KEY` + `FIRECRAWL_API_KEY` | `JINA_API_KEY` のみ |
| 無料枠 | Brave: 月$5クレジット、Firecrawl: 500クレジット（1回限り） | 1,000万トークン（使い切り、追加購入可） |

### GAS 固有の制約との相性

GAS には独特の制約があり、Jina Reader はそれとよく噛み合いました。

**npm が使えない** — GAS には `npm install` がありません。外部ライブラリの導入には手間がかかります。Jina Reader は `UrlFetchApp.fetch()` だけで完結するので、追加ライブラリが一切不要です。

**6分の実行時間制限** — GAS のトリガー実行は最大 6 分です。外部 API のレスポンス速度が重要になります。Jina Reader は単純な HTTP GET なので、POST + JS レンダリング待ちの Firecrawl と比べて体感で速く感じました（定量ベンチマークではありません）。

**Script Properties の管理コスト** — GAS の秘密情報管理は Script Properties で行いますが、GAS エディタでしか編集できず、`.env` ファイルのような管理もできません。APIキーが 1 つ減るだけで運用負荷が体感で変わります。

:::message
GAS ネイティブ（UrlFetchApp + Cheerio ライブラリ）で完結させる方法も検討しましたが、Web 検索機能がない・JS レンダリング不可・サイトごとにセレクタを書く必要がある、という理由で見送りました。
:::

### フォールバック設計

置き換えでは Gateway 層のインターフェースを維持しつつ、内部実装を差し替えました。

```
変更前: Brave Search → Firecrawl（フォールバック）
変更後: Jina Search → Brave Search（フォールバック）
```

Jina をメインに昇格し、Brave をフォールバックに降格しています。Firecrawl は削除しました。

```javascript
function searchTopicNews(topic) {
  // 1. Jina Search（メイン）
  const jinaResult = searchWithJina(topic);
  if (jinaResult) return jinaResult;

  // 2. Brave Search（APIキーが設定されている場合のみ）
  const braveResult = searchWithBrave(topic);
  if (braveResult) return braveResult;

  return null;
}
```

ポイントは「APIキーの管理コストが低いサービスをメインにする」ことです。Jina は無料で発行できる 1 つのキーで検索もスクレイピングも完結します。Brave は検索精度が高いのでフォールバックとして残す価値があります。

Gateway 層が API の差異を吸収し、上位の Service 層には同じ型を返します。レイヤードアーキテクチャの恩恵で、置き換えの影響範囲が Gateway 内に閉じました。

## 補足: Firecrawl のコスト感

Firecrawl を有料プランで使う場合の参考値です。

| プラン | 月額（年払い） | クレジット/月 | 同時リクエスト |
|--------|---------------|--------------|---------------|
| Free | $0 | 500（1回限り） | 2 |
| Hobby | $16 | 3,000 | 5 |
| Standard | $83 | 100,000 | 50 |
| Growth | $333 | 500,000 | 100 |

Hobby プランの場合、1ページあたり約 $0.005（約0.7円）です。個人開発で月数百ページ程度なら Hobby で足りますが、「検索して上位数件を取得する」程度の用途であれば、Jina Reader の無料トークンで十分まかなえます。

## まとめ

| 学び | 内容 |
|------|------|
| 有料 API の前に確認 | JS レンダリングの要否、検索/スクレイピングの要否、月間リクエスト数 |
| 小規模なら無料トークンで十分 | Jina Reader は無料発行の APIキー 1 つで検索もスクレイピングも使える |
| GAS での判断軸 | APIキーの管理コストが最優先。キーが少ないほど運用が楽 |
| フォールバック設計 | 管理コストの低いサービスをメインにして、高精度サービスをフォールバックに |
| 段階的アプローチ | まず最軽量の方法で試し、限界を感じたら次のレイヤーに進む |
