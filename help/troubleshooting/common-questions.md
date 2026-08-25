---
title: FAQ
description: AEM Managed ServicesのObservability Insightsに関する一般的な質問と調査の出発点。
feature: Operations
role: Admin
source-git-commit: 68b80f99e8be9deed37ea857d1dc7cb0ba3ec94d
workflow-type: tm+mt
source-wordcount: '186'
ht-degree: 0%

---


# FAQ {#faq}

このページは、アクティブな調査中にどこから始めればよいかわからない場合や、迅速な回答が必要な場合の出発点として使用してください。

## Observability Insightsにアクセスできないのはなぜですか？ {#cannot-access-observability-insights}

[&#x200B; アクセスとアカウント管理](../get-started/access-and-accounts.md)から開始します。 プロビジョニングが不完全または古い場合は、カスタマーサクセスエンジニア（CSE）に連絡して、アクセスまたは更新をリクエストしてください。

## ログインしようとすると、「読み込み権限」が表示されるのはなぜですか？ {#loading-permissions-error}

これは通常、ユーザープロビジョニングに関する問題を示します。 カスタマーサクセスエンジニア（CSE）に連絡し、関連チームと協力してアクセスの問題を解決できるようにします。

## 問題がアプリケーションまたはインフラストラクチャに関連しているかどうかを判断するにはどうすればよいですか？ {#application-or-infrastructure}

作成者または公開時のリクエスト率、エラー率、待ち時間を確認するには、[&#x200B; アプリケーションパフォーマンス監視](/help/applications.md)から開始します。 アプリケーションのシグナルが高い場合は、[Hosts](/help/hosts.md)を使用して、ホストレベルのリソースのプレッシャー（CPU、メモリ、ディスク、またはネットワーク）が、表示される内容を説明または複合しているかどうかを確認します。

## Observability Insightsは実際にどのようなデータを収集しますか？ {#what-data-is-collected}

範囲、アプリケーション表現、保持期間、および運用上の影響の監視については、[&#x200B; カバレッジ、環境、およびデータ保持](../get-started/coverage-and-data.md)を参照してください。
