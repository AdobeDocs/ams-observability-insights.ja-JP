---
title: パフォーマンス
description: Observability Insightsでクエリ、チャート、ダッシュボードのパフォーマンスを向上させます。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '227'
ht-degree: 3%
---

## パフォーマンス {#performance}

各クエリは、15秒のタイムアウトとバイト読み取り予算で実行されます。 このドキュメントでは、Observability Insightsが自動的にクエリを高速化する方法と、任意の時間範囲で高速に動作するダッシュボードを設計する方法について説明します。

### ロールアップアクセラレーション（自動）

`Transaction`個のグラフは、次の&#x200B;**すべて**&#x200B;が保持されている場合に事前集計されたテーブルを読み取ります：

| 条件 | 要件 |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------ |
| シェイプ | クエリに`TIMESERIES`があります（数値/テーブルの合計は常に正確さのために生で読み取られます） |
| ウィンドウ | ≥ 3時間（1分ティア） ・ ≥ 1 d （5分ティア） ・ ≥ 7 d （1時間ティア） |
| 集計 | のみ`count(*)`、`percentage(*, WHERE status = 'Error')`、`apdex(duration, 500)`、`percentile(duration, 50 \| 95 \| 99)` |
| `FACET` / `WHERE` | `service`、`appInstance`、`kind`のみ |

`CpuUtilization`と`MemoryUtilization` （`average(value)`）と`JvmCpuRecentUtilization` （`average(value)`）のグラフと`FACET host` / `WHERE host …`の≥3時間の読み取り1分のロールアップ。

7 ～ 31日間のトレンドウィジェットの&#x200B;**デザインのヒント：**。ロールアップに適したセット（p95、エラー%、Apdex @ 500、カウント、FACET サービス）に従ってください。任意の範囲で高速を維持します。 `FACET host`/`name`、`average(duration)`または`urlPath`個のフィルターを、短いウィンドウまたはテーブル用に保存します。

### 一般的なヒント

- すべてのファセット クエリで`LIMIT`を使用します。特に`FACET urlPath`、`traceId`、`jcr.path` （高い基数）を使用します。
- 低カーディナリティ属性（`service`、`kind`、`status`）の`WHERE`で早期にフィルタリングします。
- インバウンドリクエストのみが必要な場合は、`Span`よりも`Transaction`を優先します（行ははるかに少なくなります）。
- `uniqueCount`は正確です。長いウィンドウのカーディナリティ属性の値が高い場合は、最もコストの高い集計になります。
- 数分間のウィンドウを除いて`TIMESERIES MAX`を避けます。
- ダッシュボードウィジェットは30秒間キャッシュ結果を取得します。更新ボタンを押すと、再実行されます。
- 15秒のタイムアウトまたはバイト読み取りの予算エラーに達したウィジェットは、ウィンドウを絞り込んだり、フィルターを追加したり、ファセットを減らしたりします。
