---
title: カバレッジ、環境、データ保持
description: AEM Managed ServicesのObservability Insights モニター、アプリケーションの表現方法、モニタリングデータの保持期間を確認します。
feature: Operations
role: Admin
product_v2:
  - id: fd1f54a9-f50c-467d-8956-cebbaf4f3eb8
    internal-label: Experience Manager
feature_v2:
  - id: 8a70d214-ab7b-58c1-b001-2ed2e5d6303d
    internal-label: Operations
role_v2:
  - id: c66ffd68-0f65-42bb-aa23-b4020f12e0bd
    internal-label: Admin
source-git-commit: e0cc17c9d725cad021ba99da4332bca176eae6db
workflow-type: tm+mt
source-wordcount: '267'
ht-degree: 2%
---

# カバレッジ、環境、データ保持 {#coverage-environments-and-data-retention}

このページでは、AEM Managed ServicesのObservability Insightsで収集されるデータと、そのデータの整理方法について説明します。

## カバーしている部分の監視 {#monitoring-coverage}

Adobeは以下を監視します。

- Observability Insights APM Java プラグインを使用したAEM オーサー層
- Observability Insights APM Java プラグインを使用したAEM パブリッシュ層
- Observability Insights Infrastructure エージェントを使用して管理トポロジ内のホストされたサーバー

カスタム APMおよびインフラストラクチャのモニタリングは、Managed Servicesの非実稼動環境と実稼動環境の両方で可能です。

## アプリケーションの表現方法 {#how-applications-are-represented}

通常、AEM Managed Servicesの各環境には次のものが含まれます。

- オーサー用の1つのAPM アプリケーション
- パブリッシュ用の1つのAPM アプリケーション

Managed Services コントラクトレポートのすべてのトポロジを1つのObservability Insights アカウントに保存します。

## データ保持 {#data-retention}

APM指標、インフラストラクチャ指標、および関連イベントは、最大&#x200B;**30日間**&#x200B;保持されます。

## 概要テーブル {#summary-tables}

| カバレッジエリア | 監視するもの |
| -------------- | ------------------------------------------ |
| APM | AEM オーサーアプリケーションとパブリッシュアプリケーション |
| インフラストラクチャ | 管理対象トポロジ内のすべてのホスト サーバー |

| 項目 | 表現 |
| ------------------------------ | ------------------------------------------------------------- |
| AEM 環境 | 1つのオーサーAPM アプリケーションと1つの公開APM アプリケーション |
| Observability Insights アカウント | Managed Servicesのカスタマースコープごとに、Adobeで管理するアカウントを1つずつ割り当てます |

| データタイプ | 定着 |
| --------------------------------- | ------------- |
| APM指標とイベント | 最大30日間 |
| インフラ指標とイベント | 最大30日間 |

## 運用上の意味 {#what-this-means-operationally}

- Observability Insightsは、運用分析、アクティブなインシデント、最近のトレンド比較に適しています。
- リテンションウィンドウを超えた履歴分析は、必要に応じて、他のレポートまたはアーカイブプロセスを通じて処理する必要があります。
- 繰り返し発生する問題を調査する場合は、データが消える前にスクリーンショットを撮影するか、証拠を書き出します。
