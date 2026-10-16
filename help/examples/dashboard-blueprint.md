---
title: ダッシュボードのブループリント
description: アプリケーションとインフラストラクチャのモニタリングにブループリントを使用して、オブザーバビリティインサイトダッシュボードを構築します。
source-git-commit: 94e108d029b04bf4f04aec3fb8b949705c4005e1
workflow-type: tm+mt
source-wordcount: '1196'
ht-degree: 1%
---

## ダッシュボードのブループリント {#dashboard-blueprints}

各ブループリントは、包括的で複雑なダッシュボードです。 目標、タブとセクションのレイアウト、およびグラフの種類、単位、しきい値、サイズとクエリを含むすべてのウィジェットが一覧表示されます。 サイズは12列グリッドの`width×height`です。 それぞれを上から下に構築し、ドラッグして調整します。

**グラフ凡例：** **N** =番号、**L** =行、**A** =面積、**B** =棒、**P** =円、**T** =表。



### ゴールデンシグナル/SLO ボード

**目標：** 「現在、プラットフォームは正常ですか？」という画面1つ — トラフィック、エラー、遅延、飽和度。
**時間選択：**&#x200B;最後の1時間（インシデント）または24時間（毎日のレビュー）。

```text
┌──────────── Section: SLO KPIs ────────────────────────────────┐
│ N Requests │ N Error % │ N p95 │ N Apdex │ N Slow % │ N Hosts  │  (6 × 2w×2h)
├──────────── Section: Traffic & Errors ────────────────────────┤
│ A Throughput by service (6w) │ L Error % by service (6w)       │
├──────────── Section: Latency ─────────────────────────────────┤
│ L p50/p95/p99 (6w)           │ L Apdex by service (6w)         │
├──────────── Section: Saturation ──────────────────────────────┤
│ L CPU by host (4w) │ L Memory by host (4w) │ L Load 5m (4w)    │
└───────────────────────────────────────────────────────────────┘
```

**W1 ・ リクエスト** — N、2×2

```sql
SELECT count(*) AS requests FROM Transaction
```

**W2 ・ エラー%** — N、単位`%`、2×2。 しきい値：`0 → Green`、`1 → Amber`、`5 → Red`。

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**W3 ・ p95待ち時間** — N、ユニット `ms`、2×2。 しきい値：`0 → Green`、`800 → Amber`、`2000 → Red`。

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**W4 ・ Apdex** — N、2×2。 しきい値：`0 → Red`、`0.7 → Amber`、`0.85 → Green` （最も高い一致するバンドが勝ったため、低いスコアは赤のままです）。

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**W5 ・ リクエスト > 2 s** — N、単位`%`、2×2。 しきい値：`0 → Green`、`2 → Amber`、`10 → Red`。

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction
```

**W6 ・ アクティブ ホスト** — N、2×2

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction
```

**W7 ・ サービス別スループット** — A、6×4

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W8 ・ サービス別エラー%** — L、ユニット `%`、6×4

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W9 ・ レイテンシーパーセンタイル** — L、ユニット `ms`、6×4

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**W10 ・ Apdex by service** — L、6×4

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W11 ・ ホスト別CPU** — L、4×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO LIMIT 10
```

**W12 ・ ホスト別メモリ** — L、4×4

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO LIMIT 10
```

**W13 ・ Load 5m** — L, 4×4

```sql
SELECT average(value) AS load5m FROM LoadAverage5m FACET host TIMESERIES AUTO LIMIT 10
```



### エラーのトリアージ ボード

3回のクリックで、「エラー率が最大」から「このエンドポイント、この例外、このトレース」までの&#x200B;**目標：**。

```text
Tab "Overview"
│ N Errors │ N Error % │ N Affected traces │ N 5xx count │                 (4 × 3w×2h)
│ L Errors by service (8w) │ P Status code mix (4w)                       │
│ T Errors vs total by service (12w)                                       │
Tab "Drill-down"
│ T 5xx hot spots: service × path × code (12w)                             │
│ T Exception types (6w) │ T Downstream failures (6w)                      │
│ T Latest failing requests – event list (12w×6h)                          │
```

