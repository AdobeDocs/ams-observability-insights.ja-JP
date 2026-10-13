---
title: Query cookbook
description: SQL クエリレシピを使用して、Observability Insightsのアプリケーションとインフラストラクチャのテレメトリを調査します。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '580'
ht-degree: 0%
---

## Query cookbook {#query-cookbook}

クックブックは、すぐに使用できるクエリのライブラリで、トピック別にグループ化されています。 各クエリの前には、**太字**&#x200B;で、最適なグラフの種類が表示されます。 タイムピッカーがウィンドウを制御するため、時間句は除外されます。

**このドキュメントでは、** [10.1 Data discovery](#101-data-discovery) ・ [10.2 Throughput](#102-throughput) ・ [10.3 Latency](#103-latency) ・ [10.4 Errors](#104-errors) ・ [10.5 Apdex](#105-apdex--user-experience) ・ [10.6 サービスおよびホスト ](#106-services-hosts--instances) ・ [10.7 Dependencies](#107-downstream-dependencies) ・ [10.8](#108-aem-adobe-experience-manager) ・ ・ {AEM [10.9 インフラストラクチャ ](#109-infrastructure) ・ [10.10 JVM](#1010-jvm--runtime)

>[!TIP]
>
> `'checkout'`や`'web-publish-01'`などのサンプル値を、テナントの実際の値に置き換えます。 以下のデータ検出クエリは、それらをリストしています。

### データの発見

**テーブル** — トラフィックを報告するサービス：

```sql
SELECT count(*) AS requests, uniqueCount(host) AS hosts FROM Transaction FACET service LIMIT 200
```

**テーブル** — ホスト/階層が存在します：

```sql
SELECT count(*) AS requests FROM Transaction FACET hostTierPrefix, host LIMIT 200
```

**表** – 存在する環境と地域：

```sql
SELECT count(*) AS spans FROM Span FACET environment, region LIMIT 50
```

**Table** – 最新の値を持つすべてのゲージ指標レポート：

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricGauge FACET metric, unit LIMIT 200
```

**テーブル** – すべての合計/カウンター指標：

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricSum FACET metric, unit LIMIT 200
```

**テーブル** — スパン属性が入力されます（AEMの例）。

```sql
SELECT count(*) FILTER (WHERE sling.script != '') AS slingScripts, count(*) FILTER (WHERE jcr.operation != '') AS jcrOps, count(*) FILTER (WHERE db.system != '') AS dbCalls, count(*) FILTER (WHERE graphql.operation != '') AS graphql FROM Span FACET service LIMIT 50
```

### スループット

**Number** – リクエストの合計数：

```sql
SELECT count(*) AS requests FROM Transaction
```

**行** — スループットの傾向：

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

**領域** — サービス別スループット、上位8:

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**Bar** — HTTP メソッドによるスループット：

```sql
SELECT count(*) AS requests FROM Transaction FACET http.method LIMIT 10
```

**テーブル** – 最もビジストなURL パス：

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET urlPath ORDER BY requests DESC LIMIT 50
```

**Number** – 個別のトレース：

```sql
SELECT uniqueCount(traceId) AS traces FROM Span
```

### 待ち時間

**数値** — p95の応答時間（単位の自動= ミリ秒）:

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**行** — 1つのグラフのp50 / p95 / p99:

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**行** — サービスあたりのp95、最もビジーな上位10:

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**Bar** — p95による最も遅い操作：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

**テーブル** – 操作ごとの待ち時間プロファイル：

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99, max(duration) AS maxMs, stddev(duration) AS stddevMs FROM Transaction FACET name ORDER BY p95 DESC LIMIT 50
```

**棒** – 操作あたりの合計滞在時間の割合（時間予算が行われる場所）:

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Transaction FACET name LIMIT 20
```

**テーブル** – 操作ごとに1つの最も遅い要求のトレース ID:

```sql
SELECT count(*) AS calls, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTraceId FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 50
```

**Number** — 2秒より遅いリクエストのシェア：

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowShare FROM Transaction
```

**テーブル** – フィルタリングされたカウントを使用した待ち時間バケット（ヒストグラムの代用）:

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS under100ms, count(*) FILTER (WHERE duration > 100 AND duration <= 500) AS to500ms, count(*) FILTER (WHERE duration > 500 AND duration <= 2000) AS to2s, count(*) FILTER (WHERE duration > 2000) AS over2s FROM Transaction FACET service LIMIT 20
```

### エラー

**数値** – エラー率%:

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**行** — サービスあたりのエラー率%:

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**行** — エラー数の傾向：

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error' TIMESERIES AUTO
```

**円** — HTTP ステータス クラスの組み合わせ：

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode IS NOT NULL FACET http.statusCode LIMIT 15
```

**表** — 5xx ホットスポット：

```sql
SELECT count(*) AS errors5xx FROM Transaction WHERE http.statusCode >= 500 FACET service, urlPath, http.statusCode LIMIT 50
```

**テーブル** – 例外タイプ別のエラー：

```sql
SELECT count(*) AS errors, uniqueCount(traceId) AS traces, latest(statusMessage) AS lastMessage FROM Span WHERE status = 'Error' FACET service, error.type LIMIT 50
```

**表** — エラーと合計値の比較：

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service ORDER BY errors DESC LIMIT 20
```

**表** – 最新の失敗した要求（イベントリスト）:

```sql
SELECT timestamp, service, name, urlPath, http.statusCode, duration, traceId FROM Transaction WHERE status = 'Error' LIMIT 100
```

### アプリとユーザーエクスペリエンス

**数値** —Apdex （T = 500 ms）:

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**行** — サービスごとのApdex:

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**Bar** — URL パスごとの最悪のApdex （より厳格な300 ミリ秒のターゲット）:

```sql
SELECT apdex(duration, 300) AS apdex, count(*) AS requests FROM Transaction WHERE urlPath LIKE '/content/%' FACET urlPath ORDER BY apdex ASC LIMIT 20
```

### サービス、ホスト、インスタンス

**テーブル** — サービススコアカード：

```sql
SELECT count(*) AS requests, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95, apdex(duration, 500) AS apdex, uniqueCount(host) AS hosts FROM Transaction FACET serviceKey ORDER BY requests DESC LIMIT 50
```

**表** — インスタンスごと（AppRequestにはコンシューマーが含まれます）:

```sql
SELECT count(*) AS requests, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95 FROM AppRequest FACET appInstance ORDER BY requests DESC LIMIT 50
```

**行** — ホストあたりのスループット：

```sql
SELECT count(*) AS requests FROM Transaction FACET host TIMESERIES AUTO LIMIT 10
```

**Bar** – 負荷分散チェック （ホストごとに共有するリクエスト）:

```sql
SELECT count(*) AS requests FROM Transaction FACET host LIMIT 30
```

### ダウンストリームの依存関係

**Table** – 次の名前のダウンストリームサービス：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client' AND peer.service != '' FACET service, peer.service LIMIT 50
```

**テーブル** – 呼び出される外部ドメイン：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain ORDER BY calls DESC LIMIT 50
```

**Bar** — システムと操作によるデータベースの遅延：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95 FROM Span WHERE db.system != '' FACET db.system, db.operation LIMIT 30
```

**テーブル** – 低速なデータベース名前空間：

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 99) AS p99 FROM Span WHERE db.system != '' FACET db.system, db.namespace ORDER BY p99 DESC LIMIT 30
```

**行** – 宛先ごとのメッセージスループット：

```sql
SELECT count(*) AS messages FROM Span WHERE messaging.system != '' FACET messaging.destination TIMESERIES AUTO LIMIT 10
```

**表** — トピックごとのプロデューサーと消費者の比較：

```sql
SELECT count(*) FILTER (WHERE kind = 'Producer') AS produced, count(*) FILTER (WHERE kind = 'Consumer') AS consumed, percentile(duration, 95) AS p95 FROM Span WHERE messaging.system != '' FACET messaging.system, messaging.destination LIMIT 30
```

**Table** — RPC メソッド：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE rpc.system != '' FACET rpc.system, rpc.method LIMIT 30
```

### AEM（Adobe Experience Manager）

**Area** — AEM層によるトラフィック （オーサー/パブリッシュ/ディスパッチャーのプレフィックス）:

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix TIMESERIES AUTO LIMIT 10
```

**行** – パブリッシュ層p95:

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO LIMIT 10
```

**行** – 作成者層p95:

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%author%' FACET host TIMESERIES AUTO LIMIT 10
```

**テーブル** — ページ要求（`/content`）とサーブレット呼び出し（`/bin`）の比較：

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/etc.clientlibs/%') AS clientlibs, count(*) FILTER (WHERE urlPath LIKE '/content/dam/%') AS assets FROM Transaction FACET hostTierPrefix LIMIT 20
```

**テーブル** – 最も遅い`/bin` サーブレット：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE urlPath LIKE '/bin/%' FACET urlPath ORDER BY p95 DESC LIMIT 30
```

**テーブル** — Sling リクエスト URI:

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction WHERE sling.requestUri != '' FACET sling.requestUri ORDER BY requests DESC LIMIT 100
```

**テーブル** – 遅いSling スクリプト：

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE sling.script != '' FACET sling.script ORDER BY totalMs DESC LIMIT 50
```

**Bar** – 時間共有別Sling サーブレット：

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Span WHERE sling.servlet != '' FACET sling.servlet LIMIT 20
```

**テーブル** – 低速なJSP:

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95 FROM Span WHERE jsp.path != '' FACET jsp.path ORDER BY p95 DESC LIMIT 50
```

**Bar** — JCR操作ミックス：

```sql
SELECT count(*) AS operations FROM Span WHERE jcr.operation != '' FACET jcr.operation LIMIT 20
```

**テーブル** – 最もホットなJCR パス：

```sql
SELECT count(*) AS operations, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE jcr.path != '' FACET jcr.operation, jcr.path ORDER BY totalMs DESC LIMIT 50
```

**Table** — GraphQL（ヘッドレス）操作：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE graphql.operation != '' FACET graphql.operation ORDER BY calls DESC LIMIT 50
```

**行** — ホストあたりのDispatcher リクエスト率：

```sql
SELECT rate(value) AS requestsPerSecond FROM DispatcherRequestRate FACET host TIMESERIES AUTO
```

**表** — トランザクションの構成要素（セグメントの内訳、単位`ms`）:

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown WHERE rootOperation LIKE 'GET /content%' FACET rootOperation, segment ORDER BY calls DESC LIMIT 50
```

### インフラストラクチャ

**行** — ホストごとのCPU:

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

**行** — CPU ユーザー時間：

```sql
SELECT average(value) AS userPct FROM CpuUserPercent TIMESERIES AUTO
```

>[!NOTE]
>
> SQLはクエリごとに1つのエンティティを読み取るので、user/system/iowait/stealには4つの並べてウィジェットが必要です（[blueprint 11.6](#116-infrastructure-capacity-board) W7-W10を参照）。 数式は、それらを4つの系列ではなく1つの数値に折りたたみます。

**Bar** — ホスト別CPU p95:

```sql
SELECT percentile(value, 95) AS cpuP95 FROM CpuUtilization FACET host LIMIT 30
```

**テーブル** — ホスト正常性スナップショット：

```sql
SELECT average(value) AS avgCpu, max(value) AS peakCpu, latest(value) AS nowCpu FROM CpuUtilization FACET host ORDER BY peakCpu DESC LIMIT 50
```

**行** — メモリ使用率%:

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO
```

**Line** — メモリ増加率（リークハンティング）:

```sql
SELECT derivative(value) AS memoryTrend FROM MemoryUtilization FACET host TIMESERIES AUTO
```

**テーブル** — マウントごとに使用されるディスク：

```sql
SELECT latest(value) AS usedPct FROM DiskUtilization FACET host, mountpoint ORDER BY usedPct DESC LIMIT 50
```

**Number** — ホスト間で最小の空きルート ディスク （バイト単位の値、単位`bytes`を設定）:

```sql
SELECT min(value) AS freeBytes FROM DiskFreeBytes
```

**行** — ディスク読み取りと書き込みのスループット （それぞれ1つ）:

```sql
SELECT rate(value) AS readBytesPerSec FROM DiskReadRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host TIMESERIES AUTO
```

**Line** — ネットワーク受信/送信：

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS txBytesPerSec FROM NetworkTransmitRate FACET host TIMESERIES AUTO
```

**Line** — ネットワーク エラー：

```sql
SELECT rate(value) AS rxErrors FROM NetworkReceiveErrorRate FACET host TIMESERIES AUTO
```

**行** – 読み込み平均：

```sql
SELECT average(value) AS load5m FROM LoadAverage5m FACET host TIMESERIES AUTO
```

**Table** — CPUとメモリによる上位プロセス：

```sql
SELECT average(value) AS cpuPct, max(value) AS peakCpuPct FROM ProcessCpuUtilization FACET host, process.name ORDER BY cpuPct DESC LIMIT 30
```

```sql
SELECT average(value) AS memMb FROM ProcessMemoryUsage FACET host, process.name ORDER BY memMb DESC LIMIT 30
```

**テーブル** – 名前による生のゲージ（キュレーションされたエンティティは必要ありません）:

```sql
SELECT average(value) AS avgValue, max(value) AS maxValue FROM MetricGauge WHERE metric = 'system.cpu.utilization' FACET host, state LIMIT 50
```

### JVMとランタイム

**行** — ホストごとに使用されるヒープ （MB）:

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed FACET host TIMESERIES AUTO
```

**表** – 使用されるヒープと制限（2つのウィジェットまたは数式 – [ ブループリント 11.7](#117-jvm-health-board)を参照）:

```sql
SELECT latest(value) AS heapLimitMb FROM JvmHeapLimit FACET host LIMIT 50
```

**行** — JVM スレッド：

```sql
SELECT latest(value) AS threads FROM JvmThreadCount FACET host TIMESERIES AUTO
```

**行** — JVM プロセス CPU:

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization FACET host TIMESERIES AUTO
```

**テーブル** — JVM メモリ プール （汎用合計メトリック）:

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool LIMIT 100
```
