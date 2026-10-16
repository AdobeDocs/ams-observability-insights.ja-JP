---
title: 'チュートリアル 3: ダッシュボードを進める'
description: Observability Insightsで変数、式、複数のウィジェットを使用してダッシュボードを構築します。
source-git-commit: 09c8edf64e744ff0c16acd0d5905fc6796490f82
workflow-type: tm+mt
source-wordcount: '878'
ht-degree: 3%
---

## チュートリアル 3: ダッシュボードを進める {#tutorial-3-advance-your-dashboarding}

このチュートリアルでは、ユーザーエクスペリエンススコア、カウンター率、ケースグループ化、ヒストグラム、しきい値、セクション、数式ウィジェット、変数を含むKPI タイルなど、実稼動用グレードのダッシュボードのコンポーネントを構築します。

### `apdex`でユーザーエクスペリエンスをスコア化

Apdexは、待ち時間とエラーを0から1のスコアに変換します。 500 ミリ秒のしきい値が500 ミリ秒以下の場合、リクエストは&#x200B;_satisfied_、最大2,000 ミリ秒まで&#x200B;_tolerating_、その&#x200B;**以上&#x200B;_いら立つ_、失敗した場合**:

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

サービスごとのトレンド：

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

しきい値`0 → Red`、`0.7 → Amber`、`0.85 → Green`を持つ&#x200B;**Number** ウィジェットに入れます。 最も高いマッチングバンドが勝つので、低いスコアは赤のままです。

### `rate`でカウンターをレートに変換