**概要タブ**

**W1 ・ エラー** — N、しきい値`0 → Green`、`1 → Amber`、`100 → Red`

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error'
```

**W2 ・ エラー%** — N、単位`%`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**W3 ・影響を受けるトレース** — N

```sql
SELECT uniqueCount(traceId) AS traces FROM Span WHERE status = 'Error'
```

**W4 ・ HTTP 5xx** — N、しきい値`0 → Green`、`1 → Red`

```sql
SELECT count(*) AS http5xx FROM Transaction WHERE http.statusCode >= 500
```

**W5 ・ サービス別エラー** — L、8×4

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error' FACET service TIMESERIES AUTO LIMIT 10
```

**W6 ・ ステータスコードミックス** — P、4×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode >= 400 FACET http.statusCode LIMIT 10
```

**W7 ・ サービス別の合計に対するエラー** — T、12×4

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentage(*, WHERE status = 'Error') AS errorPct, count(*) FILTER (WHERE http.statusCode >= 500) AS http5xx, count(*) FILTER (WHERE http.statusCode >= 400 AND http.statusCode < 500) AS http4xx FROM Transaction FACET service ORDER BY errors DESC LIMIT 30
```

**ドリルダウン タブ**

**W8 ・ 5xx ホットスポット** — T、12×5

```sql
SELECT count(*) AS errors, percentile(duration, 95) AS p95, latest(traceId) AS lastTraceId FROM Transaction WHERE http.statusCode >= 500 FACET service, urlPath, http.statusCode ORDER BY errors DESC LIMIT 100
```

**W9 ・例外タイプ** — T、6×5

```sql
SELECT count(*) AS errors, latest(statusMessage) AS lastMessage FROM Span WHERE status = 'Error' AND error.type != '' FACET service, error.type ORDER BY errors DESC LIMIT 50
```

**W10 ・ ダウンストリーム エラー** — T、6×5

```sql
SELECT count(*) AS failures, percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND status = 'Error' FACET service, peer.service, externalDomain ORDER BY failures DESC LIMIT 50
```

**W11 ・最新の失敗したリクエスト** — T、12×6

```sql
SELECT timestamp, service, host, name, urlPath, http.statusCode, duration, statusMessage, traceId FROM Transaction WHERE status = 'Error' LIMIT 200
```



### レイテンシの詳細ボード

**目標：**&#x200B;は、_どこで_&#x200B;時間が経過するか、どの操作、どのセグメント、どの程度の一貫性があるかを見つけます。

```text
│ L p50/p95/p99 (8w)               │ N p99 (4w)                │
│ B Time share by operation (6w)   │ B Slowest ops by p95 (6w) │
│ T Latency profile per operation (12w)                         │
│ T Latency buckets per service (6w) │ B Variability (6w)       │
│ T Transaction composition – SpanBreakdown (12w)               │
│ T Slowest trace per operation (12w)                           │
```

**W1 ・ パーセンタイル** — L、`ms`、8×4

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**W2 ・ p99** — N、`ms`、しきい値`0 → Green`、`1500 → Amber`、`4000 → Red`、4×4

```sql
SELECT percentile(duration, 99) AS p99 FROM Transaction
```

**W3 ・ オペレーション別タイムシェア** — B、`%`、6×4

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Transaction FACET name LIMIT 15
```

**W4 ・最も遅い操作（p95）** — B、`ms`、6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 15
```

**W5 ・ レイテンシープロファイル** — T、12×5

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99, max(duration) AS maxMs FROM Transaction FACET service, name ORDER BY p95 DESC LIMIT 100
```

**W6 ・ レイテンシーバケット** — T、6×4

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS fast, count(*) FILTER (WHERE duration > 100 AND duration <= 500) AS ok, count(*) FILTER (WHERE duration > 500 AND duration <= 2000) AS slow, count(*) FILTER (WHERE duration > 2000) AS verySlow FROM Transaction FACET service LIMIT 20
```

