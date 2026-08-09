---
title: "【AWS】ノンインフラ出身者がインフラの深淵に挑む：今週の地雷と復盤（FinOps & DB Migration編）"
emoji: "🚀"
type: "tech"
topics: ["aws", "infrastructure", "devops", "saa"]
published: true
---

# インフラ初心者が今週踏み抜いたAWSの地雷まとめ

今週、AWS SAAの設計シナリオに挑戦する中で、実務でも致命傷になりかねない重要な仕様の勘違い（地雷）を踏み抜きました。エンジニアとしての「振り返り」を兼ねて、アーキテクチャの解説と共に資産としてここに記録します。今回はクラウド財務（FinOps）と、レガシーDBのモダナイゼーションにおける罠です。

---

## 🚨 地雷ログ：コンピュートの世代交代と割引モデルの境界

### 1. 遭遇したシナリオと私の誤認
- **要件:** 現在はEC2で24時間365日稼働しているが、1年〜1年半後にはコンテナ化してAWS Fargateへ完全移行する計画がある。この3年間のコンピュートコストを最大化しつつ、移行の柔軟性も維持したい。
- **私のミス:** 「柔軟に変更できる割引」というキーワードに釣られ、**EC2 Convertible Reserved Instances (RI)** を選択してしまいました。

### 2. クラウド設計における正解と選定理由
- **最適解:** **Compute Savings Plans (3-year term)**
- **アーキテク放談:** Convertible RIは確かに柔軟ですが、その柔軟性は「EC2の枠内（インスタンスファミリーやOSの変更）」に限定されます。EC2からFargate、あるいはAWS Lambdaといった「異なるコンピューティングプラットフォーム」を跨ぐことはできません。将来的なサーバーレス/コンテナへのアーキテクチャ刷新（モダナイゼーション）を見据える場合、インフラの形態に依存せず割引を適用できる「Compute Savings Plans」が、現代のクラウドネイティブな財務戦略（FinOps）における絶対解です。

### 🗺️ データフロー（垂直表示）
[3-Year Commitment]
     ↓
[Compute Savings Plan]
     ↓
[Year 1: Amazon EC2]
     ↓ (Rewrite App)
[Year 2: AWS Fargate]

### 🎯 脳内に刻む教訓
> 💡 `[ERR-Cost/SavingsPlans-vs-RI]` EC2からFargate/Lambdaへの「クロスプラットフォーム移行」が確定している場合、旧時代のRIは捨て、Compute Savings Plansを無条件で選択する。

---

## 🚨 地雷ログ：異種DB移行の壁と「無停止」の実現

### 1. 遭遇したシナリオと私の誤認
- **要件:** オンプレミスのミッションクリティカルなOracle DBを、AWS Aurora PostgreSQLへ「ほぼ無停止（Near-zero downtime）」で移行したい。データベースエンジンの変更に伴うスキーマやストアドプロシージャの変換も必須。
- **私のミス:** S3経由でバックアップファイルをアップロードし、CloudFormationで力技で展開する（**S3 Transfer Acceleration + CloudFormation**）という物理法則を無視した構成を選んでしまいました。

### 2. クラウド設計における正解と選定理由
- **最適解:** **AWS Schema Conversion Tool (SCT) ＋ AWS Database Migration Service (DMS) with CDC**
- **アーキテク放談:** 商用DB（Oracle）からオープンソースベースのDB（PostgreSQL）への移行は「異種データベース移行（Heterogeneous Migration）」と呼ばれ、単なるバックアップファイルのコピーではターゲット側がデータを読み込めず100%失敗します。まず、AWS SCT（翻訳官）を使って互換性のないスキーマやプロシージャをPostgreSQL用に書き換えます。次に、AWS DMS（搬送業者）にCDC（Change Data Capture）を有効化して走らせることで、稼働中のオンプレDBの「差分トランザクション」をリアルタイムで同期し続け、本番切り替え時のダウンタイムを「ほぼゼロ」に抑え込むことができます。

### 🗺️ データフロー（垂直表示）
[Legacy Oracle DB]
     ↓
[AWS SCT (Schema)]
     ↓
[AWS DMS with CDC]
     ↓
[Aurora PostgreSQL]

### 🎯 脳内に刻む教訓
> 💡 `[ERR-Migration/Heterogeneous-SCT-DMS]` 「異種DBエンジン間の移行」＋「ゼロダウンタイム」という究極要件が出たら、SCT（スキーマ翻訳）とDMS CDC（差分データ同期）の黄金コンビ以外はあり得ない。