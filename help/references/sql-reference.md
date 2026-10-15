---
title: SQL参照
description: Observability InsightsでサポートされているSQLの構文、句および関数を参照します。
source-git-commit: 91a61d33010192cedce7d40a45ffe86f32e4824b
workflow-type: tm+mt
source-wordcount: '1505'
ht-degree: 12%
---

## SQL参照 {#sql-reference}

このリファレンスでは、SQLの構文、句、関数および演算子について説明します。 各エントリを展開して、詳細と例を確認します。

### クエリコンポーネント

```text
SELECT <select-item> [, <select-item> ...]
FROM   <Entity>
[WHERE      <condition>]
[FACET      <attribute> [, <attribute> ...]]          -- up to 5
[SINCE      <n> <unit> [AGO]]
[UNTIL      NOW | <n> <unit> [AGO]]
[TIMESERIES AUTO | MAX | <n> <unit>]
[ORDER BY   <alias-or-selected-name> [ASC | DESC]]
[LIMIT      <1..5000>]

<select-item> := <function>(<args>) [FILTER (WHERE <condition>)] [AS <alias>]
               | <attribute> [AS <alias>]
<unit>        := second | seconds | minute | minutes | hour | hours | day | days | week | weeks
```

> **メモ**
>
> 上記の`--`注釈はドキュメントのみです。 SQL自体はコメントを受け付けません。

### 必要な句

#### `SELECT` – 返される値

+++詳細を表示

`SELECT`には、**生の属性** （イベントクエリ）または&#x200B;**アグリゲーター関数** （集計クエリ）が一覧表示されます。 この2つを混ぜることはできません。

| フォーム | 例 | 結果 |
| ------------ | -------------------------------------------------- | ------------------------------------------------- |
| 属性 | `SELECT timestamp, name, duration` | 個々のイベントの行（デフォルトは100） |
| 集計 | `SELECT count(*), average(duration)` | 集計ごとに1つの値（ファセットごと/時間バケットごと） |
| エイリアスを使用 | `SELECT count(*) AS requests` | 列ラベル `requests` |
| 引用符で囲まれたエイリアス | `SELECT percentile(duration, 95) AS 'p95 latency'` | スペースを含むラベル |

イベントリスト：

```sql
SELECT timestamp, service, name, duration, status, traceId FROM Transaction WHERE status = 'Error' LIMIT 50
```

複数の集計：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, max(duration) AS maxMs FROM Transaction
```

ルール：

- `SELECT *`はサポートされていません。 必要な属性をリストアップします。
- 属性または集計（`sum(a) / count(*)`）間の計算はサポートされていません。 [数式ウィジェット &#x200B;](#86-combine-queries-with-formula-widgets)を使用します。
- 属性と集計の混在が&#x200B;_Raw属性と集計を混在させることはできません。結果をグループ化するにはFACETを使用してください_。

+++

#### `FROM` — クエリするエンティティ

+++詳細を表示

`FROM`さんが、[&#x200B; データディクショナリ &#x200B;](#data-dictionary)から&#x200B;**one** エンティティを正確に名前を付けます。 エンティティ名では大文字と小文字が区別されません。

```sql
SELECT count(*) FROM Transaction
```

```sql
FROM Span SELECT count(*) FACET kind
```

各エンティティは、固定された範囲を持ちます。 例えば、`Transaction`は常にサーバーのスパンであり、`CpuUtilization`は常に`value`がパーセントに変換されたCPU ビジー指標です。 1つのクエリで複数のエンティティをクエリすることはできません。 エンティティを結合するには、[数式ウィジェット &#x200B;](#86-combine-queries-with-formula-widgets)を使用します。

+++

### オプションの句

#### `AS` – 列にラベルを付けます

+++詳細を表示

`AS`を使用して、集計または属性に読みやすい名前を付けます。 エイリアスは、列ヘッダー、凡例ラベルおよび`ORDER BY`で参照する名前になります。

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET service ORDER BY p95 DESC LIMIT 10
```

スペースを含むラベルには、一重引用符で囲まれた文字列を使用します。

```sql
SELECT percentile(duration, 95) AS 'p95 latency (ms)' FROM Transaction TIMESERIES AUTO
```

+++

#### `FACET` — グループ結果