**W7 ・ レイテンシの変動** — B、`ms`、6×4

```sql
SELECT stddev(duration) AS stddevMs FROM Transaction FACET service ORDER BY stddevMs DESC LIMIT 15
```

**W8 ・ トランザクション構成** — T、単位`ms`、12×5

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown FACET service, rootOperation, segment ORDER BY calls DESC LIMIT 100
```

**W9 ・ オペレーションあたりの最小トレース** — T、12×5

```sql
SELECT count(*) AS calls, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTraceId, latest(host, duration) AS slowestHost FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 50
```



### 下流の依存関係ボード

**目標：**&#x200B;は私たちですか、それとも彼らですか？ 呼び出されるあらゆるデータベース、API、キュー、RPCの健全性。

```text
│ N Outbound calls │ N Outbound error % │ N Outbound p95 │            (3 × 4w×2h)
│ T Downstream services (12w)                                       │
│ L External domain p95 (6w) │ T External domains (6w)              │
│ B DB p95 by system/op (6w) │ T Slow DB namespaces (6w)            │
│ L Messages per destination (6w) │ T Producer vs consumer (6w)     │
│ T RPC methods (12w)                                               │
```

**W1 ・ アウトバウンド呼び出し** — N

```sql
SELECT count(*) AS calls FROM Span WHERE kind = 'Client'
```

**W2 ・ アウトバウンドエラー%** — N、`%`、しきい値`0 → Green`、`2 → Amber`、`10 → Red`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client'
```

**W3 ・ アウトバウンド p95** — N、`ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client'
```

**W4 ・ ダウンストリーム サービス** — T、12×5

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client' FACET service, peer.service ORDER BY calls DESC LIMIT 100
```

**W5 ・外部ドメイン p95** — L、`ms`、6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain TIMESERIES AUTO LIMIT 8
```

**W6 ・外部ドメイン** — T、6×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain ORDER BY calls DESC LIMIT 50
```

**W7 ・ DB p95** — B、`ms`、6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE db.system != '' FACET db.system, db.operation ORDER BY p95 DESC LIMIT 20
```

**W8 ・低速DB名前空間** — T、6×4

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 99) AS p99, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE db.system != '' FACET service, db.system, db.namespace ORDER BY p99 DESC LIMIT 50
```

**W9 ・ Messages per destination** — L、6×4

```sql
SELECT count(*) AS messages FROM Span WHERE messaging.system != '' FACET messaging.destination TIMESERIES AUTO LIMIT 10
```

**W10 ・ プロデューサー対消費者** — T、6×4

```sql
SELECT count(*) FILTER (WHERE kind = 'Producer') AS produced, count(*) FILTER (WHERE kind = 'Consumer') AS consumed, percentile(duration, 95) AS p95 FROM Span WHERE messaging.system != '' FACET messaging.system, messaging.destination LIMIT 50
```

**W11 ・ RPC メソッド** — T、12×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE rpc.system != '' FACET service, rpc.system, rpc.method ORDER BY calls DESC LIMIT 50
```



### AEM オーサー/パブリッシュ/Dispatcher ボード

**目標：**は、従来のAEM AEM レイアウトを反映して、階層別にグループ化されたNew Relic操作ボードです。
**セクションヘッダー** （`PUBLISH`、`AUTHOR`、`DISPATCHER`）、終了ウィジェットのタイトル（`- PUBLISH`/`- AUTHOR`）を使用して、カラーキューを取得します。

