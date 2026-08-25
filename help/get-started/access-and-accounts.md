---
title: アクセスとアカウント管理
description: Observability Insights アカウントのプロビジョニング方法、アクセスの管理者、顧客チームの管理レベルについて説明します。
feature: Operations
role: Admin
source-git-commit: 6526a90a017147ac3483c0b2b626b9aa903819ba
workflow-type: tm+mt
source-wordcount: '232'
ht-degree: 0%

---


# アクセスとアカウント管理 {#access-and-account-management}

Adobeは、AEM Managed Services組織のObservability Insights アカウントをプロビジョニングおよび管理します。 カスタマーチームは、Adobe IMS機能を利用してログインし、監視対象データの読み取り専用の可視性を利用できます。

## アカウント所有モデル {#account-ownership-model}

- Adobe Managed Servicesは、Observability Insights アカウントを所有しています。
- 顧客チームには読み取り専用アクセスが許可されます。
- 管理の変更、プロビジョニング、アクセスの更新は、Adobeを通じて管理されます。

## ユーザーがアクセスする方法 {#how-users-get-access}

Observability InsightsへのアクセスにはAdobe IMSプロビジョニングが必要です。

アクセスをリクエストまたは更新するには：

1. カスタマーサクセスエンジニア（CSE）にお問い合わせください。
2. Adobe IMSのプロビジョニングに必要なユーザーの詳細を指定します。
3. 正しい組織とアクセス範囲が割り当てられていることを確認します。

プロビジョニングが完了したら、[insights.adobecqms.net](https://insights.adobecqms.net)でログインします。

## ユーザーの機能 {#what-users-can-do}

顧客ユーザーは通常、次のことが可能です。

- APM ダッシュボードを表示
- インフラダッシュボードを見る
- 監視対象アプリケーションとホストの指標を検査
- 共有ダッシュボードとトレースを使用して調査に参加する

## ユーザーができないこと {#what-users-cannot-do}

お客様は、Adobeが明示的に文書化しない限り、Adobe Managed Servicesでは管理制御が維持されることを想定する必要があります。

一般的な例には、次のようなものがあります。

- アカウントの所有権の管理
- プラットフォームレベルのプロビジョニングの変更
- 管理されたインストルメンテーション動作の変更

## 後で追加する情報 {#information-to-add-later}

この節は、完全なプロセスの詳細が使用可能になった場合に使用します。

- Adobe IMSの前提条件
- プロビジョニングのターンアラウンドタイムの予測
- 連絡先とパスのエスカレーション
- ジョイナーとリーバーのユーザーライフサイクル管理
