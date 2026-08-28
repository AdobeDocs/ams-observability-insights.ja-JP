---
title: Observability InsightsによるAEM Managed Services環境のモニタリング
description: AEM Managed ServicesのObservability Insightsの内容、対象、このガイドの残りの部分を操作する方法を理解するには、ここから始めてください。
feature: Operations
role: Admin
source-git-commit: 90ca53475d23dd9b3100236d899d3941f717edbd
workflow-type: tm+mt
source-wordcount: '744'
ht-degree: 0%

---


# Observability InsightsによるAEM Managed Services環境のモニタリング {#observability-insights-monitoring}

**オブザーバビリティのインサイト**&#x200B;は、個別のモニタリングプラットフォームを必要とせずに、AEM Managed Servicesのアプリケーションのパフォーマンス、インフラストラクチャの健全性、サービス動作を可視化します。

サービスの信頼性、インシデント対応、パフォーマンス分析を担当している場合は、**Observability Insights**&#x200B;を使用すると、症状から証拠にすばやく移行できます。 アプリケーションのテレメトリとホストレベルのヘルスシグナルを組み合わせることで、カスタマーチームとAdobeは、共通の運用上の視点から問題を調査することができます。

## ホワイトペーパーのObservability Insights

<iframe
  src="v2-assets/Observability_Insights_Overview.pdf"
  title="ホワイトペーパーのObservability Insights"
  width="100%"
  height="800"
  style="border: 0;"
></iframe>

[ホワイトペーパーをダウンロード](v2-assets/Observability_Insights_Overview.pdf)

## チームがObservability Insightsを利用する理由？ {#why-teams-use-observability-insights}

オブザーバビリティのインサイトを活用して、次のような運用上の質問に回答します。

- 問題は作成者、公開、またはその両方に影響を与えますか？
- 問題は、アプリケーションの動作、ホストリソースのプレッシャー、またはその両方の組み合わせによって引き起こされますか？
- どのトランザクション、エンドポイント、またはステータスグループがエラーまたは遅延の急増を説明しますか？
- 問題は1つの環境に分離されるか、より広範なトポロジ全体で表示されるか。

Observability Insightsは、最近の行動の運用分析のために設計されています。 変更された内容、変更された箇所、エスカレーションや是正措置の前に最も関連性の高いシグナルを特定するのに役立ちます。

## どのようなオブザーバビリティのインサイトが役立ちますか？ {#what-observability-insights-helps-you-do}

オブザーバビリティのインサイトを使用して、次のことを行います。

- オーサー層とパブリッシュ層が実際のトラフィックでどのように動作するかを理解します。
- アプリケーションの遅延、エラー率、JVMの正常性をホストレベルのシグナルに関連付けます。
- 問題が1つの環境、1つの階層、または1つのホストに分離されているかどうかを確認します。
- Adobe Managed Servicesを利用すれば、社内の各部門の状況を調査中に把握できます。

Observability Insightsは、AEM Managed Servicesに含まれています。 Adobeは、アカウントをプロビジョニングおよび管理し、サポートされている環境を測定し、結果のダッシュボードを読み取り専用の運用ツールとしてチームに公開します。

Adobeでプラットフォームの設定と実装を管理することで、エージェントのデプロイメント、アカウント管理、ダッシュボードの組み立てではなく、調査と解釈に集中できます。

## 一目で {#at-a-glance}

AEM Managed Servicesの一部として、以下を受け取ります。

- **専用のObservability Insights アカウント** — Adobe Managed Servicesによってプロビジョニングおよび管理され、チームには読み取り専用のアクセス権が付与されます。
- **Deep AEM transaction monitoring** — Observability Insights APM エージェントは、メソッド呼び出し（行番号を含む）、外部依存関係、リポジトリ操作まで、有意義なトランザクションを追跡します。
- **統合アプリケーションとホスト ビュー** — アプリケーションとホストレベルの指標を組み合わせて、パフォーマンスを包括的に最適化します。

## このドキュメントの対象 {#who-this-documentation-is-for}

このドキュメントは、主に次の目的で作成されています。

- 監視対象環境の可視化が必要なAEM Managed Services管理者
- インシデント、傾向分析、サービスレビューを処理するオペレーションおよびサポートチーム
- 調査中にAdobe Managed Servicesと提携したカスタマーエンジニアリングチーム
- モニタリングの範囲と業務上の責任を理解する必要がある関係者

## AdobeのObservability Insightsによるモニタリング {#what-we-monitor}

Adobeは、Observability Insights APM Java プラグインを使用して、AEM **Author**&#x200B;および&#x200B;**Publish**&#x200B;層を監視します。 トポロジ内のすべてのホストされているサーバーは、Observability Insights Infrastructure エージェントで監視されます。 カスタム APMおよびインフラストラクチャのモニタリングは、実稼動以外の環境と実稼動環境の両方で有効になります。

![AEM オーサーサーバー、パブリッシュサーバー、およびホストされたサーバーをまたいだObservability Insights APMとインフラストラクチャのモニタリングを示す図](v2-assets/login-screen.png)

### アカウント内のアプリケーション {#applications-in-your-account}

Observability Insights アカウントは、1つのAdobe マスターアカウントにリンクされ、以下を含む複数のアプリケーションからデータを受け取ることができます。

- AEM Managed Services環境ごとに&#x200B;**Author**&#x200B;層の1つのAPM アプリケーション
- AEM Managed Services環境ごとに&#x200B;**パブリッシュ**&#x200B;層のAPM アプリケーションを1つ作成する

各アプリケーションには独自のライセンスキーがあります。 Managed Services コントラクトレポートのすべてのトポロジを1つのObservability Insights アカウントに保存します。 APMおよびインフラストラクチャの指標とイベントは、最大&#x200B;**30日間**&#x200B;保持されます。

## アカウントにアクセス {#access}

モニタリングデータは、Adobeがプロビジョニングおよび管理するObservability Insights アカウントに統合されます。 お客様のユーザーは、エージェントによって収集されたAPMおよびインフラストラクチャ データに対する&#x200B;**読み取り専用アクセス**&#x200B;を受け取ります。 Adobe Managed Servicesでは、アカウントの所有権と管理者権限が保持されます。

### 前提条件 {#access-prerequisites}

ログインする前に、次の点を確認してください。

- お客様の組織は、アクティブな&#x200B;**AEM Managed Services** サブスクリプションを持っています。 Observability Insightsは追加費用なしで含まれています。
- カスタマーサクセスエンジニア（CSE）がAdobe IMSアカウントをプロビジョニングし、お客様の組織のObservability Insights アカウントへのアクセス権を付与しました。

>[!NOTE]
>
> **Observability Insightsへのアクセス：** アクセスを取得するには、Adobe IMSプロビジョニングが必要です。 組織のユーザーアクセスをプロビジョニングおよび管理するには、カスタマーサクセスエンジニア（CSE）にお問い合わせください。

CSEがアカウントをプロビジョニングしたら、[insights.adobecqms.net](https://insights.adobecqms.net)でログインします。 このURLは、すべてのAEM Managed Servicesのお客様に対して同じです。組織の環境とダッシュボードは、プロビジョニングされたアカウントに対してスコープされます。