```text
Tab "Tiers"
├─ Section: OVERVIEW
│ A Traffic by tier (8w) │ T Content mix by tier (4w)
├─ Section: PUBLISH
│ N Requests - PUBLISH │ N Error % - PUBLISH │ N p95 - PUBLISH │ N Apdex - PUBLISH
│ L p95 by host - PUBLISH (6w) │ L CPU - PUBLISH (6w)
│ L Heap - PUBLISH (6w) │ T Slowest /bin servlets - PUBLISH (6w)
├─ Section: AUTHOR
│ N Requests - AUTHOR │ N Error % - AUTHOR │ N p95 - AUTHOR │ N Active authors' hosts - AUTHOR
│ L p95 by host - AUTHOR (6w) │ L Heap - AUTHOR (6w)
├─ Section: DISPATCHER
│ L Dispatcher req/s (8w) │ L CPU - DISPATCHER (4w)
Tab "Code hot spots"
│ T Slow Sling scripts (6w) │ T Slow JSPs (6w)
│ T Hottest JCR paths (6w)  │ B JCR operation mix (6w)
│ T Sling request URIs (6w) │ T GraphQL operations (6w)
│ B Sling servlet time share (12w)
```

**概要**

**W1 ・ レベル別トラフィック** — A、8×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix TIMESERIES AUTO LIMIT 10
```

**W2 ・ レベル別コンテンツミックス** — T、4×4

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/content/dam/%') AS assets FROM Transaction FACET hostTierPrefix LIMIT 20
```

**セクションを公開**

**W3 ・ リクエスト – パブリッシュ** — N

```sql
SELECT count(*) AS requests FROM Transaction WHERE host LIKE '%publish%'
```

**W4 ・ エラー% - PUBLISH** — N、`%`、しきい値`0 → Green`、`1 → Amber`、`5 → Red`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction WHERE host LIKE '%publish%'
```

**W5 ・ p95 - PUBLISH** — N、`ms`、しきい値`0 → Green`、`1000 → Amber`、`3000 → Red`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%'
```

**W6 ・ Apdex - PUBLISH** — N、しきい値`0 → Red`、`0.7 → Amber`、`0.85 → Green`

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction WHERE host LIKE '%publish%'
```

**W7 ・ p95 by host - PUBLISH** — L, `ms`, 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO LIMIT 10
```

**W8 ・ CPU - PUBLISH** — L、`%`、6×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO
```

**W9 ・ Heap - PUBLISH** — L、`MB`、6×4

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO
```

**W10 ・ Slowest /bin サーブレット - PUBLISH** — T、6×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE host LIKE '%publish%' AND urlPath LIKE '/bin/%' FACET urlPath ORDER BY p95 DESC LIMIT 30
```

**作成者セクション** — W3-W9を複製し、`'%publish%'`を`'%author%'`に、タイトルのサフィックスを`- AUTHOR`に変更します。 追加ウィジェット：

**W11 ・ オーサーホストがアクティブ – オーサー** — N

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction WHERE host LIKE '%author%'
```

**Dispatcher セクション**

**W12 ・ Dispatcher req/s** — L、`req/s`、8×4

```sql
SELECT rate(value) AS requestsPerSecond FROM DispatcherRequestRate FACET host TIMESERIES AUTO
```

**W13 ・ CPU - DISPATCHER** — L、`%`、4×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization WHERE host LIKE '%dispatcher%' FACET host TIMESERIES AUTO
```

**ホットスポット タブのコード**

**W14 ・遅いSling スクリプト** — T

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE sling.script != '' FACET sling.script ORDER BY totalMs DESC LIMIT 50
```

**W15 ・ Slow JSP** — T

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95 FROM Span WHERE jsp.path != '' FACET jsp.path ORDER BY p95 DESC LIMIT 50
```

**W16 ・ Hottest JCR paths** — T

```sql
SELECT count(*) AS operations, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE jcr.path != '' FACET jcr.operation, jcr.path ORDER BY totalMs DESC LIMIT 50
```

**W17 ・ JCR操作ミックス** — B

```sql
SELECT count(*) AS operations FROM Span WHERE jcr.operation != '' FACET jcr.operation LIMIT 20
```

**W18 ・ Sling リクエスト URI** — T

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction WHERE sling.requestUri != '' FACET sling.requestUri ORDER BY requests DESC LIMIT 100
```

