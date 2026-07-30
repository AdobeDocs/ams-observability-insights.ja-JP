---
title: SynoptryxによるAEM Managed Services環境のモニタリング
description: Adobe Experience Manager Managed ServicesでのSynoptryx モニタリングの概要（Adobeのモニタリング内容、アカウントの設定方法、アクセス方法）。
feature: Operations
role: Admin
source-git-commit: f937aa4e3cebd1aae6945a35a77154add5db980c
workflow-type: tm+mt
source-wordcount: '651'
ht-degree: 0%

---


# SynoptryxによるAEM Managed Services環境のモニタリング {#synoptryx-monitoring}

Synoptryxは、別の監視プラットフォームをセットアップすることなく、アプリケーションのパフォーマンス、インフラストラクチャの健全性、エンドユーザーのエクスペリエンスを可視化します。

>[!NOTE]
>
> Synoptryxの製品概要ホワイトペーパーは、AEM Managed Servicesの完全なオブザーバビリティとモニタリングの概要に対応しており、関係者との共有やオフラインでのレビューに最適です。

## 概要 {#overview}

Synoptryxは、アプリケーションのパフォーマンス、インフラストラクチャの健全性、合成モニタリング全体にわたって統一された可視性を提供するように設計された、Adobeの次世代オブザーバビリティ・プラットフォームです。 統合された単一のエクスペリエンスを通じて、重要なビジネスサービスを先見的に監視できます。 Synoptryxは、Application Performance Monitoring （APM）、Infrastructure Monitoring、Synthetic Userジャーニーモニタリングを組み合わせて、問題がエンドユーザーに影響を与える前に特定し、解決するのに役立ちます。 このプラットフォームは、詳細なトランザクショントレース、JVM インサイト、インフラストラクチャのテレメトリ、高度な診断を提供し、根本原因の分析を迅速化します。 最新のオブザーバビリティ技術を基盤とし、複雑なエンタープライズ環境をまたいで、拡張性と安全性に優れた監視を実現します。 Synoptryxは、優れた運用をサポートするために、拡張データ保持、豊富なダッシュボード、インテリジェント分析を提供しています。 Adobe IMSのシームレスなログインエクスペリエンスにより、安全なアクセスとガバナンスが保証されます。 このプラットフォームは、サービスの信頼性を向上し、トラブルシューティングを加速し、顧客体験を向上させるように設計されています。 Adobeの戦略的オブザーバビリティ ソリューションであるSynoptryxは、マネージドサービス環境全体でモニタリング、オートメーション、運用インサイトを提供する、将来を見据えた基盤を提供します。

SynoptryxはAdobe Experience Manager Managed Servicesに含まれているため、個別のモニタリングプラットフォームやライセンスは必要ありません。 Synoptryxは、Adobe（AEM）アプリケーションとサポートインフラストラクチャのパフォーマンスを把握するための専用の基盤です。

このガイドでは、監視される内容、Synoptryx アカウントの設定方法、日々の分析やトラブルシューティングに使用するダッシュボードの操作方法について説明します。

## 一目で {#at-a-glance}

AEM Managed Servicesの一部として、以下を受け取ります。

- **専用のSynoptryx アカウント** — Adobe Managed Servicesによってプロビジョニングおよび管理され、チームには読み取り専用のアクセス権が付与されます。
- **Deep AEM transaction monitoring** — Synoptryx APM エージェントは、メソッド呼び出し（行番号を含む）、外部依存関係、リポジトリ操作まで、意味のあるトランザクションを追跡します。
- **統合アプリケーションとインフラストラクチャ ビュー** — APMとホストレベルの指標を組み合わせて、パフォーマンスを包括的に最適化します。

## SynoptryxでAdobeが監視するもの {#what-we-monitor}

Adobeは、Synoptryx APM Java プラグインを使用して、AEM **Author**&#x200B;および&#x200B;**Publish**&#x200B;層を監視します。 トポロジ内のすべてのホストされているサーバーは、Synoptryx Infrastructure エージェントで監視されます。 カスタム APMおよびインフラストラクチャのモニタリングは、実稼動以外の環境と実稼動環境の両方で有効になります。

![Synoptryx APMとインフラストラクチャのモニタリングを示す図（AEM オーサーサーバー、パブリッシュサーバー、およびホストされているサーバー） &#x200B;](assets/image6.png)

### アカウント内のアプリケーション {#applications-in-your-account}

Synoptryx アカウントは、1つのAdobe マスターアカウントにリンクされ、以下を含む複数のアプリケーションからデータを受け取ることができます。

- AEM Managed Services環境ごとに&#x200B;**Author**&#x200B;層の1つのAPM アプリケーション
- AEM Managed Services環境ごとに&#x200B;**パブリッシュ**&#x200B;層のAPM アプリケーションを1つ作成する

各アプリケーションには独自のライセンスキーがあります。 Managed Services コントラクトレポートのすべてのトポロジを1つのSynoptryx アカウントに統合します。 APMおよびインフラストラクチャの指標とイベントは、最大&#x200B;**30日間**&#x200B;保持されます。

## アクセスとアカウント {#access}

モニタリング データは、Adobeがプロビジョニングおよび管理するSynoptryx アカウントに統合されます。 エージェントが収集したすべてのAPMおよびインフラストラクチャ指標に対する&#x200B;**完全な読み取り専用アクセス**&#x200B;をチームが受け取ります。 Adobe Managed Servicesでは、アカウントの所有権と管理権限が保持されます。

>[!NOTE]
>
> **Synoptryxへのアクセス：** アクセスを取得するには、Adobe IMSプロビジョニングが必要です。 カスタマーサクセスエンジニア（CSE）は、組織のユーザーアクセスをプロビジョニングおよび管理できます。

CSEがアカウントをプロビジョニングしたら、[synoptryx.adobecqms.net](https://synoptryx.adobecqms.net)でログインできます。

## 次の手順 {#whats-next}

チームが日常的に使用するモニタリングダッシュボードで作業を続けます。

- [Application performance monitoring （APM） &#x200B;](application-performance-monitoring.md) — AEM トランザクションをトレースし、JVMの動作を分析し、外部サービスを調べます。
- [&#x200B; インフラストラクチャの監視](infrastructure-monitoring.md) — ホストレベルのシステム、ネットワーク、プロセス、およびストレージの指標を確認します。
