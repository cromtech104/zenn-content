---
title: "GitHubリポジトリからドキュメントを自動生成するSaaSを個人で作った話"
emoji: "📄"
type: "tech"
topics: ["個人開発", "AWS", "FastAPI", "githubapp", "lambda"]
published: false
---

## 作った背景

「このリポジトリ、誰も全体像を把握してないんだよね」——自分で開発していてもこういう状況になることがあります。機能は動いている。テストもある。でも半年後に見返すと、なぜこの設計にしたのか、どこから読めばいいのか、すぐには思い出せない。

チームならもっと深刻です。新しいメンバーが加わるたびに「コードを読んで把握してください」になる。ドキュメントを書こうとはするけど、機能開発の忙しさの中で後回しになる。書くコストが構造的に高すぎるのが原因だと感じていました。

そこで作ったのが **[RepoCarta](https://repocarta.jp)** です。GitHubリポジトリを接続するだけで、AIがコードを読んでドキュメントを自動生成・自動更新するSaaSです。

## どんなサービスか

3ステップで使えます。

1. **GitHubアカウントでログイン** → GitHub Appで対象リポジトリを接続
2. **ドキュメント種別を選択**（10種類から必要なものだけ）
3. **AIが数分で生成** → 以降はpushのたびに自動更新

生成できるドキュメントは10種類です。

| カテゴリ | 種別 |
|---------|------|
| 設計 | アーキテクチャ設計書、API仕様書、DB設計書、画面設計書 |
| 開発 | 環境構築手順、運用手順書、開発ガイド、トラブルシューティング |
| テスト・レビュー | テストケース、課題・改善リスト |

生成後はドキュメントに対してQ&Aチャットができます。「この認証どこ？」「このAPIの制約は？」といった質問に、コードとドキュメントを根拠に答えます。

## 技術スタック

個人開発なのでコストと運用負荷を最小化する方針で選定しました。

| カテゴリ | 技術 |
|---------|------|
| バックエンド | FastAPI + Mangum（AWS Lambda上で動かすためのアダプター） |
| フロントエンド | React + Vite |
| インフラ | AWS Lambda, API Gateway HTTP API, SQS, RDS(PostgreSQL 16), S3 |
| IaC | Terraform |
| AI（生成） | Claude API（claude-sonnet-4-6） |
| AI（検索） | Amazon Bedrock Titan Text Embeddings + pgvector |
| GitHub連携 | GitHub App + Webhooks |
| 認証 | GitHub OAuth + JWT（HttpOnly Cookie） |
| 課金 | Stripe |

## アーキテクチャ

大きく3つのLambda関数で構成しています。

```
[GitHub Webhook]
      ↓
[API Lambda]（FastAPI + Mangum）
      ↓ SQSにキューイング（即座に200を返す）
[SQS]
      ↓
[doc-updater Lambda]
      ↓ Claude APIでPR差分を分析・生成
[GitHub] ← ドキュメント更新PRを自動作成
```

Q&Aのフローは別系統です。

```
[ユーザーの質問]
      ↓
[API Lambda]
      ↓ 同期呼び出し
[qa-agent Lambda]
      ↓ Bedrock Titan Embeddingsでベクトル化 → pgvectorで類似検索
      ↓ Claude APIで回答生成
[回答 + 参照元ドキュメント一覧を返却]
```

### SQSを挟む理由

GitHubのWebhookにはタイムアウトがあります。ドキュメント生成はClaudeのAPIコールを含むため数十秒かかることがある。API Lambdaで直接処理しようとするとタイムアウトになる。

そのためAPI Lambdaは受け取ったWebhookをSQSに積んで即座に200を返し、doc-updater Lambdaが非同期で実際の処理を担う構成にしました。これでGitHubからの再送問題も防げます。

### NATインスタンスに自前EC2を使う理由

LambdaをVPC内に置く場合、RDSへの安全なアクセスのためプライベートサブネットに配置します。するとインターネット（GitHub APIやAnthropic API）への通信にNATが必要になります。

AWSのマネージドサービス「NAT Gateway」は月額約$32かかります。代わりにEC2 t3.nano（月額約$4）で自前NATを構築しました。個人開発で高可用性よりコストを優先した判断です。

### Q&Aのモデル使い分け

ドキュメント生成にはClaudeのSonnetモデル、Q&Aの判定フェーズにはHaikuモデルを使い分けています。「AIが判断・分類するだけ」の処理にはHaiku、「AIが実際にコンテンツを書く」処理にはSonnetという方針です。Q&Aのおよそ70%はドキュメント検索だけで回答できるため、その部分のコストをHaikuで抑えています。

## 苦労したこと

### GitHub App認証

GitHub AppはOAuthとは全く異なる認証フローを持ちます。

- App自体の認証：RSA秘密鍵で署名したJWT
- リポジトリへのアクセス：Installation Access Token（1時間で失効）
- Webhookのイベント分岐：インストール・push・PRマージなどを同一エンドポイントで受け取り、イベントタイプで処理を分岐

Installation Access Tokenのキャッシュ管理（失効5分前に再取得）や、Webhook署名検証の実装など、ドキュメントが散らばっていて把握するまでが大変でした。

### pgvectorの導入

Q&A機能のセマンティック検索にはpgvector（PostgreSQLのベクトル拡張）を使っています。Amazon Bedrock Titan Text Embeddingsでドキュメントをベクトル化し、質問もベクトル化してコサイン類似度で上位5件を取得、Claudeが回答を生成します。

RDSにpgvectorを有効化するためのマイグレーションスクリプトを書いたり、初回インデックス構築スクリプトを書いたりと、ドキュメント生成以外の基盤部分の作り込みが予想より多かったです。

### LambdaのZIPビルド問題

macOSでビルドしたZIPをLambdaにデプロイすると、pydanticなどネイティブバイナリを含むパッケージで`ImportError`が発生します。LambdaはLinux x86_64環境で動くため、DockerでLinux向けにビルドする必要があります。

GitHub Actionsで自動化するまでは手動でDockerビルドしていました。

## 現在の料金

| プラン | 価格 | プロジェクト数 |
|--------|------|--------------|
| Free | ¥0 | 1件 |
| Solo | ¥4,980（税込） | 5件 |
| Team / Agency | 提供予定 | — |

Freeプランはドキュメント種別が最大3種、Soloプランで全10種が利用できます。詳細は[料金ページ](https://repocarta.jp/#pricing)を参照してください。

## まとめ

「コードはあるがドキュメントがない」という問題は、個人開発でもチーム開発でも共通の課題です。RepoCarta はその問題をAIと自動化で解決しようとしています。

GitHubアカウントがあれば**無料で始められます**。プライベートリポジトリにも対応しています。

https://repocarta.jp

フィードバックお待ちしています。コメント欄・X（[@takuyahoritacromtech](https://x.com/takuyahoritacromtech)）どちらでも歓迎です。
