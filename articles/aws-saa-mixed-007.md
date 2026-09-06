---
title: "【AWS】ノンインフラ出身者がインフラの深淵に挑む：今週の地雷と復盤（VPC Endpoints & ECR編）"
emoji: "🚀"
type: "tech"
topics: ["aws", "infrastructure", "devops", "saa"]
published: true
---

# インフラ初心者が今週踏み抜いたAWSの地雷まとめ

今週、AWS SAAの設計シナリオに挑戦する中で、実務でも致命傷になりかねない重要な仕様の勘違い（地雷）を踏み抜きました。今回は、AWSのマネージドサービスの「裏側で動いている依存関係」を知らないと完全に詰む、ネットワークとコンテナの罠です。エンジニアとしての「振り返り」を兼ねて、アーキテクチャの解説と共に資産としてここに記録します。

---

## 🚨 地雷ログ：完全閉域網からのコンテナプルとECRの「隠れた依存関係」

### 1. 遭遇したシナリオと私の誤認
- **要件:** NATゲートウェイもインターネットゲートウェイ（IGW）も一切存在しない、完全に隔離されたプライベートVPC内で、ECS Fargateを起動し、Amazon ECRからコンテナイメージをプル（Pull）する。
- **私のミス:** 「ECRからイメージをダウンロードするのだから、ECR用のVPCエンドポイント（Interface Endpoint）さえ開通させれば通信できるはずだ」という直感に従い、他のエンドポイントを不要とする構成（**Only Interface VPC Endpoints for Amazon ECR**）を選択してしまいました。

### 2. クラウド設計における正解と選定理由
- **最適解:** **Interface VPC Endpoints for Amazon ECR ＋ Gateway VPC Endpoint for Amazon S3**
- **アーキテク放談:** これは実務でも本当によくある（そして原因究明に時間がかかる）トラップです。実は、Amazon ECRはAPI認証とマニフェストの管理を行っているだけであり、重い「Dockerイメージのレイヤーデータ（実体）」は、裏側で**Amazon S3**に保存されています。つまり、完全な閉域網（プライベート環境）でFargateがイメージをプルしようとした時、ECRのエンドポイントだけでは認証は通っても、実データのダウンロードでS3への通信がインターネットに抜けようとしてタイムアウト（Timeout）してしまいます。これを防ぐためには、ECRのInterface Endpoint（API用）と、S3のGateway Endpoint（データダウンロード用）の2つをセットで配置することがアーキテクチャ上の絶対条件となります。

### 🗺️ データフロー（垂直表示）
[ECS Fargate]
     ↓ (Auth & Manifest)
[ECR Interface VPC EP]
     ↓
[Amazon ECR API]
     ↓ (Image Layers)
[S3 Gateway VPC EP]
     ↓
[Amazon S3 (Data)]

### 🎯 脳内に刻む教訓
> 💡 `[ERR-Network/ECR-S3-Dependency]` 隔離されたVPCでECRからイメージを引く時、ECRはS3と「一心同体」であると心得る。ECR（Interface）とS3（Gateway）のエンドポイントは必ずセットで構築せよ。