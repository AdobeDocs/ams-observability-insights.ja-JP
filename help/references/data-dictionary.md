---
source-git-commit: 4949fcad799260aa73354a8c0b9d79e6394cb257
workflow-type: tm+mt
source-wordcount: '585'
ht-degree: 21%
---
## データディクショナリ {#data-dictionary}

クエリできるあらゆるエンティティと属性です。 ライブカタログには、説明が含まれています。また、エクスプローラーの&#x200B;**クエリガイド** パネルとウィジェットの追加ダイアログにも表示されます。

### トレースエンティティ

| エンティティ | 次を含む | 固定範囲 | ロールアップ |
| ------------- | --------------------------------------------------------- | ---------------------------- | ------------------------------------------ |
| `Transaction` | インバウンドサーバーリクエスト | `kind = Server` | 1m / 5m / 1h （[&#x200B; パフォーマンス &#x200B;](#performance)） |
| `AppRequest` | インバウンドサーバー&#x200B;**および**&#x200B;件のメッセージコンシューマー要求 | `kind IN (Server, Consumer)` | — |
| `Span` | あらゆるスパン（サーバー、クライアント、プロデューサー、消費者、社内） | — | — |

<details open>
<summary><strong> トレース属性</strong> （3つのエンティティすべてで同じ）</summary>

| 属性 | タイプ | 説明 |
| --------------------------------------------------- | ----------- | ----------------------------------------------------------------------------------------------- |
| `timestamp` | タイムスタンプ | スパン開始時間 |
| `traceId`, `spanId`, `parentSpanId` | 文字列 | 識別子（ルートスパンの値は`parentSpanId = ''`） |
| `name` | 文字列 | 操作またはトランザクション名 |
| `kind` | 文字列 | `Server`, `Client`, `Producer`, `Consumer`, `Internal` |
| `service` | 文字列 | 送信サービス名 |
| `serviceKey` | 文字列 | 効果的なサービス ID （`service`が空白または親が粗い場合は`appName`にフォールバックします） |
| `appName` | 文字列 | Observability Insights アプリケーション名 |
| `appInstance` | 文字列 | インスタンスごとのID：オーサー/パブリッシュホストプレフィックス、アプリ名、サービス |
| `hostTierPrefix` | 文字列 | ホスト名が`author` / `publish` / `dispatcher`の役割サフィックスで切り捨てられました |
| `duration` | 数値（ミリ秒） | スパン期間（ミリ秒単位） |
| `status` | 文字列 | `Ok`, `Error`, `Unset` |
| `statusMessage` | 文字列 | ステータスの説明 |
| `host` | 文字列 | ホスト名 |
| `environment` | 文字列 | デプロイ環境 |
| `region` | 文字列 | クラウド地域 |
| `cloud.provider` | 文字列 | AWS/Azure/GCP |
| `http.method` | 文字列 | `GET`、`POST`、... |
| `http.statusCode` | 数値 | HTTP レスポンスステータス |
| `http.route` | 文字列 | 正規化されたルート テンプレート |
| `url` | 文字列 | 完全URL |
| `urlPath` | 文字列 | URL （`/content/site/en.html`）のパス部分 |
| `externalDomain` | 文字列 | アウトバウンド URLのドメイン （クライアントのスパンで使用） |
| `peer.service` | 文字列 | というダウンストリームサービス |
| `db.system`, `db.operation`, `db.namespace` | 文字列 | データベース技術、操作およびスキーマ |
| `messaging.system`, `messaging.destination` | 文字列 | Kafka、RabbitMQ、...およびトピックまたはキュー |
| `rpc.system`, `rpc.method` | 文字列 | gRPCと類似 |
| `error.type` | 文字列 | 例外またはエラータイプ |
| `sling.script`, `sling.servlet`, `sling.requestUri` | 文字列 | AEM Slingの詳細 |
| `jsp.path` | 文字列 | AEM JSPが実行されました |
| `jcr.operation`, `jcr.path` | 文字列 | AEM JCR リポジトリの操作とパス |
| `graphql.operation` | 文字列 | GraphQL操作名 |

</details>

### 汎用指標エンティティ

| エンティティ | 次を含む |
| ------------- | -------------------------------------------------- |
| `MetricGauge` | あらゆる時点のOpenTelemetry ゲージサンプル |
| `MetricSum` | すべての累積または差分OpenTelemetry合計サンプル |

**検出**&#x200B;またはキュレートされたエンティティを持たない指標に使用します：`WHERE metric = 'jvm.gc.duration'`。

<details open>
<summary><strong>指標の属性</strong> （すべての指標エンティティで共有）</summary>

| 属性 | タイプ | 説明 |
| ----------------------------------------------------------------------- | --------- | ------------------------------------------------------- |
| `timestamp` | タイムスタンプ | サンプル時間 |
| `metric` | 文字列 | OpenTelemetry メトリック名 |
| `value` | 数値 | サンプル値（キュレーションされたエンティティで変換。以下を参照） |
| `unit` | 文字列 | 指標単位 |
| `service`, `appName`, `host`, `environment`, `region`, `cloud.provider` | 文字列 | リソース ID |
| `state` | 文字列 | CPU モードまたはメモリステート |
| `cpu`, `device`, `direction`, `mountpoint` | 文字列 | CPUごと、デバイスごと、I/O方向、ファイルシステム |
| `pool` | 文字列 | JVM メモリプールまたは接続プール |
| `topic` | 文字列 | カフカ語 |
| `process.pid`, `process.name`, `process.owner` | 文字列 | プロセス ID |

</details>

### キュレートされた指標エンティティ

キュレーションされた各エンティティは、1つのOpenTelemetry指標を適切なフィルターと単位の変換でラップするため、`value`は既に意味があります。 **カウンター** エンティティ （✔）には累積合計が含まれています。 `rate(value)`と`TIMESERIES`でクエリします。

| エンティティ | `value`の意味 | 単位 | カウンター | メモ |
| ------------------------------------------------------------------------------ | ----------------------------------- | --------- | ------- | -------------------------------------------- |
| `CpuUtilization` | CPUが使用中 | % |         | ホストによる`average(value)`の1分間のロールアップ |
| `CpuUserPercent` / `CpuSystemPercent` / `CpuIowaitPercent` / `CpuStealPercent` | CPU時間（モード別） | % |         |                                              |
| `MemoryUtilization` | 使用メモリ | % |         | `average(value)`の1分間のロールアップ |
| `LoadAverage1m` / `LoadAverage5m` / `LoadAverage15m` | 読み込み平均 | — |         |                                              |
| `LoadAverageWithFallback` | 5m負荷、1mにフォールバック | — |         |                                              |
| `DiskUtilization` | マウントごとに使用されるディスク | % |         | `mountpoint`のファセット |
| `DiskFreeBytes` | ルートファイルシステムを無料で | バイト |         |                                              |
| `DiskReadRate` / `DiskWriteRate` | ディスクスループット | バイト/秒 | ✔ |                                              |
| `DiskReadOpsRate` / `DiskWriteOpsRate` | ディスク IOPS | ops/s | ✔ |                                              |
| `NetworkReceiveRate` / `NetworkTransmitRate` | eth0 スループット | バイト/秒 | ✔ |                                              |
| `NetworkReceiveRateAllDevices` / `NetworkTransmitRateAllDevices` | すべてのデバイス | バイト/秒 | ✔ |                                              |
| `NetworkIoRate` | すべての方向とデバイスを組み合わせた | バイト/秒 | ✔ |                                              |
| `NetworkReceiveErrorRate` / `NetworkTransmitErrorRate` | ネットワークエラー | エラー/秒 | ✔ |                                              |
| `NetworkReceivePacketRate` / `NetworkTransmitPacketRate` | パケット | パケット/秒 | ✔ |                                              |
| `ProcessCpuUtilization` | プロセス単位のCPU | % |         | `process.name`のファセット |
| `ProcessMemoryUsage` | プロセスメモリ | MB |         |                                              |
| `JvmHeapUsed` / `JvmHeapLimit` | JVM ヒープ | MB |         | メモリプールごとに報告されました |
| `JvmThreadCount` | JVM スレッド | — |         |                                              |
| `JvmCpuRecentUtilization` | JVM プロセス CPU | % |         | 1分ロールアップ |
| `DispatcherRequestRate` | AEM Dispatcher HTTP リクエスト | req/s | ✔ |                                              |

&#x200B;> [!NOTE]
> JVM メモリ指標はメモリプールごとに&#x200B;**報告されるので、`JvmHeapUsed` / `JvmHeapLimit`の`average(value)`はプール間の平均を示します。**&#x200B;プールごとの正確なビューには`FACET host, pool`を使用します。

### `SpanBreakdown`：事前集計されたトランザクション構成

`SpanBreakdown`は、各トランザクションの1分あたりの子スパン構成を保持します。 取り込み時に計算されるので、クエリするのは安くなります。

| 属性 | タイプ | 説明 |
| --------------- | --------- | ---------------------------------------------------------------------- |
| `timestamp` | タイムスタンプ | 1分バケット |
| `service` | 文字列 | サービス |
| `rootOperation` | 文字列 | 親トランザクション名 |
| `segment` | 文字列 | 子スパン名 |
| `callCount` | 数値 | バケット内のこのセグメントのコール |
| `avgDurationMs` | 数値 | バケットの平均セグメント期間（単位を手動で`ms`に設定） |
