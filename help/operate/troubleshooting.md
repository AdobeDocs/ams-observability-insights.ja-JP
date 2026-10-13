---
source-git-commit: ec98f6bd132cadaba1da7c4fb727d7e031790645
workflow-type: tm+mt
source-wordcount: '327'
ht-degree: 6%
---
## トラブルシューティング {#troubleshooting}

クエリを&#x200B;**検証**&#x200B;または&#x200B;**実行**&#x200B;し、ダッシュボード上のウィジェット自体を実行すると、エラーメッセージがエディターの下に表示されます。 最初の列でメッセージを見つけます。

| メッセージ/症状 | 原因 | 修正 |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| `Unknown entity 'X'` | タイプミスまたはカタログ以外の名前 | [&#x200B; データディクショナリ &#x200B;](#data-dictionary) / クエリガイドパネルを確認します。 |
| `Attribute 'x' is not available on Entity` | そのエンティティに属していない属性（例：`CpuUtilization`の`duration`） | 属性と指標の属性をトレースします。[&#x200B; データディクショナリ &#x200B;](#data-dictionary)を参照してください。 |
| `Raw attributes cannot be mixed with aggregates; use FACET to group results` | `SELECT service, count(*)` | `SELECT count(*) … FACET service`. |
| `FACET and TIMESERIES require an aggregate query` | FACET/TIMESERIESによるイベントクエリ | 集計を使用します。 |
| `FACET supports at most five attributes` | > 5 ファセット | 削減。 |
| `average requires a numeric attribute` | e.g. `average(service)` | `uniqueCount`/`latest`または数値属性を使用してください。 |
| `rate requires TIMESERIES` / `rate is only available for counter measures` | `rate()`は時系列なし、またはゲージに含まれていません | `TIMESERIES AUTO`を追加します。✔ カウンターエンティティ （[&#x200B; キュレートされた指標エンティティ &#x200B;](#53-curated-metric-entities)）を使用します。ゲージには`derivative()`を使用します。 |
| `rate cannot be combined with other aggregates` | `rate()`を持つ複数の項目 | ウィジェットごとに1つの`rate()`。数式と組み合わせる。 |
| `percentOfTotal cannot be combined with TIMESERIES` | — | 時系列を削除します。 |
| `apdex requires an entity with a status attribute` | 指標エンティティの`apdex` | `Transaction`/`Span`/`AppRequest`を使用します。 |
| `ORDER BY 'x' must reference a selected value or alias` | 選択されていない項目による注文 | エイリアスで集計と順序を指定します。 |
| `IN requires between 1 and 100 values` | — | `LIKE`を分割または使用します。 |
| `Use IS NULL or IS NOT NULL for null comparisons` | `attr = NULL` | `IS NULL` の使用 |
| `Unexpected character '{'` | 追加ウィジェットのプレビューまたは保存時の`{{variable}}` | [&#x200B; チュートリアル 3、ステップ 8.7](#87-make-dashboards-reusable-with-variables)の制限を参照してください。 |
| `Unexpected character ';'` / `'"'` | セミコロンまたは二重引用符 | `;`を削除します。一重引用符を使用します。 |
| `Expected a positive duration such as 30 minutes` | `TIMESERIES 5m` | `TIMESERIES 5 minutes`を書き込みます。 |
| `Time range cannot exceed 31 days` | ピッカー> 31 d | ウィンドウを絞り込みます。 |
| `Select a customer before running a SQL query` | テナントなし/デモテナント | テナントスイッチャーで顧客を選択します。 |
| `All queries in a formula must be the same kind…` | 合計と時系列の組み合わせ、またはファセット | すべてのサブクエリをプレーンまたは全`TIMESERIES AUTO`にします。ファセットを削除します。 |
| `Query 'x' must select exactly one metric…` | サブクエリには2つの集計があります | サブクエリごとに1つの集計。 |
| `Unknown query name 'x' - must be one of: …` | 数式は見つからない名前を参照します | 名前を正確に一致させます（大文字と小文字を区別します）。 |
| **保存に失敗しました** | ウィジェットクエリがサーバーサイドで無効です（多くの場合`{{var}}`） | 最近編集した各ウィジェットを開き、**実行**&#x200B;します。無効なウィジェットを修正または削除します。 |
| _{{x}}_&#x200B;の値を選択 | 変数に選択範囲がありません | 変数バーで値を選択します。 |
| _前回のアクティビティはありません…_ | ウィンドウに行がありません | ウィンドウを広げます。検出クエリ （[&#x200B; データ検出](#101-data-discovery)）を使用してフィルターを確認します。 |
| 線がドットに砕ける | データケイデンスよりも小さいバケット | `TIMESERIES AUTO` （≥ 2分）または棒グラフを使用します。 |
| ファセットに空の文字列グループが表示される | 一部の行に属性がありません | 次の `WHERE attr != ''`.を追加します。 |
| _この従来のSQL パネルは実行されていません_ | 古いraw-SQL ビルダーのパネル | SQLで編集→再構築。 |