**W19 ・ GraphQL操作** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE graphql.operation != '' FACET graphql.operation ORDER BY calls DESC LIMIT 50
```

**W20 ・ Sling サーブレットの時間共有** — B、`%`、12×4

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Span WHERE sling.servlet != '' FACET sling.servlet LIMIT 20
```



### インフラ能力局

**目標：** キャパシティプランニングと飽和ハンティングをホスト間で実行します。
**時間ピッカー：** トレンドの7 d （ロールアップによってこれが高速になります）、インシデントの1 h。

```text
Tab "Compute"
│ N Hosts reporting │ N Peak CPU % │ N Peak memory % │ N Max load 5m │
│ L CPU by host (6w) │ L Memory by host (6w)                         │
│ L CPU user / system / iowait / steal (4 × 3w)                     │
│ T Host snapshot (12w)                                              │
│ T Top processes by CPU (6w) │ T Top processes by memory (6w)       │
Tab "Storage & Network"
│ T Disk used per mount (6w) │ N Lowest free root disk (6w)          │
│ L Disk read (6w) │ L Disk write (6w)                               │
│ L Network rx (6w) │ L Network tx (6w)                              │
│ L Total network I/O for one host – formula (12w)                   │
```

**W1 ・ ホストのレポート** — N

```sql
SELECT uniqueCount(host) AS hosts FROM CpuUtilization
```

**W2 ・ ピーク CPU** — N、`%`、しきい値`0 → Green`、`75 → Amber`、`90 → Red`

```sql
SELECT max(value) AS peakCpu FROM CpuUtilization
```

**W3 ・ ピークメモリ** — N、`%`、しきい値`0 → Green`、`80 → Amber`、`92 → Red`

```sql
SELECT max(value) AS peakMemory FROM MemoryUtilization
```

**W4 ・最大負荷5m** — N

```sql
SELECT max(value) AS maxLoad FROM LoadAverage5m
```

**W5 ・ ホスト別CPU** — L、`%`、6×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

**W6 ・ ホスト別メモリ** — L、`%`、6×4

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO
```

**W7-W10 ・ モード別CPU** — L、`%`、各3×4

```sql
SELECT average(value) AS userPct FROM CpuUserPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS systemPct FROM CpuSystemPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS iowaitPct FROM CpuIowaitPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS stealPct FROM CpuStealPercent FACET host TIMESERIES AUTO LIMIT 10
```

**W11 ・ ホスト スナップショット** — T、12×5

```sql
SELECT average(value) AS avgCpu, percentile(value, 95) AS p95Cpu, max(value) AS peakCpu, latest(value) AS nowCpu FROM CpuUtilization FACET host ORDER BY p95Cpu DESC LIMIT 100
```

**W12 ・ CPUによる上位プロセス** — T、6×5

```sql
SELECT average(value) AS cpuPct, max(value) AS peakPct FROM ProcessCpuUtilization FACET host, process.name ORDER BY cpuPct DESC LIMIT 30
```

**W13 ・ メモリ別の上位プロセス** — T、`MB`、6×5

```sql
SELECT average(value) AS memMb, max(value) AS peakMb FROM ProcessMemoryUsage FACET host, process.name ORDER BY memMb DESC LIMIT 30
```

**ストレージとネットワーク タブ**

**W14 ・ マウントごとに使用されるディスク** — T、`%`

```sql
SELECT latest(value) AS usedPct, max(value) AS peakPct FROM DiskUtilization FACET host, mountpoint ORDER BY usedPct DESC LIMIT 50
```

**W15 ・ Lowest free root disk** — N、unit `bytes`、しきい値`0 → Red`、`5000000000 → Amber`、`20000000000 → Green`

```sql
SELECT min(value) AS freeBytes FROM DiskFreeBytes
```

**W16 / W17 ・ ディスク スループット** — L、`bytes/s`

```sql
SELECT rate(value) AS readBytesPerSec FROM DiskReadRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host TIMESERIES AUTO
```

**W18 / W19 ・ ネットワーク スループット** — L、`bytes/s`

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS txBytesPerSec FROM NetworkTransmitRate FACET host TIMESERIES AUTO
```