+++詳細を表示

`FACET`は、SQL `GROUP BY`のように、最大&#x200B;**5**&#x200B;属性で結果をグループ化します。

```sql
SELECT count(*) FROM Transaction FACET service, http.method, http.statusCode LIMIT 30
```

| クエリ | `LIMIT n`の意味 | デフォルト |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| `TIMESERIES`を含まない`FACET` | 行数 | 10 |
| `FACET` with `TIMESERIES` | **最上位n系列**。ウィンドウ全体の最初の集計の合計でランク付けされます。 すべてのバケットは、このシリーズ用に保持されます。 | 全シリーズ（5,000行を上限） |

ファセット付き時系列、上位5:

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 5
```

ファセットの値には、文字列または数値を使用できます。 空白の文字列は、独自のグループを形成します。 `WHERE attr != ''`でそれらを除外します。

+++

#### `FILTER` – 条件付き集計

+++詳細を表示

任意の集計に`FILTER (WHERE condition)`を追加して、行のサブセットで計算します。 これにより、1回のパスで複数の条件を比較できます。

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentile(duration, 95) FILTER (WHERE http.method = 'GET') AS getP95 FROM Transaction FACET service LIMIT 20
```

`FILTER`は、`apdex`、`rate`または`derivative`では使用できません。 `percentage`は既に独自の条件を使用しています。

+++

#### `LIMIT` – 行数または系列の数

+++詳細を表示

`LIMIT`は正の整数を取ります。 5,000を超える値は5,000に固定されます。

| 結果シェイプ | デフォルト |
| ------------ | ---------- |
| 看板 | 1 |
| ファセット | 10 |
| 時系列 | 5,000行 |
| イベント | 100 |

```sql
SELECT count(*) FROM Transaction FACET urlPath LIMIT 50
```

>[!TIP]
>
> クエリ駆動変数と高基数ファセット （`urlPath`、`traceId`、`jcr.path`、`url`）には常に`LIMIT`を設定します。

+++

#### `ORDER BY` – 結果の並べ替え

+++詳細を表示

`ORDER BY`は、**選択したエイリアスまたは選択した属性**&#x200B;で並べ替えます。 既定の方向は`DESC`です。

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

| 結果シェイプ | デフォルトの注文 |
| ----------------- | --------------------------------------------------------------------------- |
| 看板/ファセット | 最初の集計、`DESC` |
| 時系列 | 時間、`ASC` |
| イベント | 最初に選択された列、`DESC` （「新しい最初」には`timestamp`を最初に入力） |

ルール：

- 選択されていない項目を参照すると、_ORDER BY &#39;x&#39;で失敗します。選択した値またはエイリアス_&#x200B;を参照する必要があります。
- `TIMESERIES` クエリに`ORDER BY`を追加しないでください。 チャートで必要な時間の順序が上書きされます。

+++

#### `SINCE` — ウィンドウの開始

+++詳細を表示

`SINCE n unit [AGO]`は、ウィンドウの開始位置を設定します。 単位：`second`、`minute`、`hour`、`day`、`week` （単数または複数形）。 `AGO`はオプションです。

```sql
SELECT count(*) FROM Transaction SINCE 6 hours ago
```

- デフォルトは、`SINCE 1 hour ago` です。
- ウィンドウは&#x200B;**31日**&#x200B;を超えることはできません（_時間範囲は31日_）を超えることはできません。
- 絶対日付（`SINCE '2026-01-01'`）と`TODAY` / `YESTERDAY`はサポートされていません。

