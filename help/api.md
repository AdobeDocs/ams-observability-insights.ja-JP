---
source-git-commit: e5523081fcd68500602e5d1bf853694d1f6c3980
workflow-type: tm+mt
source-wordcount: '1104'
ht-degree: 7%

---
# Observability Insights パブリック API

Observability Insights Public APIを使用すると、リクエストの概要、サービスカタログ、トレース、指標などの独自のオブザーバビリティデータを、独自のツール、スクリプト、ダッシュボードに直接取り込むことができます。

- **API ベース URL （API_BASE_URL）:** `https://insights.adobecqms.net/`
- **形式：** JSON over HTTPS
- **認証：** API キー（ベアラートークン）

> このドキュメント全体の`{{API_BASE_URL}}`を、Observability Insights インスタンスのAPI ホスト （例：`https://insights.adobecqms.net/`）に置き換えます。

---

## &#x200B;1. API キーの取得

API キーは、アカウントに関連付けられ、単一の組織にスコープされる個人の資格情報です。 キーは、作成された組織に属するテナントのデータのみを読み取ることができます。他の組織のデータを表示することはできません。

### キーを生成

1. [Observability Insights ダッシュボード ](https://insights.adobecqms.net/)にログインします。
2. **API キー**→プロファイルメニュー（右上）を開きます。
   ![API キーメニュー](v2-assets/api-key.png)
3. 「**API キー**」タブで、「**キーを生成**」をクリックします。
   ![API キーを生成](v2-assets/api-key-gen.png)
4. わかりやすい名前（例：`CI pipeline`、`Grafana datasource`）を付け、スコープを設定する組織を選択し、オプションで有効期限を設定します。
5. 「**キーを生成**」をクリックします。 キーは&#x200B;**once**&#x200B;の形式で表示されます。

   ```
   synx_9pQ2v6f1WYbLZk3n0aRtEo4jXcHsVmDgUiPq7B8l1yc
   ```

   **すぐにコピーして安全な場所に保存** （シークレットマネージャー、CI シークレットストアなど） — ダッシュボードで再度表示することはできません。 紛失した場合は、取り消して新しく生成します。

### 既存のキーの管理

「API キー」セクションには、組織、作成日、有効期限、最後に使用したタイムスタンプなど、作成したすべてのキーが一覧表示されます。 キーの横にあるごみ箱アイコンをクリックして&#x200B;**取り消し**&#x200B;します。取り消しは即座に行われ、取り消しを元に戻すことはできません。

### 主要なセキュリティ

- API キーをパスワードとまったく同じように扱います。 キーを持つ人は誰でも、そのキーが失効するか期限切れになるまで、対象の組織内のすべてのテナントに関するすべてのオブザーバビリティ データを読み取ることができます。
- キーをソースコントロールにコミットしたり、プレーンテキスト（チャット、メール、チケット）で共有したりしないでください。
- キーを定期的に回転させ、使用されなくなったキーを取り消します。
- キーが漏洩した場合は、**組織の設定→ API キー**&#x200B;から直ちに取り消して、代わりのキーを生成します。

---

## &#x200B;2. リクエストの認証

パブリック APIへのあらゆるリクエストには、キーを`Authorization` ヘッダーに含める必要があります。

```
Authorization: Bearer synx_9pQ2v6f1WYbLZk3n0aRtEo4jXcHsVmDgUiPq7B8l1yc
```

有効なキーを持たないリクエスト、または期限切れ/失効したキーを持つリクエストは、`401 Unauthorized`を受け取ります。 セッションのログイン（ブラウザーのCookie/トークン）は、このAPIで&#x200B;**not**&#x200B;受け入れられます。

---

## &#x200B;3. 基本コンセプト

### テナント

各エンドポイントには、どのテナントのデータを読み取るかを識別する`tenant_id` クエリパラメーターが必要です。 キーは、作成された組織に属するテナントのみをクエリできます。その組織外のテナントをリクエストすると、`403 Forbidden`が返されます。 このAPIには「すべてのテナント」モードはありません。常に特定の`tenant_id`を渡します。

キーで使用できる`tenant_id`の値がわかりませんか？ [`GET /public/v1/tenants`](#get-publicv1tenants)を呼び出す – キーがクエリを許可されているテナントを正確に一覧表示します。

### 時間範囲

`from` / `to` パラメーターを受け入れるエンドポイントは、Unix タイムスタンプ （秒）、ミリ秒タイムスタンプ、またはISO 8601 datetime文字列を受け取ります。例：

```
from=1735689600
from=2025-01-01T00:00:00Z
```

省略した場合、ほとんどのエンドポイントは最近のローリングウィンドウにデフォルトで設定されます（以下の各エンドポイントを参照）。

### レート制限

リクエストは、API キーごとにレート制限があります。 制限を超えると、以下が表示されます。

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60

{ "error": "Too Many Requests", "message": "Rate limit of 300 requests/60s exceeded" }
```

`Retry-After` ヘッダーの秒数を過ぎてから戻り、再試行してください。 ユースケースで上限が必要な場合は、サポートにお問い合わせください。

### エラー

エラーは、`error` フィールドと、通常は人間が読み取れる`message`を持つJSONとして返されます。

```json
{ "error": "Bad Request", "message": "tenant_id is required" }
```

| ステータス | 意味 |
| ------------------------- | ------------------------------------------------------------------ |
| `400 Bad Request` | パラメーターが見つからないか無効です（例：`tenant_id`、不適切な時間範囲） |
| `401 Unauthorized` | API キーが見つからない、無効、期限切れ、失効している |
| `403 Forbidden` | 要求されたテナントに対してキーが承認されていません |
| `429 Too Many Requests` | レート制限を超えました – `Retry-After`を参照してください |
| `502 Bad Gateway` | アップストリームクエリが失敗しました – 再試行しても安全です |
| `503 Service Unavailable` | データバックエンドは一時的に利用できません |

---

## &#x200B;4. エンドポイント

### `GET /public/v1/tenants`

キーがクエリを実行する権限を持つテナント IDを一覧表示します。 最初に呼び出します。他のすべてのエンドポイントでは、これらの値の1つが`tenant_id`として必要です。

```bash
curl -s "{{API_BASE_URL}}/public/v1/tenants" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{ "tenants": ["tenant1", "tenant2"] }
```

### `GET /public/v1/overview`

一定期間におけるテナントの高レベルの正常性KPI （リクエスト量、エラー率、待ち時間のパーセンタイル）です。

| Param | 必須 | 説明 |
| ------------ | -------- | ------------------------------------------------------------------------- |
| `tenant_id` | はい | クエリするテナント |
| `from`, `to` | いいえ | 時間範囲（[時間範囲](#time-ranges)を参照） |
| `minutes` | いいえ | `from`/`to`が指定されていない場合の「最後のN分」の短縮形（デフォルトは`15`） |

```bash
curl -s "{{API_BASE_URL}}/public/v1/overview?tenant_id=<tenant_id>&minutes=30" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "from": 1735689000,
  "to": 1735690800,
  "total_spans": 48213,
  "errors": 112,
  "error_rate_pct": 0.23,
  "p50_ms": 34,
  "p95_ms": 210,
  "p99_ms": 480,
  "service_count": 12,
  "trace_count": 9021
}
```

### `GET /public/v1/services`

テナントの異なるサービス名のレポートを一覧表示します。

| Param | 必須 | 説明 |
| ------------ | -------- | --------------------------------------------------------------------- |
| `tenant_id` | はい | クエリするテナント |
| `from`, `to` | いいえ | このウィンドウに表示されるサービスに制限します。デフォルトは過去7日間です |

```bash
curl -s "{{API_BASE_URL}}/public/v1/services?tenant_id=<tenant_id>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "services": ["checkout-api", "payments-worker", "web-frontend"]
}
```

### `GET /public/v1/traces`

オプションのフィルターを使用して、テナントの最近のトレースを検索します。

| Param | 必須 | 説明 |
| ----------------- | -------- | ------------------------------------------------- |
| `tenant_id` | はい | クエリするテナント |
| `from`, `to` | いいえ | 期間。デフォルトは過去24時間です。 |
| `limit` | いいえ | 返す最大行数（1 ～ 200、デフォルトは100） |
| `offset` | いいえ | ページネーションのオフセット（デフォルトは0） |
| `service` | いいえ | サービス名でフィルタリング |
| `app_name` | いいえ | アプリケーション/インスタンス名でフィルタリング |
| `status` | いいえ | トレース ステータスでフィルター：`ok`、`error`または`unset` |
| `search` | いいえ | スパン/操作名を跨いだフリーテキスト検索 |
| `min_duration_ms` | いいえ | このデュレーション以上のトレースのみ |

```bash
curl -s "{{API_BASE_URL}}/public/v1/traces?tenant_id=<tenant_id>&status=error&limit=25" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "data": [
    {
      "TraceId": "4bf92f3577b34da6a3ce929d0e0e4736",
      "ServiceName": "checkout-api",
      "DurationMs": 812,
      "StatusCode": "Error",
      "Timestamp": "2026-08-30T09:12:44Z"
    }
  ],
  "rows": 137,
  "limit": 25,
  "offset": 0
}
```

`rows` （一致する合計数）を`limit`/`offset`と一緒に使用して、結果をページ化します。

### `GET /public/v1/traces/:traceId`

1つのトレースの完全なスパン ウォーターフォールを返します。

| Param | 必須 | 説明 |
| ----------- | -------- | ---------------------------------------- |
| `tenant_id` | はい | トレースが属するテナント |
| `limit` | いいえ | 返す最大スパン数（1 ～ 500、デフォルトは500） |
| `offset` | いいえ | 非常に大きなトレースのページネーションのオフセット |

```bash
curl -s "{{API_BASE_URL}}/public/v1/traces/4bf92f3577b34da6a3ce929d0e0e4736?tenant_id=<tenant_id>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "spans": [
    {
      "SpanId": "00f067aa0ba902b7",
      "Name": "POST /checkout",
      "DurationMs": 812,
      "children": []
    }
  ],
  "totalDurationMs": 812,
  "spanCount": 14,
  "limit": 500,
  "offset": 0
}
```

### `GET /public/v1/metrics`

テナントの生の指標データポイントを返します。

| Param | 必須 | 説明 |
| ---------------------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `tenant_id` | はい | クエリするテナント |
| `metric` | `metric`/`like`のいずれか | 正確な指標名 |
| `like` | `metric`/`like`のいずれか | 複数のメトリック名に一致するSQL `LIKE` パターン |
| `type` | いいえ | `gauge` （既定値）または`sum` |
| `from`, `to` | いいえ | 期間。デフォルトは過去24時間です。 |
| `service` | いいえ | サービス名でフィルタリング |
| `host` | いいえ | ホスト名でフィルタリングします。 以下のインフラストラクチャホスト指標に必要です。この指標を使用しない場合、テナント内のすべてのホストからの読み取りは混在します |
| `attribute_key`, `attribute_value` | いいえ | 特定の指標属性でフィルタリングします（一緒に使用する必要があります） |

```bash
curl -s "{{API_BASE_URL}}/public/v1/metrics?tenant_id=<tenant_id>&metric=jvm.memory.used&type=gauge" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "data": [
    {
      "TimeUnix": "2026-08-30T09:00:00Z",
      "MetricName": "jvm.memory.used",
      "Value": 512482816,
      "ServiceName": "checkout-api",
      "host": ""
    }
  ],
  "rows": 1
}
```

#### インフラホスト指標

同じエンドポイントは、インフラストラクチャ ダッシュボード（CPU、メモリ、ロード平均、ディスク I/O、ネットワーク I/O）に表示されるホストレベルの指標も提供します。 `metric` / `attribute_key` / `attribute_value`の正確な組み合わせを使用します。常に`host`を使用します。

| ダッシュボードウィジェット | `metric` | `attribute_key` | `attribute_value` |
| --------------------- | ------------------------------------- | --------------- | ------------------------------------------------------------------------------------------- |
| CPU % | `system.cpu.utilization` | `state` | `idle` （「使用中」の場合は1から減算）、または個別に`user`/`system`/`iowait`をクエリして合計 |
| メモリ使用率% | `system.memory.utilization` | `state` | `used` |
| 負荷平均（1m） | `system.cpu.load_average.1m` | — | — |
| ディスク読み取りI/O | `system.disk.io` (`type=sum`) | `direction` | `read` |
| ディスク書き込みI/O | `system.disk.io` (`type=sum`) | `direction` | `write` |
| ディスク読み取り操作 | `system.disk.operations` (`type=sum`) | `direction` | `read` |
| ディスク書き込み操作 | `system.disk.operations` (`type=sum`) | `direction` | `write` |
| のネットワーク | `system.network.io` (`type=sum`) | `direction` | `receive` |
| ネットワークアウト | `system.network.io` (`type=sum`) | `direction` | `transmit` |

```bash
curl -s "{{API_BASE_URL}}/public/v1/metrics?tenant_id=<tenant_id>&metric=system.cpu.utilization&type=gauge&attribute_key=state&attribute_value=idle&host=<host_name>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