**W20 ・ 1つのホスト （数式）**&#x200B;のネットワーク I/Oの合計 – フォーミュラ ウィジェット、L、`bytes/s`、12×4。 数式は`FACET`できず、`rate()`は1つのホストにスコープを設定する必要があります（[ キュレートされた指標エンティティ ](../references/data-dictionary.md#curated-metric-entities)を参照）。そのため、ホストを固定します – `web-publish-01`をW5の実際のホスト名に置き換えます。

クエリ `netRx`:

```sql
SELECT rate(value) AS v FROM NetworkReceiveRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

クエリ `netTx`:

```sql
SELECT rate(value) AS v FROM NetworkTransmitRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

計算式：

```text
netRx + netTx
```



### JVM ヘルスボード

**目標：**&#x200B;は、ヒープの負荷、スレッドのリーク、およびJVM CPUが停止する前に急増する問題に対処します。

```text
│ N Heap used % (formula) │ N Peak threads │ N JVM CPU % │ N Max heap MB │
│ L Heap used % over time – formula (12w)                                │
│ L Heap used by host (6w) │ T Heap limit by host (6w)                    │
│ L Threads by host (6w)   │ L JVM CPU by host (6w)                       │
│ T Memory pools (12w)                                                   │
```

**W1 ・ ヒープ使用率% （式）** – 数式、N、`%`、しきい値`0 → Green`、`75 → Amber`、`90 → Red`

クエリ `used`:

```sql
SELECT average(value) AS v FROM JvmHeapUsed
```

クエリ `limit`:

```sql
SELECT average(value) AS v FROM JvmHeapLimit
```

計算式：

```text
(used / limit) * 100
```

**W2 ・ ピークスレッド** — N

```sql
SELECT max(value) AS peakThreads FROM JvmThreadCount
```

**W3 ・ JVM CPU** — N、`%`

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization
```

**W4 ・最大ヒープ使用済み** — N、`MB`

```sql
SELECT max(value) AS maxHeapMb FROM JvmHeapUsed
```

**W5 ・ ヒープ使用率%時間（数式）** – 数式、L、`%`、12×4

クエリ `used`:

```sql
SELECT average(value) AS v FROM JvmHeapUsed TIMESERIES AUTO
```

クエリ `limit`:

```sql
SELECT average(value) AS v FROM JvmHeapLimit TIMESERIES AUTO
```

計算式：

```text
(used / limit) * 100
```

>[!NOTE]
>
> 1つのホストに数式をスコープするには、`WHERE host = 'my-publish-01'`を&#x200B;**両方**&#x200B;のクエリに追加します。

**W6 ・ ホストで使用されるヒープ** — L、`MB`、6×4

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed FACET host TIMESERIES AUTO
```

**W7 ・ ホストごとのヒープ制限** — T、`MB`、6×4

```sql
SELECT latest(value) AS heapLimitMb FROM JvmHeapLimit FACET host LIMIT 50
```

**W8 ・ ホスト別Threads** — L、6×4

```sql
SELECT latest(value) AS threads FROM JvmThreadCount FACET host TIMESERIES AUTO
```

