---
source-git-commit: ec98f6bd132cadaba1da7c4fb727d7e031790645
workflow-type: tm+mt
source-wordcount: '134'
ht-degree: 7%
---
## 付録 {#appendix}

**このドキュメントでは、** [14.1の制限](#141-limits) ・ [14.2書き出しファイル形式](#142-export-file-format) ・ [14.3 ワンページチートシート ](#143-one-page-cheat-sheet)

### 制限

| 項目 | 制限 |
| -------------------------------- | ----------------------------------------------------------------------- |
| クエリテキスト | 4 KB |
| `FACET` 属性 | 5 |
| `LIMIT` | 1 - 5 000 （デフォルト：数値1、ファセット 10、イベント 100、時系列5 000） |
| `IN (…)`個の値 | 1 - 100 |
| 時間ウィンドウ | 31日間 |
| クエリのタイムアウト | 15秒（さらにバイト読み取り予算） |
| 最小バケット：`TIMESERIES AUTO` | 2 分 |
| 数式 | 2 ～ 6個のクエリ、式≤500文字、名前≤40文字 |
| ダッシュボード | ≤ 40個のタブ ・ ≤ 200個のウィジェット/タブ ・ ≤ 50個の変数 |
| ウィジェット | title ≤ 200 ・ subtitle ≤ 500 ・ unit ≤ 40 ・ ≤ 10 threshold |
| テキストウィジェット | ≤ 4,000文字 |
| 変数 | 名前≤ 64文字・ ≤ 500個のオプション/値・ クエリ ≤ 4 KB |

### 書き出しファイル形式

**書き出し**&#x200B;を使用してこれを作成し、**読み込み**&#x200B;を使用して読み込みます。 ダッシュボードのバージョン管理や、テナント間でのプロモーションに役立ちます。

```json
{
  "name": "Golden Signals",
  "templateVariables": [
    {
      "id": "var-1",
      "name": "svc",
      "label": "Service",
      "kind": "query",
      "multi": true,
      "query": "SELECT count(*) FROM Transaction FACET service LIMIT 200",
      "value": null
    }
  ],
  "pages": [
    {
      "id": "page-1",
      "name": "Overview",
      "widgets": [
        {
          "id": "sec-kpis",
          "title": "SLO KPIs",
          "queryLanguage": "text",
          "text": "",
          "isSection": true,
          "collapsed": false,
          "layout": { "x": 0, "y": 0, "w": 12, "h": 1 }
        },
        {
          "id": "w-error-rate",
          "title": "Error %",
          "queryLanguage": "SQL",
          "query": "SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction",
          "vizType": "stat",
          "unit": "%",
          "thresholds": [
            { "value": 0, "status": "ok" },
            { "value": 1, "status": "warning" },
            { "value": 5, "status": "error" }
          ],
          "layout": { "x": 0, "y": 1, "w": 3, "h": 2 }
        },
        {
          "id": "w-throughput",
          "title": "Throughput by service",
          "queryLanguage": "SQL",
          "query": "SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8",
          "vizType": "area",
          "layout": { "x": 3, "y": 1, "w": 9, "h": 4 }
        },
        {
          "id": "w-availability",
          "title": "Availability %",
          "queryLanguage": "SQL-formula",
          "formulaQueries": [
            {
              "name": "ok",
              "query": "SELECT count(*) AS v FROM Transaction WHERE status != 'Error'"
            },
            {
              "name": "total",
              "query": "SELECT count(*) AS v FROM Transaction"
            }
          ],
          "formulaExpression": "(ok / total) * 100",
          "vizType": "stat",
          "unit": "%",
          "layout": { "x": 0, "y": 3, "w": 3, "h": 2 }
        }
      ]
    }
  ],
  "exportedAt": "2026-10-01T00:00:00.000Z"
}
```

`vizType`値：`timeseries` （行）、`area`、`bar`、`pie`、`stat` （番号）、`table`。 しきい値`status`: `ok` （緑）、`warning` （琥珀）、`error` （赤）。

### ワンページカンニングペーパー

```text
SHAPE        SELECT agg FROM Entity                         -> Number
             ... FACET a[, b..e] LIMIT n                     -> Bar / Pie / Table
             ... TIMESERIES AUTO                             -> Line / Area
             ... FACET a TIMESERIES AUTO LIMIT n             -> top-n series
             SELECT attr1, attr2 FROM Entity LIMIT n         -> event Table

AGGREGATES   count(*)  average(x)  sum(x)  min(x)  max(x)  uniqueCount(x)
             percentile(x, 95)  stddev(x)  latest(x[, rank])  earliest(x[, rank])
             percentage(*, WHERE c)  apdex(duration, 500)  percentOfTotal(x)
             rate(value)        -- counters, TIMESERIES only
             derivative(value)  -- gauges,   TIMESERIES only
             agg FILTER (WHERE c) AS alias

WHERE        = != < <= > >=   LIKE / NOT LIKE ('%', '_')   IN / NOT IN (...)
             IS [NOT] NULL   AND OR NOT ( )   strings in 'single quotes'
             string attr present?  ->  attr != ''

ORDER/LIMIT  ORDER BY alias [ASC|DESC]   LIMIT 1..5000

VARIABLES    = {{name}}   IN ({{name}})   numeric: > {{name}}

FORMULA      2-6 named queries, one metric each, no FACET, same shape
             expression: + - * / ( ) numbers names
```
