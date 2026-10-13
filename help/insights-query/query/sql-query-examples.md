---
title: SQL クエリの例
description: Observability Insights テレメトリのSQL クエリの例を確認します。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '306'
ht-degree: 10%
---

## SQL クエリの例

例をいくつか挙げてみましょう。 このクエリの場合：

- エンティティとして`Transaction`を選択します。
- `SELECT`を使用して、95 パーセンタイル期間を計算します。
- サービス別に結果をグループ化するには、`FACET`を使用します。
- `TIMESERIES AUTO`を使用して、自動的にサイズ調整された間隔でデータをプロットします。

```sql
FROM Transaction SELECT percentile(duration, 95) AS p95 FACET service TIMESERIES AUTO
```

これにより、折れ線グラフでサービスごとに1行が生成され、デフォルトでは最もビジーな10のサービスが表示されます。

以下の各例をクリックして展開します。

<details>
<summary><strong> インバウンドリクエストの基本クエリ </strong></summary>

すべてのインバウンドリクエストの平均応答時間（ミリ秒単位）:

```sql
SELECT average(duration) FROM Transaction
```

</details>

<details>
<summary><strong> ドットを含む属性名</strong></summary>

ドット付きの名前は引用符で囲む必要はありません。

```sql
SELECT count(*) FROM Transaction WHERE http.statusCode >= 500 FACET http.route
```

</details>

<details>
<summary><strong>複数の列を返すクエリ </strong></summary>

1つのクエリ内の複数の集計がテーブルの列になります。

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, percentile(duration, 95) AS p95, max(duration) AS maxMs FROM Transaction FACET service
```

</details>

<details>
<summary>1回のパスで<strong>条件付き集計</strong></summary>

合計、エラー、5xxを並べて表示：

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, count(*) FILTER (WHERE http.statusCode >= 500) AS http5xx FROM Transaction FACET service
```

</details>

<details>
<summary><strong> キュレートされた指標のクエリ </strong></summary>

ホストあたりの時間の平均CPU:

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

</details>

<details>
<summary><strong>任意のOpenTelemetry メトリックを名前</strong>でクエリしています</summary>

キュレーションされたエンティティを持たない指標には`MetricGauge` / `MetricSum`を使用します：

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool
```

</details>

<a id="what-sql-doesnt-do"></a>

## SQLで不可能なこと

SQLは意図的に小さい。 これにより、クエリを安全に、迅速に、テナント分離できます。 これらの機能のいずれかが必要な場合は、代替を使用します。

| 機能 | ステータス | 代替 |
| ------------------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------ |
| `SELECT *` | サポート対象外 | 必要な属性をリストします：`SELECT timestamp, name, duration, traceId …`。 |
| 生の属性と集計の混在 | サポート対象外 | `FACET attr`または`latest(attr)` / `earliest(attr)`を使用してください。 |
| `SELECT`の計算（`a - b`、`count(*) / 60`） | サポート対象外 | **式** ウィジェット （ダッシュボード）。 |
| 結合とサブクエリ | サポート対象外 | **式** ウィジェット、または2つのウィジェットを並べて表示します。 |
| `HAVING` | サポート対象外 | `ORDER BY alias` + `LIMIT`. |
| `COMPARE WITH` | サポート対象外 | 時間ピッカーを変更するか、2つのウィジェットを別のタブに配置します。 |
| `FACET CASES` | サポート対象外 | 1つのケースにつき1つの`count(*) FILTER (WHERE …)`列（チュートリアル 3](#83-group-values-into-cases-with-filter)を参照）。[ |
| `histogram()` | サポート対象外 | バケット化された`FILTER`列（[ チュートリアル 3](#84-build-a-latency-histogram)を参照）。 |
| `uniques()` / `keyset()` | サポート対象外 | `FACET attr LIMIT n`には個別の値が一覧表示されます。 検出クエリには、指標と入力された属性が一覧表示されます。 |
| `RLIKE` （正規表現） | サポート対象外 | `LIKE` / `NOT LIKE` （ワイルドカード `%`および`_`）、または`IN (…)`。 |
| コメント （`--`、`//`、`/* */`） | サポート対象外 | ウィジェットのタイトルまたは&#x200B;**テキスト** ウィジェットにメモを配置します。 |
| `SINCE`の絶対日付 | サポート対象外 | 時間ピッカーの絶対範囲を使用します。 |
| 型変換（`numeric()`, `string()`） | 不要 | 属性はカタログに入力されます。 |