**重要 – ディスクとネットワークの値は、レートではなく、常に増加する未加工のカウンターです。** ダッシュボードの「bytes/sec」チャートと「operations/sec」チャートは、2つの連続したカウンター読み取りを取得し、経過時間で割って計算されます。

```
rate = (value_at_t2 - value_at_t1) / (t2 - t1_in_seconds)
```

### `GET /public/v1/pages`

Dispatcher インスタンスごとに、リクエスト数でランク付けされた上位のリクエスト済みコンテンツページ （`.html`）。 `dispatcher.httpd.requests`指標に基づくサポート – このエンドポイントは、一般的なページ分析ツールではなく、AEM Dispatcher/CDN スタイルのアクセスログに固有です。

| Param | 必須 | 説明 |
| ------------ | -------- | -------------------------------------- |
| `tenant_id` | はい | クエリするテナント |
| `from`, `to` | いいえ | 期間。デフォルトは過去24時間です。 |
| `limit` | いいえ | 返す最大行数（1 ～ 500、デフォルトは50） |

```bash
curl -s "{{API_BASE_URL}}/public/v1/pages?tenant_id=<tenant_id>&limit=50" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "from": 1735689000,
  "to": 1735690800,
  "data": [
    {
      "instance": "<instance_name>",
      "domain": "www.abc.com",
      "path": "/join-us/insights.html",
      "full_url": "https://www.abc.com/join-us/insights.html",
      "requests": 7
    }
  ],
  "rows": 1
}
```

---

## &#x200B;5. このAPIが行わないこと

- **生のSQL アクセスはありません。** あらゆるエンドポイントは、収集された目的に合わせて構築されたデータシェイプを返します。基盤となるデータストアを直接クエリすることはできません。
- **クロステナントクエリはありません。** すべてのリクエストは、正確に1つの`tenant_id`にスコープが設定されています。
- **書き込みアクセス権がありません。** パブリック APIは読み取り専用です。

---

## &#x200B;6. サポート

予期しないエラーが発生した場合、またはこれらのエンドポイントでカバーされていないユースケースがある場合は、カスタマーサクセス/イネーブルメントエンジニアにお問い合わせください。