**W9 ・ ホスト別JVM CPU** — L、`%`、6×4

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization FACET host TIMESERIES AUTO
```

**W10 ・ メモリ プール** — T、12×5

```sql
SELECT latest(value) AS bytesUsed, max(value) AS peakBytes FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool ORDER BY peakBytes DESC LIMIT 100
```



### 11.8 エグゼクティブ / NOC ウォールボード

**目標：**&#x200B;は、見やすく色分けされた壁面スクリーンです。 ほとんどの場合、しきい値を持つ数値ウィジェット、行ごとに1つのトレンド。

```text
│ N Requests │ N Availability % (formula) │ N Apdex │ N p95 │ N 5xx │ N Hosts │
│ L Throughput (12w)                                                          │
│ B Requests by service (6w) │ P Traffic by tier (6w)                         │
│ Text: "SLO: 99.9 % availability, p95 < 800 ms. Owner: platform-sre" (12w×1h) │
```

**W1 ・ リクエスト** — N

```sql
SELECT count(*) AS requests FROM Transaction
```

**W2 ・ Availability % （formula）** – 数式、N、`%`、しきい値`0 → Red`、`99 → Amber`、`99.9 → Green`

クエリ `ok`:

```sql
SELECT count(*) AS v FROM Transaction WHERE status != 'Error'
```

クエリ `total`:

```sql
SELECT count(*) AS v FROM Transaction
```

計算式：

```text
(ok / total) * 100
```

**W3 ・ Apdex** — N、しきい値`0 → Red`、`0.7 → Amber`、`0.85 → Green`

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**W4 ・ p95** — N、`ms`、しきい値`0 → Green`、`800 → Amber`、`2000 → Red`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**W5 ・ 5xx** — N、しきい値`0 → Green`、`1 → Amber`、`50 → Red`

```sql
SELECT count(*) AS http5xx FROM Transaction WHERE http.statusCode >= 500
```

**W6 ・ ホスト** — N

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction
```

**W7 ・ スループット** — L、12×4

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

**W8 ・ サービス別リクエスト** — B、6×4

```sql
SELECT count(*) AS requests FROM Transaction FACET service LIMIT 10
```

**W9 ・ レベル別トラフィック** — P、6×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix LIMIT 10
```

**W10 ・ SLO note** — テキストウィジェット、12×1。



### Ratios &amp; Headroom board （フォーミュラ・ショーケース）

**目標：**&#x200B;は、複数のクエリを必要とする派生KPIです。

**W1 ・ ホストあたりのリクエスト** – 数式、L、`req/host`

クエリ `requests`:

```sql
SELECT count(*) AS v FROM Transaction TIMESERIES AUTO
```

クエリ `hosts`:

```sql
SELECT uniqueCount(host) AS v FROM Transaction TIMESERIES AUTO
```

計算式：

```text
requests / hosts
```

**W2 ・ 5xxすべてのエラーの共有** – 数式、N、`%`

クエリ `e5xx`:

```sql
SELECT count(*) AS v FROM Transaction WHERE http.statusCode >= 500
```

クエリ `errors`:

```sql
SELECT count(*) AS v FROM Transaction WHERE http.statusCode >= 400
```

計算式：

```text
(e5xx / errors) * 100
```

**W3 ・ データベース内の時間と合計サーバー時間** – 数式、L、`%`

クエリ `db`:

```sql
SELECT sum(duration) AS v FROM Span WHERE db.system != '' TIMESERIES AUTO
```

クエリ `server`:

```sql
SELECT sum(duration) AS v FROM Transaction TIMESERIES AUTO
```

計算式：

```text
(db / server) * 100
```

**W4 ・ ヒープ ヘッドルーム （MB）** – 数式、L、`MB`

クエリ `limit`:

```sql
SELECT average(value) AS v FROM JvmHeapLimit TIMESERIES AUTO
```

クエリ `used`:

```sql
SELECT average(value) AS v FROM JvmHeapUsed TIMESERIES AUTO
```

計算式：

```text
limit - used
```

**W5 ・ 1つのホストに対するネットワークのアウト/イン比率** – 数式、L （ホストを固定します。[ キュレートされた指標エンティティ ](../references/data-dictionary.md#curated-metric-entities)を参照）

クエリ `rx`:

```sql
SELECT rate(value) AS v FROM NetworkReceiveRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