>[!IMPORTANT]
>
> ダッシュボードとエクスプローラーで、**時間ピッカーが** `SINCE`と`UNTIL`を上書きします。 [時間ウィンドウの優先順位](#time-window-precedence)を参照してください。

+++

#### `TIMESERIES` – 時間の経過に伴うプロット

+++詳細を表示

`TIMESERIES`は、折れ線グラフ、エリア グラフまたは棒グラフの準備ができている時間バケットごとに1行（ファセット値ごとに）を返します。

| フォーム | バケットサイズ |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `TIMESERIES 5 minutes` | あなたが求めているもの（任意のユニット） |
| `TIMESERIES AUTO` | 120÷ウィンドウについて、1s / 5s / 10s / 15s / 30s / 1m / 5m / 10m / 15m / 30m / 1h / 3h / 6h / 12h / 1dに丸めます。 **2分以内に完了しない。** |
| `TIMESERIES MAX` | 1秒バケットです。 数分以上は使用しないでください。 |

典型的な`AUTO` バケット：

| ウィンドウ | 1時間 | 6時間 | 24時間 | 7 d | 30 d |
| ------ | ----- | ----- | ------ | --- | ---- |
| バケット | 2分 | 5分 | 15分 | 3時間 | 6時間 |

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction TIMESERIES AUTO
```

```sql
SELECT count(*) FROM Transaction TIMESERIES 1 hour SINCE 1 day ago
```

>[!TIP]
>
> ダッシュボードでは`AUTO`を優先します。 タイムピッカーが何と言っても約120 ポイントを保持します。 また、ホスト指標は約90秒ごとにスクレイピングされるので、ホストごとに「粉々になる」ことも回避できます。

+++

#### `UNTIL` — ウィンドウの終わり

+++詳細を表示

`UNTIL NOW` （デフォルト）または`UNTIL n unit [AGO]`。 `SINCE`と組み合わせて、過去のウィンドウを表示します。

```sql
SELECT count(*) AS yesterday FROM Transaction SINCE 2 days ago UNTIL 1 day ago
```

`SINCE`は`UNTIL`より前である必要があります。 それ以外の場合、クエリは&#x200B;_SINCEで失敗します。これは、_&#x200B;以前である必要があります。

+++

#### `WHERE` – 行のフィルタリング

+++詳細を表示

`WHERE`は、条件に一致する行のみを保持します。 条件を`AND`、`OR`、`NOT`および括弧と組み合わせます。 [演算子と条件](#45-operators-and-conditions)を参照してください。

```sql
SELECT count(*) FROM Transaction WHERE service = 'checkout' AND (status = 'Error' OR http.statusCode >= 500)
```

`WHERE`は1回のみ表示できます。 「すべての条件を入れる」。

+++

### 集計関数

| 関数 | 引数 | 戻り値 | 作業中 |
| -------------------------------------- | ----------------------------------- | ---------------------- | ------------------------------------- |
| [`apdex`](#fn-apdex) | 数値デュレーション、しきい値`T` | 0-1 | トレースエンティティ |
| [`average` / `avg`](#fn-average) | 数値 | 入力と同じ単位 | 任意 |
| [`count`](#fn-count) | `*`または属性 | 行数 | 任意 |
| [`derivative`](#fn-derivative) | 数値 | 1秒当たりの変化 | 任意（`TIMESERIES`が必要） |
| [`earliest`](#fn-latest) | 任意の[、ランク ] | 属性値 | 任意 |
| [`latest`](#fn-latest) | 任意の[、ランク ] | 属性値 | 任意 |
| [`max` / `min`](#fn-minmax) | いずれか | 入力と同じ単位 | 任意 |
| [`percentage`](#fn-percentage) | `*`または属性、`WHERE`条件 | % | 任意 |
| [`percentile`](#fn-percentile) | 数値、レベル `p` | 入力と同じ単位 | 任意 |
| [`percentOfTotal`](#fn-percentoftotal) | 数値 | % | 任意（なし`TIMESERIES`） |
| [`rate`](#fn-rate) | 数値カウンター | 毎秒 | カウンターエンティティ （必要なのは`TIMESERIES`） |
| [`stddev`](#fn-stddev) | 数値 | 入力と同じ単位 | 任意 |
| [`sum`](#fn-sum) | 数値 | 入力と同じ単位 | 任意 |
| [`uniqueCount`](#fn-uniquecount) | いずれか | 個別の値 | 任意 |

<a id="fn-apdex"></a>

#### `apdex(attribute, T)`

+++詳細を表示

属性の単位のしきい値`T`を使用して、[Apdex](https://www.apdex.org/)のスコアを0 （不満）から1 （満足）に計算します（`duration`のミリ秒）。

| バケット | 条件 | 線幅 |
| ---------- | ------------------------------------ | ------ |
| 満足度 | `duration ≤ T`でエラーではありません | 1 |
| 耐える | `T < duration ≤ 4T`でエラーではありません | 0.5 |
| 不満 | `duration > 4T`、**またはエラー** | 0 |

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service
```

- トレース エンティティ （`Transaction`、`AppRequest`、`Span`）でのみ機能します。
- `T`は0より大きくなければなりません。 `FILTER`は許可されていません。
- `Transaction`の`apdex(duration, 500)`は、長いウィンドウでロールアップ高速化されます。

+++

<a id="fn-average"></a>

#### `average(attribute)` / `avg(attribute)`

+++詳細を表示

数値属性の算術平均。

```sql
SELECT average(duration) AS avgMs FROM Transaction FACET service TIMESERIES AUTO
```

+++

<a id="fn-count"></a>

#### `count(*)` / `count(attribute)`

+++詳細を表示

- `count(*)`は行をカウントします。
- `count(attribute)`は、属性が&#x200B;**NULL**&#x200B;でない行をカウントします。 文字列属性の場合、欠落している値はNULLではなく空白（`''`）であるため、まだカウントされます（[型処理](#46-type-handling-blank-vs-null)を参照）。

```sql
SELECT count(*) AS requests, count(http.statusCode) AS withStatusCode FROM Transaction
```

+++

<a id="fn-derivative"></a>

#### `derivative(attribute)`

+++詳細を表示

連続するバケット間の1秒あたりの`average(attribute)`の変更：（このバケット−前のバケット）÷ バケットの秒数。 結果は否定的になる可能性があります。 ゲージで使用して、値が増加しているか減少しているかを確認します。

```sql
SELECT derivative(value) AS heapGrowthPerSec FROM JvmHeapUsed WHERE host = 'publish-1' TIMESERIES 5 minutes
```

- `TIMESERIES`が必要です。**のみ**&#x200B;集計である必要があります。 `FILTER`は許可されていません。
- 最初のバケットには先行タスクがありません。

+++

<a id="fn-latest"></a>

#### `latest(attribute [, rankAttribute])` / `earliest(…)`

+++詳細を表示

- `latest(attr)`は、最新の行の値を返します。 `earliest(attr)`は、最も古い行の値を返します。
- 2番目の引数を指定すると、行は時間ではなく、その属性によって選択されます。 `latest(traceId, duration)`は、**最も遅い**&#x200B;要求のトレース IDを返します。 `earliest(traceId, duration)`は最速を返します。

```sql
SELECT latest(value) AS current FROM MemoryUtilization FACET host
```

```sql
SELECT max(duration) AS slowestMs, latest(traceId, duration) AS slowestTrace FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 10
```

+++

<a id="fn-minmax"></a>

#### `max(attribute)` / `min(attribute)`

+++詳細を表示

最大値または最小値。 数字、文字列、タイムスタンプで動作します。

```sql
SELECT min(duration) AS fastestMs, max(duration) AS slowestMs, max(timestamp) AS lastSeen FROM Transaction FACET service
```

+++

<a id="fn-percentage"></a>

#### `percentage(*, WHERE condition)`

+++詳細を表示

`condition`に一致する行のシェア （0 ～ 100）。

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service TIMESERIES AUTO
```

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction
```

`percentage`には独自の条件があるため、`FILTER`は使用されません。

+++

<a id="fn-percentile"></a>

#### `percentile(attribute, p)`

+++詳細を表示

おおよそのp番目のパーセンタイル（`0 < p ≤ 100`）。 レベルごとに1つの`percentile`を使用してください。

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

`Transaction`のp50、p95、p99は、長いウィンドウでロールアップ高速化されます。

+++

<a id="fn-percentoftotal"></a>

#### `percentOfTotal(attribute)`

+++詳細を表示

数値属性の&#x200B;**合計合計**&#x200B;の各ファセット行のシェア。 行は合計で100%になります。

```sql
SELECT sum(duration) AS totalMs, percentOfTotal(duration) AS shareOfTime FROM Transaction FACET name LIMIT 15
```

`TIMESERIES`では許可されていません。

+++

<a id="fn-rate"></a>

#### `rate(attribute)`

+++詳細を表示

**累積カウンター**&#x200B;の1秒あたりの率。 各バケットで、`rate`はシリーズごとに最も高いカウンター読み取りを取り、前のバケットと比較します。 カウンターリセットは0で実行され、最初のバケットは0です。

```sql
SELECT rate(value) AS bytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

- カウンターのエンティティ（[&#x200B; データディクショナリ &#x200B;](#53-curated-metric-entities)の✔とマークされている）でのみ機能します。
- `TIMESERIES`が必要です。**のみ**&#x200B;集計である必要があります。 `FILTER`は許可されていません。

>[!WARNING]
>
> 常に`FACET host`、または単一のホストにフィルターを適用します。 多くのホストでファセットされていない`rate()`は、異なるホストのカウンターを比較し、意味のない数値を生成します。 ディスク カウンターはデバイスごとに存在するため、`FACET host, device`が最も正確です。

+++

<a id="fn-stddev"></a>

#### `stddev(attribute)`

+++詳細を表示

サンプル標準偏差。 `average`と併用して、待ち時間の一貫性を判断します。

```sql
SELECT average(duration) AS avgMs, stddev(duration) AS stddevMs FROM Transaction FACET name LIMIT 20
```

+++

<a id="fn-sum"></a>

#### `sum(attribute)`

+++詳細を表示

数値属性の合計。

```sql
SELECT sum(callCount) AS calls FROM SpanBreakdown FACET segment LIMIT 20
```

+++

<a id="fn-uniquecount"></a>

#### `uniqueCount(attribute)`

+++詳細を表示

個別の値の&#x200B;**exact**&#x200B;個。

```sql
SELECT uniqueCount(host) AS hosts, uniqueCount(traceId) AS traces FROM Transaction FACET service
```

+++

機能制約の概要：

| 制約 | 関数 |
| ------------------------------------- | ----------------------------------------------------------------------------------------- |
| 数値属性が必要です | `average`, `sum`, `percentile`, `stddev`, `percentOfTotal`, `rate`, `derivative`, `apdex` |
| いいえ`FILTER (WHERE …)` | `apdex`, `rate`, `derivative`, `percentage` |
| `TIMESERIES`が必要です。集計のみ | `rate`, `derivative` |
| `TIMESERIES`には対応していません | `percentOfTotal` |
| トレースエンティティのみ | `apdex` |
| カウンターエンティティのみ | `rate` |

### 演算子と条件

| 演算子 | 意味 | 例 |
| ------------------------- | -------------------------------------------------------------------------- | ---------------------------------------- |
| `=` | 等号 | `status = 'Error'` |
| `!=`, `<>` | 等しくない | `kind != 'Internal'` |
| `<`, `<=`, `>`, `>=` | 比較 | `http.statusCode >= 500` |
| `LIKE` | ワイルドカード一致：`%` =任意の文字、`_` = 1文字。 大文字と小文字を区別。 | `urlPath LIKE '/content/%'` |
| `NOT LIKE` | 否定的なワイルドカード | `urlPath NOT LIKE '%.css'` |
| `IN (…)` | 1 ～ 100個の値のいずれかに一致 | `service IN ('checkout', 'cart')` |
| `NOT IN (…)` | どの値にも一致しません | `http.method NOT IN ('OPTIONS', 'HEAD')` |
| `IS NULL` / `IS NOT NULL` | ヌルチェック | `http.statusCode IS NOT NULL` |
| `AND`, `OR`, `NOT` | ロジック | `NOT (status = 'Ok')` |
| `( … )` | グループ化 | `(a = 1 OR b = 2) AND c = 3` |

```sql
SELECT count(*) FROM Transaction WHERE (status = 'Error' OR duration > 2000) AND NOT host LIKE '%dispatcher%' AND http.method IN ('GET', 'POST')
```

> **警告**
>
> `attr = NULL`はエラーです。 `attr IS NULL` の使用

### タイプ処理：空白とNULL

すべての属性には、カタログ内の型（`string`、`number`、`boolean`、または`timestamp`）があります。 SQL型チェック関数を指定します（例：`average(service)`は拒否されます）。 タイプ変換関数は必要ないか、使用できます。

> **重要**
>
> **空白vs NULL。** ほとんどの文字列属性はOpenTelemetry属性マップから取得されます。このマップでは、_missing_ キーは`NULL`ではなく`''`空の文字列として読み取られます。 属性を持つスパンのみを保持するには、`IS NOT NULL`ではなく&#x200B;**`attr != ''`**&#x200B;を使用します。 `http.statusCode`などの数値属性は、欠席時に真の`NULL`となるため、`IS NOT NULL`が機能します。

```sql
SELECT count(*) AS jcrCalls FROM Span WHERE jcr.operation != '' FACET jcr.operation
```

### SQLからSQLへの翻訳

| SQL | SQL |
| ------------------------------------------------------------ | ------------------------------------------------------------------- |
| `WHERE ts > now() - INTERVAL 1 HOUR` | なし。時間選択（または`SINCE 1 hour ago`）はウィンドウを設定します |
| `GROUP BY service` | `FACET service` |
| `GROUP BY toStartOfInterval(ts, INTERVAL 5 MINUTE), service` | `FACET service TIMESERIES 5 minutes` |
| `quantile(0.95)(duration)` | `percentile(duration, 95)` |
| `countIf(status = 'Error')` | `count(*) FILTER (WHERE status = 'Error')` |
| `100 * countIf(x) / count()` | `percentage(*, WHERE x)` |
| `count(DISTINCT host)` | `uniqueCount(host)` |
| `argMax(traceId, duration)` | `latest(traceId, duration)` |
| `x / sum(x) OVER ()` | `percentOfTotal(x)` |
| `ORDER BY p95 DESC LIMIT 20` | `ORDER BY p95 DESC LIMIT 20` |
| `HAVING count() > 100` | _サポートされていません_。 代わりに`LIMIT`を並べ替えます。 |
| `JOIN` / サブクエリ | _サポートされていません_。 **式** ウィジェットを使用します。 |
| `CASE WHEN` | _サポートされていません_。 `FILTER`列、または`WHERE … LIKE` + `FACET`を使用してください。 |

### 既存のダッシュボードのインサイトへのインポート

既存のクエリを知っている場合は、次に移行の詳細を示します

| 既存のクエリ | SQL | メモ |
| ------------------------------------------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `FROM Transaction SELECT count(*)` | 同上 | `FROM` ファースト注文がサポートされています |
| `SINCE 1 day ago` | 同上 | ダッシュボードでは無視され、ピッカーが勝ちます |
| `TIMESERIES AUTO` / `MAX` / `5 minutes` | 同上 | `AUTO`階は2分です |
| `FACET appName` | `FACET appName` / `FACET service` |                                                                                  |
| `percentage(count(*), WHERE error IS true)` | `percentage(*, WHERE status = 'Error')` |                                                                                  |
| `filter(count(*), WHERE …)` | `count(*) FILTER (WHERE …)` | SQL標準フォーム |
| `apdex(duration, t: 0.5)` | `apdex(duration, 500)` | `duration`さんは&#x200B;**ms**&#x200B;にいます |
| `latest(x)` / `earliest(x)` | 同上 | ランク付きフォームを含む：`latest(x, rankAttr)` |
| `uniqueCount(x)` | 同上 | 完全一致 |
| `rate(count(*), 1 minute)` | `count(*)` with `TIMESERIES 1 minute` | SQL `rate()`はカウンター専用です |
| `derivative(x, 1 minute)` | `derivative(x)` + `TIMESERIES` | 毎秒 |
| `uniques(x)` | `count(*) … FACET x LIMIT n` |                                                                                  |
| `keyset()` / `SHOW EVENT TYPES` | [検出クエリ &#x200B;](../examples/query-cookbook.md#data-discovery)、またはクエリガイドパネル |                                                                                  |
| `histogram(duration, …)` | `FILTER` バケット | [&#x200B; チュートリアル 3](../tutorials/tutorial-3.md#build-a-latency-histogram)を参照 |
| `FACET CASES (…)` | `FILTER`列 | [&#x200B; チュートリアル 3](../tutorials/tutorial-3.md#group-values-into-cases-with-filter)を参照 |
| `COMPARE WITH 1 week ago` | サポート対象外 | 時間ピッカーを変更するか、API経由で`SINCE … UNTIL …`を使用します |
| `RLIKE` | `LIKE` / `IN` |                                                                                  |
| `SELECT`の数式/ネストされた集計 | **式** ウィジェット |                                                                                  |
| `WITH TIMEZONE`, `EXTRAPOLATE`, `SLIDE BY` | サポート対象外 |                                                                                  |