[&#x200B; データディクショナリ &#x200B;](#53-curated-metric-entities)）ストアのカウンターエンティティ（✔）の合計は増加し続けています。 `rate(value)`は、1秒あたりの値に変換します。 `TIMESERIES`が必要です。集計は次のみである必要があります：

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host, device TIMESERIES AUTO LIMIT 10
```

>[!WARNING]
>
> 常に`rate()`を`host` （およびディスクの場合は`device`）によってファセットするか、1つのホストにフィルター処理します。 これがなければ、異なるホストからの読み取りが混在し、結果は意味がありません。

トレースから&#x200B;**リクエスト**&#x200B;のレートを取得するには、代わりに固定バケットでリクエストをカウントします。 これは、1分あたりのリクエストを与えます。

```sql
SELECT count(*) AS requestsPerMinute FROM Transaction TIMESERIES 1 minute
```

### 値を`FILTER`のケースにグループ化します

NRQLの`FACET CASES`は、行を名前付きバケットにグループ化します。 SQLでは、1つのケースにつき1つの`FILTER`列を書き込みます。 サービスごとのステータスクラスは次のとおりです。

```sql
SELECT count(*) FILTER (WHERE http.statusCode < 400) AS ok, count(*) FILTER (WHERE http.statusCode >= 400 AND http.statusCode < 500) AS clientErrors, count(*) FILTER (WHERE http.statusCode >= 500) AS serverErrors FROM Transaction FACET service LIMIT 20
```

同じ考え方が時間の経過とともに機能し、ケースごとに1行を与えます。

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/graphql/%') AS graphql FROM Transaction TIMESERIES AUTO
```

### 待ち時間ヒストグラムの作成

SQLに`histogram()`関数がありません。 `FILTER`で自分でバケットを作成し、結果を&#x200B;**棒**&#x200B;または&#x200B;**表** ウィジェットとしてレンダリングします。

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS 'le 100ms', count(*) FILTER (WHERE duration > 100 AND duration <= 250) AS '100-250ms', count(*) FILTER (WHERE duration > 250 AND duration <= 500) AS '250-500ms', count(*) FILTER (WHERE duration > 500 AND duration <= 1000) AS '0.5-1s', count(*) FILTER (WHERE duration > 1000 AND duration <= 2000) AS '1-2s', count(*) FILTER (WHERE duration > 2000) AS 'over 2s' FROM Transaction
```

`FACET service`を追加して、複数のサービスのディストリビューションを並べて比較します。

### KPI タイルとセクションを構築する

優れたダッシュボードは、KPI タイルの行から始まり、セクションで詳細チャートをグループ化します。

1. 4つの&#x200B;**Number** ウィジェットを追加します。各3列は幅が広くなります。

   | タイトル | クエリ | 単位 | しきい値 |
   | -------- | --------------------------------------------------------------------------- | ---- | ---------------------------------------- |
   | リクエスト | `SELECT count(*) AS requests FROM Transaction` |      |                                          |
   | エラー% | `SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction` | `%` | `0 → Green`, `1 → Amber`, `5 → Red` |
   | p95 | `SELECT percentile(duration, 95) AS p95 FROM Transaction` | `ms` | `0 → Green`, `800 → Amber`, `2000 → Red` |
   | Apdex | `SELECT apdex(duration, 500) AS apdex FROM Transaction` |      | `0 → Red`, `0.7 → Amber`, `0.85 → Green` |

2. **テキスト** ウィジェットを追加し、_これをセクションヘッダー_&#x200B;にして、**待ち時間**&#x200B;と呼びます。 その下に待ち時間チャートを追加します。
3. 別のセクションヘッダー「**ERRORS**」とエラーチャートを追加します。 断面を山形で折りたたんで、もう一方に焦点を合わせます。
4. 様々なオーディエンスに対して&#x200B;**タブ**&#x200B;を使用します。例：_概要_、_AEM_、_インフラストラクチャ_。 クエリを実行するのは、「表示」タブのみです。

### クエリを数式ウィジェットと組み合わせる

SQLは`SELECT`で計算を実行できず、サブクエリはありません。 **数式ウィジェット**&#x200B;はそのギャップを埋めます。2 ～ 6個の名前付きクエリを実行し、その結果を`+ - * / ( )`と組み合わせます。

**例：ホストあたりのリクエスト数** （数式、行）

1. **数式**→ウィジェットを追加します。
2. 最初のクエリ `requests`と2番目のクエリ `hosts`に名前を付けます。
3. クエリと数式を入力し、**数式を実行**&#x200B;をクリックします。

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

**例：データベースに費やした時間の割合** （数式、数値、単位`%`）

クエリ `db`:

```sql
SELECT sum(duration) AS v FROM Span WHERE db.system != ''
```

クエリ `server`:

```sql
SELECT sum(duration) AS v FROM Transaction
```

計算式：

```text
(db / server) * 100
```

数式クエリのルール：

| ルール | 詳細 |
| --------------- | -------------------------------------------------------------------------------------------------------------------- |
| 氏名 | 文字、数字、および`_` （先頭は文字または`_`）、最大40文字、一意 |
| 式 | `+ - * / ( )`、数字、クエリ名。最大500文字 |
| 同一シェイプ | すべてのクエリはプレーン合計（数値）、**または**&#x200B;はすべて`TIMESERIES`で、**同じバケットサイズ** （行/領域/バー）です |
| 各1つの指標 | 各クエリは、集計を1つだけ返し、`FACET`は返しません |
| 環境にないデータ | 値が見つからないか、ゼロで除算すると、空のポイントが生成されますが、`Infinity`や`NaN`は生成されません |

>[!TIP]
>
> 時系列数式のすべてのクエリで`TIMESERIES AUTO`を使用します。 クエリは1つのウィンドウを共有するため、常に同じバケットサイズに解決されます。

### 変数を使用してダッシュボードを再利用可能にする

変数を使用すると、1つのダッシュボードで多くのサービス、階層、しきい値を提供できます。 各変数は、変数バー内のピッカーです。 ウィジェットのタイトルとテキストウィジェットで`{{name}}`として参照します（ウィジェット SQLで以下の制限が修正されると）。

**変数を作成** （_組織およびスーパー管理者_）:「**+変数**」をクリックします。

| フィールド | メモ |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **名前** | 文字、数字、および`_`。先頭は文字または`_`です。 一意である必要があります。 `{{name}}`として使用されます。 |
| **ラベル** | ピッカーの表示内容（オプション）。 |
| **種類** | **カスタムリスト** （行ごとに1つの値）、**クエリ**&#x200B;から（1つの`FACET`を持つSQL クエリ。その一意の値がオプションになる）、または&#x200B;**テキストボックス** （フリーテキスト）。 |
| **複数の値の選択を許可** | コンマ区切りのリストに展開します。 `IN (…)`と一緒に使用します。 |
| **値は数値です** | 引用符で囲まれていない値（`'500'`ではなく`500`）を挿入します。 |

**置換の仕組み**

| 選択範囲 | `{{svc}}`様は |
| ---------------------------------- | ------------------------------------------------------- |
| 単一の文字列`checkout` | `'checkout'` |
| 複数の値`checkout`、`cart` | `'checkout', 'cart'` |
| 数値`500` | `500` |
| `'`を含む値 | `''`としてエスケープ |
| 選択されていません | ウィジェットは待機し、_を表示します。{{svc}}_&#x200B;の値を選択してください |

```text
WHERE service = {{svc}}            single select
WHERE service IN ({{svc}})         multiple select
WHERE duration > {{slowMs}}        numeric text box
```

クエリ駆動変数の場合、**は常に`LIMIT`**&#x200B;を追加します。 ファセットクエリは、デフォルトでは上位10個の値のみを返します。 このリストには、最大200のサービスが表示されます。

```sql
SELECT count(*) FROM Transaction FACET service LIMIT 200
```

テナントまたはタイムピッカーが変更されたときにオプションが更新され、アルファベット順に並べ替えられます。 現在の選択が無効になると、最初の値（単一選択）またはすべての値（複数選択）が自動的に選択されます。 変数のクエリは、他の変数を参照できません。

> **警告**
>
> **現在のビルド制限。** ダッシュボードグリッドで代用が行われますが、**ウィジェットを追加** プレビューと&#x200B;**サーバーサイドの保存チェック**&#x200B;は未加工の`{{name}}` トークンを受け取り、`Unexpected character '{'`で却下します。 実際の内容：
>
> - **ウィジェットタイトル**&#x200B;と&#x200B;**テキストウィジェット**&#x200B;の`{{variables}}`がエンドツーエンドで動作します。
> - **ウィジェット SQL**&#x200B;内の`{{variables}}`は、**インポート**&#x200B;の後に正しくレンダリングされますが、そのようなウィジェットが存在する間、ダッシュボードを&#x200B;**保存**&#x200B;することはできません。
>
> これが修正されるまで、リテラルフィルター（例：`WHERE service = 'checkout'`）を使用してコンテキストごとのウィジェットを作成し、**複製**&#x200B;を使用して複製します。

### まとめる

[&#x200B; ダッシュボードのブループリント &#x200B;](../examples/dashboard-blueprint.md)で使用されているすべての構築ブロックが見つかります。しきい値、セクション、パーセンタイル、Apdex、レート、ケース列、ヒストグラム、数式、変数を含むKPI タイルです。 ブループリントでは、それらを組み合わせて、包括的なゴールデンシグナル、エラートリアージ、AEM、インフラストラクチャ、JVM ダッシュボードを作成する方法を示しています。
