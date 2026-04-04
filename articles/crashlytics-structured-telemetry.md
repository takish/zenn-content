---
title: "Crashlyticsのログを構造化テレメトリに変える — AI解析しやすいログ設計の実践"
emoji: "🔍"
type: "tech"
topics: ["firebase", "crashlytics", "bigquery", "swift", "observability"]
published: false
---

# CrashlyticsのログをAIが解析しやすくするための構造化テレメトリ設計

Firebase MCPの登場で、Claude CodeやCursorからCrashlyticsのクラッシュログを直接取得・分析できるようになりました（Crashlytics関連のMCPツールは2026年3月時点でExperimentalステータスです）。「過去1週間で最も影響の大きいクラッシュを教えて」と聞けば、AIがCrashlyticsのデータを引いて回答してくれます。

ただ、AIに解析させるとき、精度を左右するのは**AIの性能ではなく、送る側のログ設計**です。

Crashlyticsの `log()` にフリーテキストを渡していると、BigQueryにエクスポートしても `logs.message` にはただの文字列が入るだけです。AIはまずそのフォーマットを推測するところから始まることになります。一方、JSON構造化されたログなら、キー名から意味が自明なので、パターン検出や異常分析に集中できます。

筆者はiPad向けの業務アプリで、外部アプリ連携を含む複雑なフローのデバッグに苦労した経験があります。フリーテキストのログをBigQueryに流してAIに「異常パターンを見つけて」と聞いても、返ってくるのは曖昧な推測ばかりでした。ログをJSON構造化してからは、AIが「`auth_flow` の `token_refresh` ステップで `error_code: 401` が集中している」のように具体的な分析を返すようになり、原因特定までの時間が明らかに短くなりました。

この記事では、Crashlyticsに送るログを「AIが解析しやすい形」に設計する方法と、その構造化ログをどう活用するかをセットで紹介します。

---

## この記事で得られること

- Crashlyticsの3機能（Breadcrumb / Custom Keys / Non-Fatal）の使い分けの判断基準
- AIやBigQueryで解析しやすいJSON構造化のフォーマット設計
- Flow IDによるトレーシングの仕組み
- 構造化ログの活用方法（Firebase MCP → BigQuery → AIスキルの3段階）
- 設計時に気をつけるべき制限事項とアンチパターン

---

## 前提: Crashlyticsの3つのテレメトリ機能

Crashlyticsには3つの記録方法があります。それぞれ用途と制限が異なるので、「何をどこに送るか」を意識して使い分ける必要があります。

| 機能 | API | 用途 | 制限 | BigQuery上の列 |
|------|-----|------|------|---------------|
| **Breadcrumb** | `log()` | イベントの時系列記録 | 最新64KB（超過分は古い方から消える） | `logs` 配列 |
| **Custom Keys** | `setCustomValue()` | 状態のスナップショット | 最大64個、各1KBまで | `custom_keys` 配列 |
| **Non-Fatal** | `record(error:)` | エラー記録（Issuesに表示） | セッション内最大8個 | `error` 配列 |

