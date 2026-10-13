---
source-git-commit: 263ba9e9b3eb1c6e4dd253ca03bd0ce8954c588b
workflow-type: tm+mt
source-wordcount: '491'
ht-degree: 1%
---
## チュートリアル 4：高度な関数 {#tutorial-4-advanced-functions}

このチュートリアルでは、より詳細な調査を実現する機能とエンティティについて説明します。変数、成長、「最悪の例」検索、合計のシェア、トランザクション構成、任意のOpenTelemetry指標です。

**このチュートリアルでは、** [9.1 stddev](#91-measure-consistency-with-stddev) ・ [9.2 derivative](#92-spot-growth-with-derivative) ・ [9.3が最新](#93-find-the-worst-example-with-ranked-latest) ・ [9.4 percentOfTotal](#94-find-where-time-goes-with-percentoftotal) ・ [9.5 SpanBreakdown](#95-break-down-transactions-with-spanbreakdown) ・ [9.6任意の指標](#96-discover-and-query-any-metric) ・ [9.7 エンティティを選択](#97-choose-the-right-trace-entity) ・ [9.8 Nested aggregation](#98-work-around-nested-aggregation)

### `stddev`との整合性を測定

2つの操作は平均を共有でき、それでも非常に異なる動作をします。 標準偏差は、レイテンシの変化を示します。

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, stddev(duration) AS stddevMs FROM Transaction FACET name ORDER BY stddevMs DESC LIMIT 20
```

平均よりも大きい標準偏差は、リクエストが非常に遅い場合がある操作を指します。 次にp99を見てみましょう。

### `derivative`で成長を見つける

`derivative(attr)`は、バケット間の属性の平均の1秒あたりの変化を返します。 この機能は、リークの前にヒープの使用状況など、上昇しているゲージを検出するのに便利です。

```sql
SELECT derivative(value) AS heapMBPerSec FROM JvmHeapUsed FACET host TIMESERIES 5 minutes
```

持続的なプラスの線は成長を意味する。 負の値は、ゲージが下がっていることを意味します（ガベージコレクションの後など）。 `rate`と同様に、`derivative`には`TIMESERIES`が必要で、集約は唯一である必要があります。

### ランク付けされた`latest`で最悪の例を検索

`latest`と`earliest`の2番目の引数は、「最新」を「この属性による最高」に変更します。 `latest(traceId, duration)`は、最も遅いリクエストのトレース IDを返します。これをトレース検索に貼り付けることができます。

```sql
SELECT count(*) AS requests, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTrace, latest(urlPath, duration) AS slowestPath FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 15
```

`earliest(traceId, duration)`を使用すると、比較に最も速い例を取得できます。

### `percentOfTotal`の時間を検索

`percentOfTotal(attr)`は、各グループの合計シェアを示します。 `duration`に適用すると、どの操作が最も多くのサーバー時間を消費するかを示します。 これは、多くの場合、どの操作が最も遅いよりも便利です。

```sql
SELECT count(*) AS requests, sum(duration) AS totalMs, percentOfTotal(duration) AS shareOfServerTime FROM Transaction FACET name LIMIT 15
```

`percentOfTotal`を`TIMESERIES`と組み合わせることはできません。

### `SpanBreakdown`とのトランザクションを分割

取り込み時に`SpanBreakdown`が事前集計されます。 各トランザクション （`rootOperation`）が時間を費やした子操作（`segment`）を分単位で記録します。 「このエンドポイントは何をしているのか？」という質問に答えます 安い：

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown WHERE rootOperation LIKE '%GET%' FACET rootOperation, segment LIMIT 25
```

あるセグメントを時間をかけてトレンド化する：

```sql
SELECT average(avgDurationMs) AS avgMs FROM SpanBreakdown FACET segment TIMESERIES AUTO LIMIT 8
```

&#x200B;> [!NOTE]
> `avgDurationMs`はユニットで自動タグ付けされていません。 ウィジェット **Unit**&#x200B;を`ms`に設定します。

### 任意の指標の発見とクエリ

SQLに`keyset()`または`SHOW EVENT TYPES`がありません。 代わりに、レポートするすべての指標とその単位および最新の値をリストします。

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricGauge FACET metric, unit LIMIT 200
```

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricSum FACET metric, unit LIMIT 200
```

次に、任意の指標を名前でクエリします。 例えば、プールごとに使用されるJVM メモリは次のとおりです。

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool LIMIT 50
```

各サービスに入力されている&#x200B;**スパン属性**&#x200B;を確認するには、空白でない値をカウントします。

```sql
SELECT count(*) FILTER (WHERE http.route != '') AS withRoute, count(*) FILTER (WHERE db.system != '') AS dbCalls, count(*) FILTER (WHERE peer.service != '') AS withPeer, count(*) FILTER (WHERE error.type != '') AS withErrorType FROM Span FACET service LIMIT 50
```

&#x200B;> [!TIP]
> キュレーションされたエンティティが存在する場合（例：`CpuUtilization`）、`MetricGauge`よりも優先します。 適切なフィルターと単位の変換が適用され、ロールアップ高速化される場合があります。

### 適切なトレースエンティティの選択

| 使用方法 | 必要なとき | 例 |
| ------------- | ------------------------------------------------------------------------------------------- | --------------------------------- |
| `Transaction` | 受信HTTP/RPC リクエスト KPI （スループット、レイテンシ、エラー、Apdex）。 ロールアップ高速化： | ゴールデンシグナル |
| `AppRequest` | 同じKPI、**メッセージ消費者**&#x200B;を含む（Kafkaと類似） | イベント駆動型サービス |
| `Span` | リクエスト以下の任意の項目：アウトバウンド HTTP、DB、メッセージング、AEM Sling/JCR/JSP、内部スパン | 依存関係とコードパス分析 |

各スパンの種類をカウントして、何が実装されているかを確認します。

```sql
SELECT count(*) AS spans, percentile(duration, 95) AS p95 FROM Span FACET kind
```

外部ドメインによるアウトバウンドコール：

```sql
SELECT count(*) AS calls, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain LIMIT 20
```

### ネストされた集計の回避策

NRQLでは、「ホストごとの最大数の平均」などのサブクエリを使用して集計を集計できます。 SQLはそうではありません。 ここでは、その選択肢を紹介します。

| 目標 | SQL アプローチ |
| --------------------------------------- | ---------------------------------------------------------- |
| グループごとの値を入力し、グループをランク付けします | `FACET group` + `ORDER BY alias` + `LIMIT` |
| 2つの集計の比率 | [数式ウィジェット &#x200B;](#86-combine-queries-with-formula-widgets) |
| ホストごとの値、次に「最悪のホスト」 | `FACET host ORDER BY alias DESC LIMIT 1` |
| ホストあたりの平均時間 | `FACET host TIMESERIES AUTO`、シリーズを読む |

例：p95による最悪のホスト：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET host ORDER BY p95 DESC LIMIT 1
```