クエリ `tx`:

```sql
SELECT rate(value) AS v FROM NetworkTransmitRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

計算式：

```text
tx / rx
```

**W6 ・ パブリッシュとオーサートラフィックの比率** – 数式、N

クエリ `publish`:

```sql
SELECT count(*) AS v FROM Transaction WHERE host LIKE '%publish%'
```

クエリ `author`:

```sql
SELECT count(*) AS v FROM Transaction WHERE host LIKE '%author%'
```

計算式：

```text
publish / author
```

**W7 ・ Weighted Error Budget burn （SLO 99.9 %）** – 数式、L

クエリ `errors`:

```sql
SELECT count(*) AS v FROM Transaction WHERE status = 'Error' TIMESERIES AUTO
```

クエリ `total`:

```sql
SELECT count(*) AS v FROM Transaction TIMESERIES AUTO
```

計算式（消化率：1.0 = スケジュールどおりに予算を正確に使用）:

```text
(errors / total) / 0.001
```



### Service Explorer （変数駆動型）

**目標：**&#x200B;任意のチームが&#x200B;_自分の_&#x200B;のサービスと環境をポイントできる1つのボード。

>[!WARNING]
>
> [ チュートリアル 3、ステップ 8.7](#87-make-dashboards-reusable-with-variables)の&#x200B;**現在のビルド制限**&#x200B;を参照してください。ウィジェット SQL内の`{{variables}}`は、まだプレビューまたは保存できません。 これが修正されるまで、各`{{…}}`をリテラル （`'checkout'`など）に置き換え、_重複_&#x200B;を使用してサービスごとにボードを複製します。 以下のデザインは、エンドツーエンドで代用がサポートされている場合に使用できます。

**変数**

| 名前 | ラベル | 種類 | 複数 | 数値 | 定義 |
| -------- | ------------------- | ------------ | ----- | ------- | ---------------------------------------------------------- |
| `svc` | サービス | クエリから | ✔ |         | `SELECT count(*) FROM Transaction FACET service LIMIT 200` |
| `env` | 環境 | クエリから | ✔ |         | `SELECT count(*) FROM Span FACET environment LIMIT 50` |
| `slowMs` | 遅いしきい値（ms） | テキストボックス |       | ✔ | デフォルト `2000` |

```text
Section: "{{svc}} in {{env}}"
│ N Requests │ N Error % │ N p95 │ N Slow requests │
│ L Throughput by service (6w) │ L p95 by service (6w)   │
│ T Operations (12w)                                     │
│ T Slow requests over {{slowMs}} ms – events (12w)       │
│ T Downstream calls (12w)                               │
```

**W1 ・ リクエスト —{{svc}}** — N

```sql
SELECT count(*) AS requests FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W2 ・ エラー%** — N、`%`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W3 ・ p95** — N、`ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W4 ・要求が遅い** — N

```sql
SELECT count(*) AS slow FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) AND duration > {{slowMs}}
```

**W5 ・ サービス別スループット** — L

```sql
SELECT count(*) AS requests FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service TIMESERIES AUTO
```

**W6 ・ p95 by service** — L、`ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service TIMESERIES AUTO
```

**W7 ・ オペレーション** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service, name ORDER BY calls DESC LIMIT 100
```

**W8 ・低速なリクエスト（イベント）** — T

```sql
SELECT timestamp, service, host, name, urlPath, duration, traceId FROM Transaction WHERE service IN ({{svc}}) AND duration > {{slowMs}} ORDER BY duration DESC LIMIT 200
```

**W9 ・ ダウンストリーム呼び出し** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND service IN ({{svc}}) FACET peer.service, externalDomain, db.system ORDER BY calls DESC LIMIT 50
```