出典: [Customize crash reports (Apple platforms)](https://firebase.google.com/docs/crashlytics/ios/customize-crash-reports)

### 使い分けの判断基準

```
フローの途中経過を追いたい → Breadcrumb (log())
クラッシュ時点の状態を知りたい → Custom Keys (setCustomValue())
Issuesタブでエラーを管理したい → Non-Fatal (record(error:))
```

**Breadcrumb (`log()`)** — クラッシュやNon-Fatalエラーが発生した時に、その直前のログがレポートに含まれます。「何が起きてクラッシュに至ったか」を時系列で追跡するためのものです。64KBを超えると古いエントリから消えるため、ループ内での大量出力は避けます。

**Custom Keys (`setCustomValue()`)** — 最後に設定された値だけが残ります。「クラッシュした瞬間、どのステップにいたか」を一発で特定するのに使います。時系列の記録には向きません。

**Non-Fatal (`record(error:)`)** — Crashlytics Consoleの Issues タブに表示されます。`domain` と `code` でグルーピングされるため、一意な値（タイムスタンプ、ユーザーIDなど）を `domain` や `code` に入れるとカーディナリティが爆発します。一意な識別子は `userInfo` に含めるのが公式の推奨です。

また、**セッション内で8個を超えるとNon-Fatalは古いものから破棄される**ため、頻発するイベントをNon-Fatalで記録するのは適切ではありません。頻発するイベントはBreadcrumbで記録し、Non-Fatalは「本当に把握すべきエラー」に限定します。

:::message
筆者の経験では、リトライ処理のたびに `record(error:)` を呼んでいた時期がありました。結果、セッション内のNon-Fatalが8個の上限に達し、本当に重要なエラー（認証失敗）が破棄されてCrashlyticsのIssuesに現れないという事態になりました。この仕様は公式ドキュメントにも記載がありますが、実際にぶつかるまで気づきにくいポイントです。
:::

---

## 設計: JSON構造化ログのフォーマット

### なぜJSON構造化するのか

Crashlyticsの `log()` に渡したメッセージは、BigQueryの `logs.message` カラムにそのまま文字列として格納されます（[BigQuery dataset schema](https://firebase.google.com/docs/crashlytics/bigquery-dataset-schema)）。

フリーテキストで送った場合:

```
リトライ開始 3件
外部APIに送信した
コールバック受信OK orderId=abc123
```

BigQueryで検索するには `LIKE '%リトライ%'` のような曖昧なクエリしかできません。AI解析でも、まずフォーマットの推測から始まります。

JSON構造化した場合:

```json
{"event":"sync_flow","step":"retry_start","ctx":{"count":"3"}}
{"event":"sync_flow","step":"api_request_sent","ctx":{"endpoint":"/api/orders"}}
{"event":"sync_flow","step":"callback_received","ctx":{"orderId":"abc123"}}
```

`JSON_EXTRACT_SCALAR(log.message, '$.step')` で正確にフィールドを参照でき、AIもキー名から意味を即座に把握できます。

### フォーマット定義

```json
{
  "event": "sync_flow",
  "step": "retry_start",
  "flowId": "a1b2c3d4",
  "ctx": { "count": "3", "endpoint": "/api/orders" }
}
```

| フィールド | 必須 | 説明 |
|-----------|------|------|
| `event` | ✅ | ドメイン名。`sync_flow`, `auth_flow`, `api_request` など |
| `step` | ✅ | フロー内のステップ。`{action}_{result}` 形式を推奨 |
| `flowId` | — | フロー1回分を紐付けるトレースID |
| `ctx` | — | 追加コンテキスト。デバッグに必要な最小限の情報 |

### ステップの命名規則

```
{action}_{result}
```

例: `retry_start`, `api_request_sent`, `callback_received`, `signIn_failed`

ドメインは `event` フィールドで分けます:

```swift
AppLogger.logEvent("sync_flow", step: "...")     // データ同期
AppLogger.logEvent("auth_flow", step: "...")      // 認証
AppLogger.logEvent("api_request", step: "...")     // API通信
```

---

## 実装: テレメトリラッパー

Crashlyticsへの送信を一箇所にまとめるラッパーです。実際のプロダクトで運用している構造をベースに、汎用化した例を示します。コード例はSwiftですが、CrashlyticsのAPIはiOS/Androidで同じ構造なので、Kotlinでも同様の設計が適用できます。

### AppLogger 基本構造

```swift
import os.log
import FirebaseCrashlytics

struct AppLogger {
    private let logger: Logger
    let category: String

    private static let subsystem = Bundle.main.bundleIdentifier ?? "app"

    init(category: String) {
        self.category = category
        self.logger = Logger(subsystem: Self.subsystem, category: category)
    }

    // MARK: - Log Levels

    func debug(_ message: String) { logger.debug("\(message)") }
    func info(_ message: String) { logger.info("\(message)") }
    func warning(_ message: String) { logger.warning("\(message)") }

    /// エラーログ（Crashlyticsに自動送信）
    /// context は [String: Any] を受け取る（NSError の userInfo にマージするため）。
    /// テレメトリ側の logEvent() は [String: String] に限定している（JSONシリアライズの安全性のため）。
    func error(_ message: String, error: Error? = nil, context: [String: Any]? = nil) {
        logger.error("\(message)")
        recordToCrashlytics(message: message, error: error, context: context)
    }

    // MARK: - Crashlytics Non-Fatal

    private func recordToCrashlytics(
        message: String, error: Error?, context: [String: Any]?
    ) {
        var userInfo: [String: Any] = [
            "message": message,
            "category": category
        ]
        if let context { userInfo.merge(context) { _, new in new } }
        if let error {
            userInfo["originalError"] = error.localizedDescription
        }

        // domain にカテゴリを含め、BigQuery の error_type でフィルタ可能にする
        // 一意な値は userInfo に入れ、domain + code はグルーピング用に固定値を使う
        let nsError = NSError(
            domain: "\(Self.subsystem).\(category)",
            code: (error as NSError?)?.code ?? -1,
            userInfo: userInfo
        )
        Crashlytics.crashlytics().record(error: nsError)
    }

    /// View層など、インスタンスなしでエラーを記録する場合
    static func recordError(
        _ message: String, error: Error? = nil,
        category: String, context: [String: Any]? = nil
    ) {
        let logger = AppLogger(category: category)
        logger.error(message, error: error, context: context)
    }
}
```

`category` がNSErrorの `domain` に反映されます。`AppLogger(category: "Sync")` で送ったエラーは `com.example.app.Sync` というドメインでCrashlyticsに記録されます。BigQuery上では `error` 配列内のサブフィールドとして格納されます（筆者の環境では `error.title` にNSErrorの `domain` が入ることを確認しています）。`UNNEST(error)` で展開してフィルタリングできます。

### Breadcrumb + Flow ID + Custom Keys

```swift
extension AppLogger {
    private static let telemetryLogger = AppLogger(category: "Telemetry")
    /// Note: プロダクションでは actor や DispatchQueue での排他制御を追加してください
    private(set) static var currentFlowId: String?

    // MARK: - JSON構造化ログ（Breadcrumb）

    static func logEvent(
        _ event: String,
        step: String,
        context: [String: String]? = nil
    ) {
        var payload: [String: Any] = ["event": event, "step": step]
        if let flowId = currentFlowId {
            payload["flowId"] = flowId
        }
        if let context, !context.isEmpty {
            payload["ctx"] = context
        }

        let message: String
        if let data = try? JSONSerialization.data(withJSONObject: payload),
           let json = String(data: data, encoding: .utf8) {
            message = json
        } else {
            message = "[\(event)] \(step)"
        }

        telemetryLogger.info(message)
        Crashlytics.crashlytics().log(message)
    }

    /// ドメイン固定の短縮版
    static func syncFlow(_ step: String, context: [String: String]? = nil) {
        logEvent("sync_flow", step: step, context: context)
    }

    // MARK: - Flow ID（トレーシング）

    @discardableResult
    static func startFlow() -> String {
        let flowId = UUID().uuidString.prefix(8).lowercased()
        currentFlowId = String(flowId)
        Crashlytics.crashlytics().setCustomValue(flowId, forKey: "flow_id")
        return String(flowId)
    }

    static func endFlow() {
        currentFlowId = nil
        Crashlytics.crashlytics().setCustomValue("", forKey: "flow_id")
    }

    // MARK: - Custom Keys（状態スナップショット）

    static func setCurrentStep(_ step: String) {
        Crashlytics.crashlytics().setCustomValue(step, forKey: "current_step")
    }
}
```

Flow IDはOpenTelemetryのTrace IDに相当する仕組みで、散在するログを「フロー1回分」として紐付けます。UUID短縮8文字（16^8 = 約43億通り）なので、実用上の衝突リスクはほぼありません。

### 使い方

```swift
// フロー開始
AppLogger.startFlow()
AppLogger.setCurrentStep("syncOrders_start")
AppLogger.syncFlow("syncOrders_start", context: ["count": "\(orders.count)"])
// → {"event":"sync_flow","step":"syncOrders_start","flowId":"a1b2c3d4","ctx":{"count":"5"}}

// 外部API呼び出し
AppLogger.setCurrentStep("waitForResponse")
AppLogger.syncFlow("api_request_sent", context: ["endpoint": "/api/orders"])

// 完了
AppLogger.setCurrentStep("syncOrders_success")
AppLogger.syncFlow("syncOrders_success")
AppLogger.endFlow()

// エラー発生時はインスタンスメソッド側でNon-Fatalを記録
let logger = AppLogger(category: "Sync")
logger.error("Order sync failed", error: error, context: [
    "orderId": orderId,
    "statusCode": "\(response.statusCode)"
])
```

各ステップで3つのことが同時に起きます:

1. **Breadcrumb**: JSON構造化ログが `logs.message` に時系列で記録される
2. **Custom Keys**: `current_step` に最新のステップが上書きされる
3. **Flow ID**: すべてのログが同じ `flowId` で紐付けられる

---

## 解析: 3段階の活用方法

ログを構造化して送る「設計」だけでなく、**どう解析するか**までセットで用意すると、構造化の恩恵を最大限に引き出せます。活用方法は手軽さと詳細さのトレードオフで3段階に分かれます。

```
手軽                                          詳細
├── Firebase MCP ──┤── BigQuery クエリ ──┤── AIスキル ──┤
   自然言語で即座に       SQLで正確に          テンプレート化して
   問い合わせ可能         フィルタ・集計        自動化
```

### Stage 1: Firebase MCPで手軽に問い合わせる

Firebase MCPを使えば、Claude CodeやCursorからCrashlyticsのデータを直接取得できます。「過去24時間のTop Issuesを教えて」のような自然言語での問い合わせが可能です。セットアップ方法は[Firebase公式ドキュメント](https://firebase.google.com/docs/crashlytics)や[先行記事](https://zenn.dev/lclco/articles/firebase-mcp-crashlytics-auto-fix)を参照してください。

ここで構造化ログの効果が出ます。Firebase MCPが返すイベント詳細にBreadcrumbが含まれていれば、AIは「`sync_flow` の `waitForResponse` ステップで `error_code: 500` が集中している」と即座に判断できます。フリーテキストだと「外部APIに送信した」という文字列から状況を推測する必要があり、分析の精度が落ちます。

ただし、Firebase MCPで取得できるのはCrashlytics Consoleで見える範囲の情報（Top Issues、個別issueの詳細、Breadcrumbなど）です。`logs.message` の中身をSQLでフィルタ・集計したり、`flowId` でフロー全体を再構成するといった操作はできません。「どのクラッシュが多いか」の概要把握にはFirebase MCPで十分ですが、「このフローの中で何が起きたか」を深掘りするにはBigQueryが必要になります。

**向いているケース**: 日常的なクラッシュ調査、「今何が起きているか」の概要把握

### Stage 2: BigQueryで詳細に分析する

Firebase MCP では取得できないレベルの詳細分析（時系列集計、デバイス相関、フロー再構成）にはBigQuery Exportを使います。

:::message
**BigQuery Export の有効化手順**: Firebase Console → Crashlytics → 設定 → BigQuery リンクを有効化。Blazeプラン（従量課金）が必要です。ストリーミングエクスポートを有効にするとニアリアルタイムで反映されますが、Streaming Insertの料金が別途かかります。日次バッチであれば追加コストはほぼありません。小〜中規模のアプリであれば日次バッチで十分なケースが多いです。初回エクスポートには最大24時間かかります。
:::

以下は、構造化ログを前提としたクエリテンプレートです。

**カテゴリ別のエラー抽出** — 「Syncカテゴリのエラーだけ見たい」ときに使います:

```sql
SELECT
  event_timestamp,
  issue_id,
  error_type,
  e.title AS error_title,
  e.subtitle AS error_subtitle,
  blame_frame.symbol AS symbol
FROM `project.dataset.app_table`,
  UNNEST(error) AS e
WHERE error_type = 'NON_FATAL'
  AND e.title LIKE '%Sync%'
ORDER BY event_timestamp DESC
LIMIT 50
```

**Flow IDでフロー再構成** — 「特定のフロー1回分を時系列で追いたい」ときに使います:

```sql
SELECT
  log.timestamp,
  JSON_EXTRACT_SCALAR(log.message, '$.event') AS event,
  JSON_EXTRACT_SCALAR(log.message, '$.step') AS step,
  JSON_EXTRACT(log.message, '$.ctx') AS context
FROM `project.dataset.app_table`,
  UNNEST(logs) AS log
WHERE JSON_EXTRACT_SCALAR(log.message, '$.flowId') = 'a1b2c3d4'
ORDER BY log.timestamp
```

**特定ステップでのクラッシュ抽出** — 「どのステップでクラッシュが多いか」を調べるときに使います:

```sql
SELECT event_timestamp, issue_id, device.model
FROM `project.dataset.app_table`
WHERE EXISTS(
  SELECT 1 FROM UNNEST(custom_keys) AS ck
  WHERE ck.key = 'current_step' AND ck.value = 'waitForResponse'
)
```

**issue_id 別のトレンド** — 「直近7日間で増えているエラーはどれか」を把握するときに使います:

```sql
SELECT
  issue_id,
  error_type,
  COUNT(*) AS count,
  MIN(event_timestamp) AS first_seen,
  MAX(event_timestamp) AS last_seen
FROM `project.dataset.app_table`
WHERE event_timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND error_type = 'NON_FATAL'
GROUP BY 1, 2
ORDER BY count DESC
LIMIT 20
```

**向いているケース**: 特定条件での深掘り、トレンド分析、フロー全体の再構成

### Stage 3: AIスキルとして自動化する

Stage 2のクエリを毎回手で書くのは手間です。筆者のプロジェクトでは、これらのクエリテンプレートをClaude Codeのスキル（プロジェクト固有のコマンド）として組み込んでいます。

```
/crashlytics 3/27 16:00-16:30のSyncエラー
```

このように指示すると、以下が自動で行われます:

1. JST → UTC のタイムゾーン変換（BigQueryの `event_timestamp` はUTC）
2. 適切なクエリテンプレートの選択・`bq query` での実行
3. 結果の分析（エラーパターンの特定、バージョン・デバイスとの相関、`blame_frame` からの原因箇所特定）

スキルに持たせている情報は3つです:

**カテゴリ対応表** — AIが「Syncのエラー」と言われたときにBigQueryのどのフィールドを検索すべきかがわかる:

```markdown
| AppLogger category | NSError domain (= error.title) |
|-------------------|-------------------------------|
| Sync              | com.example.app.Sync          |
| Auth              | com.example.app.Auth          |
| API               | com.example.app.API           |
| Network           | com.example.app.Network       |
```

**タイムゾーン変換ルール** — ユーザーのJST指定をBigQueryのUTCに自動変換するための指示。

**分析の観点** — クエリ結果に対して何を見るかの指針:
- 同一 `issue_id` のグルーピングによるエラーパターン特定
- バージョン別・デバイス別の発生頻度
- `custom_keys` からの追加コンテキスト抽出
- `blame_frame` からの原因箇所特定

構造化ログがあるからこそ、AIスキルによる自動化が「使える精度」で動きます。フリーテキストだとクエリテンプレート自体が成り立ちません。

**向いているケース**: チームでの定常的なエラー調査、オンコール対応の効率化

---

## 避けるべきパターン

### 機密情報をログに含めてしまう

Crashlyticsのログは BigQuery にエクスポートされる可能性があります。チームメンバー全員、連携ツールからも見える前提で設計します。

```swift
// ❌
Crashlytics.crashlytics().log("User: \(user.email), Token: \(token)")

// ✅ 内部識別子のみ
AppLogger.syncFlow("retry_success", context: ["orderId": orderId])
```

- 個人情報（メール、名前、電話番号）→ ログしない
- 認証情報（トークン、パスワード、APIキー）→ ログしない
- 内部ID → OK（デバッグに必要な最小限）

### Breadcrumbの64KB上限を圧迫する

ループ内で大量にログを吐くと、重要なログが押し出されます。JSON構造化するとフリーテキストより1エントリあたりのサイズが大きくなる（100〜200バイト程度）ため、64KBに入るエントリ数は300〜650件程度が目安です。

```swift
// ❌ リスト全件 → 数十KBを一瞬で消費
for item in items {
    Crashlytics.crashlytics().log("Item: \(item.name)")
}

// ✅ サマリーだけ
AppLogger.syncFlow("batch_summary", context: [
    "itemCount": "\(items.count)"
])
```

### Non-Fatalを頻発イベントに使う

セッション内で**8個を超えるNon-Fatalは古いものから破棄**されます。リトライのたびに `record(error:)` を呼ぶと、本当に重要なエラーが消える可能性があります。

```swift
// ❌ リトライごとにNon-Fatal → 8個を超えると消える
for attempt in 1...maxRetries {
    AppLogger.recordError("Retry failed", category: "Sync")
}

// ✅ リトライ経過はBreadcrumb、最終失敗だけNon-Fatal
for attempt in 1...maxRetries {
    AppLogger.syncFlow("retry_attempt", context: ["attempt": "\(attempt)"])
}
AppLogger.recordError("All retries failed", category: "Sync")
```

### Custom KeysをBreadcrumb代わりに使う

Custom Keysは最後の値しか残りません。時系列の記録にはBreadcrumbを使います。

```swift
// ❌ 上書きされて "step_1" は消える
Crashlytics.crashlytics().setCustomValue("step_1", forKey: "last_event")
Crashlytics.crashlytics().setCustomValue("step_2", forKey: "last_event")

// ✅ 時系列はBreadcrumb、現在の状態はCustom Key
Crashlytics.crashlytics().log("step_1")
Crashlytics.crashlytics().log("step_2")
Crashlytics.crashlytics().setCustomValue("step_2", forKey: "current_step")
```

---

## まとめ

Firebase MCPの登場で、AIにCrashlyticsのデータを直接解析させることが現実的になりました。ただ、AIが返す分析の精度は、送る側のログ設計に依存します。

**送る側の設計**:

| 要素 | 内容 |
|------|------|
| JSON構造化 | `event` + `step` + `flowId` + `ctx` の統一フォーマット |
| 3機能の使い分け | 時系列 → Breadcrumb、状態 → Custom Keys、エラー管理 → Non-Fatal |
| Flow ID | 1回のフローに一意のIDを振り、散在ログを紐付ける |
| カテゴリ分類 | `AppLogger(category:)` でNSErrorの `domain` にカテゴリを付与 |

**解析する側の仕組み**:

| Stage | 方法 | 向いているケース |
|-------|------|----------------|
| 1 | Firebase MCP | 日常的なクラッシュ調査。自然言語で即座に問い合わせ |
| 2 | BigQuery | 時系列集計、デバイス相関、フロー再構成など詳細分析 |
| 3 | AIスキル | BigQueryクエリのテンプレート化・自動化。定常的な運用 |

構造化してログを送り、その構造を前提にした解析の仕組みを用意する。この2つがセットになることで、Crashlyticsのテレメトリが「送りっぱなし」ではなく「AIが活用できるデータ」になります。

---

## 参考リンク

- [Customize crash reports (Apple platforms) | Firebase Crashlytics](https://firebase.google.com/docs/crashlytics/ios/customize-crash-reports)
- [Export Crashlytics data to BigQuery | Firebase Crashlytics](https://firebase.google.com/docs/crashlytics/bigquery-export)
- [BigQuery dataset schema | Firebase Crashlytics](https://firebase.google.com/docs/crashlytics/bigquery-dataset-schema)
- [Firebase MCP × Claude Code Action でクラッシュを自動修正する](https://zenn.dev/lclco/articles/firebase-mcp-crashlytics-auto-fix)
- [Firebase MCPでモバイルアプリのクラッシュ対応を自動化する | TVer Tech Blog](https://techblog.tver.co.jp/entry/negishi/firebase-mcp-crashlytics)
