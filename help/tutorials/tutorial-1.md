---
title: チュートリアル 1：最初のクエリ
description: Observability Insights テレメトリに対して最初のSQL クエリを作成します。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '576'
ht-degree: 0%
---

## チュートリアル 1：最初のクエリ {#tutorial-1-your-first-queries}

SQLへようこそ。 このチュートリアルでは、最初のクエリの作成、時間ウィンドウの制御、トレンドのプロット、フィルターとグループ化の結果、最初のグラフをダッシュボードに配置します。 各ステップは前のステップの上に構築されます。

**このチュートリアルでは、** [開始する前](#before-you-begin) ・ [6.1 クエリイベント ](#61-query-events-with-select-from-and-limit) ・ [6.2 Aggregate](#62-aggregate-your-data) ・ [6.3いくつかの集計](#63-return-several-aggregates-at-once) ・ [6.4時間ウィンドウ ](#64-set-the-time-window) ・ [6.5時間](#65-plot-trends-with-timeseries) ・ [6.6 WHERE](#66-filter-with-where) ・ [6.7 FACET](#67-group-with-facet) ・ [6.8 ダッシュボードに保存](#68-save-your-query-to-a-dashboard)

### 始める前に

- Observability Insightsにアクセスし、少なくとも1つの顧客テナントがトレースまたは指標を送信する必要があります。
- テナントスイッチャーで&#x200B;**customer**&#x200B;を選択します。 **すべてのアカウント**&#x200B;が選択されている間、SQLは実行されません。
- **探索**&#x200B;を開き、**SQL** タブに切り替えます。 各クエリの後に&#x200B;**実行** （⌘/Ctrl + Enter）を使用します。

### `SELECT`、`FROM`、`LIMIT`のイベントをクエリします

すべてのクエリには2つの句が必要です。`SELECT`は何を返すかを示し、`FROM`はどのエンティティを読むかを示します。 インバウンドリクエストから始める：

```sql
SELECT timestamp, service, name, duration FROM Transaction
```

これは&#x200B;**イベントクエリ**&#x200B;です。 各行は1つのリクエストです。 `LIMIT`がないと、最大100行になります。 要求する回数を減らす：

```sql
SELECT timestamp, service, name, duration FROM Transaction LIMIT 10
```

行は最初の列で降順に並べ替えられるので、`timestamp`を最初に配置すると、最新のリクエストが上部に表示されます。

>[!NOTE]
>
> SQLに`SELECT *`がありません。 属性のリストは、結果を小さく読みやすいものにします。 [ データディクショナリ ](#data-dictionary)とエディターのオートコンプリートに、使用可能な内容が表示されます。

### データの集約

多くのダッシュボードでは、個々のイベントは表示されていません。 要約します。 属性を&#x200B;**アグリゲーター関数**&#x200B;に置き換えます。

```sql
SELECT count(*) FROM Transaction
```

これはウィンドウ内のインバウンドリクエストの数です。 もう少し試してください：

```sql
SELECT average(duration) FROM Transaction
```

```sql
SELECT max(duration) FROM Transaction
```

```sql
SELECT uniqueCount(host) FROM Transaction
```

### 複数の集計を一度に返します

集計をコンマで区切り、それぞれに`AS`という名前を付けます。

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, max(duration) AS maxMs FROM Transaction
```

>[!TIP]
>
> エイリアスは列ヘッダーおよび凡例ラベルとなり、これらは`ORDER BY`で参照されるものです。 すべての集計に名前を付ける習慣に入ります。

### 時間ウィンドウの設定

デフォルトでは、クエリは&#x200B;**過去1時間**&#x200B;をカバーします。 `SINCE`を使用して詳細を確認します：

```sql
SELECT count(*) AS requests FROM Transaction SINCE 1 day ago
```

`UNTIL`を使用して、過去にウィンドウを終了します。 これは、48時間から24時間前までの昨日のトラフィックをカウントします。

```sql
SELECT count(*) AS requests FROM Transaction SINCE 2 days ago UNTIL 1 day ago
```

単位は`second`、`minute`、`hour`、`day`または`week`です。 ウィンドウは最大で31日間です。

>[!IMPORTANT]
>
> **探索**&#x200B;および&#x200B;**ダッシュボード**&#x200B;では、ページ上部の&#x200B;**時間ピッカー**&#x200B;がウィンドウを制御し、`SINCE`/`UNTIL`を上書きします。 別の期間を見るには、ピッカーを変更します。 このガイドの残りの部分では、その理由で`SINCE`がダッシュボードクエリから除外されています。

### `TIMESERIES`でトレンドをプロット

`TIMESERIES`を追加して、1つの数値を時間をかけて1行に変換します。

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

`AUTO`はバケット サイズを選択して、グラフに約120 ポイントが含まれるようにします（1時間ウィンドウの場合は2分、24時間の場合は15分）。 バケットを明示的に設定することもできます。

```sql
SELECT average(duration) AS avgMs FROM Transaction TIMESERIES 5 minutes
```

### `WHERE`でフィルター

`WHERE`は自分が気にする行のみを保持します。 失敗したリクエストのみをカウント：

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error'
```

条件を`AND`、`OR`および括弧と組み合わせる：

```sql
SELECT count(*) AS slowOrFailed FROM Transaction WHERE service = 'checkout' AND (status = 'Error' OR duration > 2000)
```

部分一致に`LIKE`を`%`と共に使用：

```sql
SELECT count(*) AS contentRequests FROM Transaction WHERE urlPath LIKE '/content/%'
```

>[!NOTE]
>
> 文字列値では大文字と小文字が区別され、一重引用符を使用します：`"Error"`または`'error'`ではなく`'Error'`。 `checkout`をテナントのサービスに置き換えます。 次の手順では、それらを一覧表示する方法を示します。

### `FACET`さんとグループ化

SQLの`GROUP BY`など、`FACET`件のグループの結果。 サービスのリストと、各サービスの利用頻度を表示します。

```sql
SELECT count(*) AS requests FROM Transaction FACET service
```

ファセットクエリでは、デフォルトで上位10個のグループが返されます。 `LIMIT`を使用して詳細を表示：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs FROM Transaction FACET service LIMIT 50
```

`FACET`と`TIMESERIES`を組み合わせて、グループごとに1行を取得します。

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 5
```

ここでは、`LIMIT 5`は5つの最も混雑したシリーズを保持しています。

### クエリをダッシュボードに保存します

1. **ダッシュボードを開く→ ダッシュボードを作成→ カスタムダッシュボードを作成**&#x200B;し、_最初のダッシュボード_&#x200B;と名前を付けます。
2. 「**ウィジェットを追加**」をクリックし、**単一クエリ**&#x200B;を選択し、トグルを&#x200B;**SQL**&#x200B;に切り替えます。
3. 手順6.7のクエリを貼り付け、**クエリを実行**&#x200B;をクリックします。
4. **タイトル**&#x200B;を&#x200B;_サービス別リクエスト_&#x200B;に設定し、**折れ線** グラフの種類を選択します。
5. 上部のバーで「**ウィジェットを保存**」、「**保存**」の順にクリックします。

初めてのダッシュボードを構築しました。
