---
title: AEM Managed Services環境を [!DNL Synoptryx]でモニタリングします
description: Adobeでの [!DNL Synoptryx] 監視 [!DNL Experience Manager] Managed Servicesの概要：Adobeの監視内容、アカウントの設定方法、およびチームのアクセス方法。
feature: Operations
role: Admin
source-git-commit: e8de2213d91e09da68a8f7014b075f81bd7f07ef
workflow-type: tm+mt
source-wordcount: '618'
ht-degree: 0%

---


# [!DNL Synoptryx]を使用してAEM Managed Services環境をモニタリングする {#synoptryx-monitoring}

[!DNL Synoptryx]は、別の監視プラットフォームを設定することなく、アプリケーションのパフォーマンス、インフラストラクチャの健全性、エンドユーザーのエクスペリエンスを可視化します。

>[!NOTE]
>
> [!DNL Synoptryx]製品概要ホワイトペーパーは、AEM Managed Servicesの観測可能性とモニタリングの全体像に使用できます。関係者との共有やオフラインでのレビューに最適です。

## 概要 {#overview}

[!DNL Synoptryx]は、アプリケーションのパフォーマンス、インフラストラクチャの健全性、合成モニタリング全体にわたって統一された可視性を提供するように設計された、Adobeの次世代オブザーバビリティ プラットフォームです。 統合された単一のエクスペリエンスを通じて、重要なビジネスサービスを先見的に監視できます。 [!DNL Synoptryx]は、Application Performance Monitoring （APM）、Infrastructure Monitoring、合成ユーザージャーニーモニタリングを組み合わせて、問題がエンドユーザーに影響を与える前に特定し、解決するのに役立ちます。 このプラットフォームは、詳細なトランザクショントレース、JVM インサイト、インフラストラクチャのテレメトリ、高度な診断を提供し、根本原因の分析を迅速化します。 最新のオブザーバビリティ技術を基盤とし、複雑なエンタープライズ環境をまたいで、拡張性と安全性に優れた監視を実現します。 [!DNL Synoptryx]では、優れた運用をサポートするために、拡張データ保持、豊富なダッシュボード、インテリジェント分析を提供しています。 [!DNL Adobe IMS]とのシームレスなログイン エクスペリエンスにより、安全なアクセスとガバナンスが保証されます。 このプラットフォームは、サービスの信頼性を向上し、トラブルシューティングを加速し、顧客体験を向上させるように設計されています。 Adobeの戦略的オブザーバビリティ ソリューションである[!DNL Synoptryx]は、マネージド サービス環境全体のモニタリング、オートメーション、運用インサイトの将来を見据えた基盤を提供します。

[!DNL Synoptryx]はAdobe [!DNL Experience Manager] Managed Servicesに含まれています。個別のモニタリングプラットフォームやライセンスは必要ありません。 Adobeは、標準の機能の一部として環境の可用性とパフォーマンスを監視します。また、[!DNL Synoptryx]は、Adobe [!DNL Experience Manager] （AEM）アプリケーションとサポートインフラストラクチャのパフォーマンスを把握するためにチームが使用できる専用プラットフォームです。

このガイドでは、監視される内容、[!DNL Synoptryx] アカウントの設定方法、日々の分析やトラブルシューティングに使用するダッシュボードの操作方法について説明します。

## 一目で {#at-a-glance}

AEM Managed Servicesの一部として、以下を受け取ります。

- **専用[!DNL Synoptryx] アカウント** — Adobe Managed Servicesによってプロビジョニングおよび管理され、チームには読み取り専用のアクセス権が付与されます。
- **Deep AEM transaction monitoring** — [!DNL Synoptryx] APM エージェントは、メソッド呼び出し（行番号を含む）、外部依存関係、リポジトリ操作まで、意味のあるトランザクションを追跡します。
- **統合アプリケーションとインフラストラクチャ ビュー** — APMとホストレベルの指標を組み合わせて、パフォーマンスを包括的に最適化します。

## Adobeが[!DNL Synoptryx]で監視するもの {#what-we-monitor}

Adobeは、[!DNL Synoptryx]APM Java プラグインを使用して、AEM **Author**&#x200B;および&#x200B;**Publish**&#x200B;層を監視します。 トポロジ内のすべてのホスト サーバーは、[!DNL Synoptryx] Infrastructure エージェントで監視されます。 カスタム APMおよびインフラストラクチャのモニタリングは、実稼動以外の環境と実稼動環境の両方で有効になります。

![Synoptryx APMとインフラストラクチャのモニタリングを示す図（AEM オーサーサーバー、パブリッシュサーバー、およびホストされているサーバー） &#x200B;](assets/image6.png)

### アカウント内のアプリケーション {#applications-in-your-account}

お客様の[!DNL Synoptryx] アカウントは、1つのAdobe マスターアカウントにリンクされており、以下を含む複数のアプリケーションからデータを受け取ることができます。

- AEM Managed Services環境ごとに&#x200B;**Author**&#x200B;層の1つのAPM アプリケーション
- AEM Managed Services環境ごとに&#x200B;**パブリッシュ**&#x200B;層のAPM アプリケーションを1つ作成する

各アプリケーションには独自のライセンスキーがあります。 Managed Services コントラクト レポートのすべてのトポロジを1つの[!DNL Synoptryx] アカウントに統合します。 APMおよびインフラストラクチャの指標とイベントは、最大&#x200B;**30日間**&#x200B;保持されます。

## アクセスとアカウント {#access}

監視データは、Adobeがプロビジョニングおよび管理する[!DNL Synoptryx] アカウントに統合されます。 エージェントが収集したすべてのAPMおよびインフラストラクチャ指標に対する&#x200B;**完全な読み取り専用アクセス**&#x200B;をチームが受け取ります。 Adobe Managed Servicesでは、アカウントの所有権と管理権限が保持されます。

>[!NOTE]
>
> **アクセス権：** [!DNL Synoptryx]へのアクセスには[!DNL Adobe IMS]のプロビジョニングが必要です。 カスタマーサクセスエンジニア（CSE）は、組織のユーザーアクセスをプロビジョニングおよび管理できます。

CSEがアカウントをプロビジョニングしたら、[synoptryx.adobecqms.net](https://synoptryx.adobecqms.net)でログインできます。

## 次の手順 {#whats-next}

チームが日常的に使用するモニタリングダッシュボードで作業を続けます。

- [Application performance monitoring （APM） &#x200B;](application-performance-monitoring.md) — AEM トランザクションをトレースし、JVMの動作を分析し、外部サービスを調べます。
- [&#x200B; インフラストラクチャの監視](infrastructure-monitoring.md) — ホストレベルのシステム、ネットワーク、プロセス、およびストレージの指標を確認します。

