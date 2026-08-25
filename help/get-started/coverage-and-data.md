---
title: カバレッジ、環境、データ保持
description: AEM Managed ServicesのObservability Insights モニター、アプリケーションの表現方法、モニタリングデータの保持期間を確認します。
feature: Operations
role: Admin
source-git-commit: 1d54a6a398360b040221db5b2780d301722894bf
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
