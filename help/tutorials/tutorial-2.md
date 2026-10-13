---
source-git-commit: 263ba9e9b3eb1c6e4dd253ca03bd0ce8954c588b
workflow-type: tm+mt
source-wordcount: '344'
ht-degree: 0%
---
## チュートリアル 2：データの制御 {#tutorial-2-control-your-data}

このチュートリアルでは、未加工カウントを答えに変換する関数と句について説明します。具体的なカウント、最新の値、パーセンタイル、ワイルドカード、並べ替え、多次元グループ化です。

**このチュートリアルでは、** [7.1 uniqueCount](#71-count-distinct-values-with-uniquecount) ・ [7.2 latest/earliest](#72-find-the-latest-and-earliest-values) ・ [7.3 percentage](#73-calculate-percentages) ・ [7.4 percentile](#74-understand-latency-with-percentile) ・ [7.5 Labels](#75-make-results-readable-with-labels) ・ [7.6 Wildcards and lists](#76-match-patterns-and-lists) ・ [7.7 Sorting](#77-sort-and-rank-with-order-by) ・ [7.8 Multiple facets](#78-group-by-several-attributes) ・ [7.9 Past](#79-look-at-a-past-window)

### `uniqueCount`で異なる値をカウント

各サービスに含まれるホストと個別のトレースの数

```sql
SELECT uniqueCount(host) AS hosts, uniqueCount(traceId) AS traces FROM Transaction FACET service
```

`uniqueCount`は正確で、見積もりではありません。

**list**&#x200B;に対して、属性上のファセットで、異なる値を指定します（NRQLが`uniques()`と呼ぶもの）。

```sql
SELECT count(*) AS requests FROM Transaction FACET host LIMIT 100
```

### 最新の値と初期の値の検索

`latest(attr)`は最新の値を返し、`earliest(attr)`は最も古い値を返します。 これらは、「現在の状態」テーブルに便利です。

```sql
SELECT latest(value) AS currentCpu FROM CpuUtilization FACET host
```

```sql
SELECT earliest(timestamp) AS firstSeen, latest(timestamp) AS lastSeen FROM Transaction FACET service
```

### パーセンテージを計算

`percentage(*, WHERE condition)`は、条件に一致する行の共有を返します。 エラー率は次のとおりです。

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction
```

サービスごとのトレンド：

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

低速リクエストの割合を追跡することもできます。

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction FACET service
```

### `percentile`の待ち時間について

平均値は外れ値を非表示にします。 百分率は、ほとんどのユーザーが体験すること（p50）と、不運なユーザーが体験すること（p95、p99）を示します。

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

> [!TIP]
> `Transaction.duration`のp50、p95、およびp99は、長いウィンドウにわたってロールアップから提供されるので、毎週の表示のために高速に保たれます。

### ラベルを使って結果を読みやすくする

スペースを含むラベルに対して`AS`を一重引用符で囲みます：

```sql
SELECT count(*) AS 'Requests', percentile(duration, 95) AS 'p95 (ms)' FROM Transaction FACET service
```

> [!NOTE]
> エイリアスは`SELECT`個の項目にのみ適用されます。 ファセット列には、常に属性名のラベルが付けられます。

### パターンとリストの一致

`LIKE`は、任意の文字数に`%`を使用し、正確に1文字に`_`を使用します。

```sql
SELECT count(*) AS requests FROM Transaction WHERE urlPath LIKE '/content/%.html' FACET urlPath LIMIT 20
```

`NOT LIKE`の静的アセットを除外：

```sql
SELECT count(*) AS requests FROM Transaction WHERE urlPath NOT LIKE '%.js' AND urlPath NOT LIKE '%.css' AND urlPath NOT LIKE '%.png' FACET urlPath LIMIT 20
```

`IN`および`NOT IN`のリストと照合します（最大100個の値）:

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.method IN ('POST', 'PUT', 'DELETE') FACET service
```

> [!NOTE]
> SQLに正規表現（`RLIKE`）がありません。 `LIKE`個のパターンを`OR`と組み合わせるか、`IN`を使用してください。

### `ORDER BY`で並べ替えおよびランク付け

デフォルトでは、ファセット結果は最初の集計で並べ替えられます。 `ORDER BY`を使用して、p95による最も遅い操作などの他の項目でランク付けします。

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

最も利用されていないサービスを優先させるか：

```sql
SELECT count(*) AS requests FROM Transaction FACET service ORDER BY requests ASC LIMIT 10
```

> [!TIP]
> `HAVING`はありません。 「ビジー」操作のみを検索するには、`requests`でランク付けして上位Nを維持するか、ノイズを除去する`WHERE`を追加します（例：`WHERE http.method != 'OPTIONS'`）。

### 複数の属性でグループ化

`FACET`は、最大5つの属性を受け入れます。 各組み合わせは、1行または系列になります。

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode >= 400 FACET service, http.method, http.statusCode LIMIT 30
```

### 過去のウィンドウを見る

`COMPARE WITH`はありません。 ダッシュボードとエクスプローラーで、**時間ピッカー**&#x200B;を変更します。 明示的な範囲を指定せずにAPIを呼び出すと、`SINCE … UNTIL …`は過去のウィンドウを表示します。

```sql
SELECT count(*) AS lastWeekSameHour FROM Transaction SINCE 169 hours ago UNTIL 168 hours ago
```
