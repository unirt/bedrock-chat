# Bedrock Chat バックエンドインフラストラクチャ

## 1. 全体アーキテクチャ

Bedrock Chatのバックエンドは、AWS CDKを使用してインフラストラクチャがコード化されています。主要なコンポーネントは以下の通りです：

- API Gateway + Lambda (FastAPI)
- DynamoDB (データストレージ)
- Amazon Cognito (認証)
- Amazon Bedrock (AI機能)
- Amazon S3 (ドキュメント保存)
- AWS WAF (セキュリティ)
- Amazon OpenSearch Serverless (ナレッジベース)
- EventBridge Pipes + Step Functions (データ処理パイプライン)

## 2. CDKスタック構成

### 2.1 メインスタック (`BedrockChatStack`)

`BedrockChatStack`は、アプリケーションの主要なリソースを定義するメインのCDKスタックです。このスタックは以下のコンストラクトを含んでいます：

- `Frontend`: CloudFront + S3によるフロントエンドホスティング
- `Auth`: Cognitoによるユーザー認証
- `Database`: DynamoDBテーブル
- `Api`: API Gateway + LambdaによるバックエンドAPI
- `WebSocket`: リアルタイム通信用のWebSocketAPI
- `Embedding`: ドキュメント埋め込み処理
- `BotStore`: カスタムボットのストア機能
- `UsageAnalysis`: 使用状況分析

### 2.2 その他のスタック

- `FrontendWafStack`: フロントエンドのWAF設定
- `BedrockCustomBotStack`: カスタムボット用のリソース
- `ApiPublishmentStack`: APIの公開用リソース

## 3. データベース設計

### 3.1 DynamoDBテーブル

#### 3.1.1 会話テーブル (`ConversationTableV3`)

会話履歴を保存するためのテーブルです。

- パーティションキー: `PK` (ユーザーID)
- ソートキー: `SK` (会話ID)
- グローバルセカンダリインデックス:
  - `SKIndex`: 会話IDによる検索用

#### 3.1.2 ボットテーブル (`BotTableV3`)

カスタムボットの情報を保存するためのテーブルです。

- パーティションキー: `PK` (ユーザーID)
- ソートキー: `SK` (アイテムタイプ)
- ローカルセカンダリインデックス:
  - `StarredIndex`: スター付きボットの検索用
  - `LastUsedTimeIndex`: 最終使用時間による検索用
- グローバルセカンダリインデックス:
  - `BotIdIndex`: ボットIDによる検索用
  - `SharedScopeIndex`: 共有スコープと状態による検索用
  - `ItemTypeIndex`: アイテムタイプによる検索用

#### 3.1.3 WebSocketセッションテーブル (`WebsocketSessionTable`)

WebSocketセッション情報を保存するためのテーブルです。

- パーティションキー: `ConnectionId`
- ソートキー: `MessagePartId`
- TTL属性: `expire`

### 3.2 OpenSearch Serverless

ボットストア機能とナレッジベース検索のために使用されます。ベクトル検索と全文検索の機能を提供します。

## 4. バックエンドAPI

### 4.1 API構成

FastAPIフレームワークを使用したPythonベースのAPIです。主要なルーターは以下の通りです：

- `conversation_router`: 会話管理API
- `bot_router`: ボット管理API
- `api_publication_router`: API公開機能
- `admin_router`: 管理者機能
- `user_router`: ユーザー管理
- `bot_store_router`: ボットストア機能
- `published_api_router`: 公開されたAPI用

### 4.2 Lambda関数

バックエンドAPIはLambda関数としてデプロイされ、AWS Lambda Web Adapterを使用してFastAPIアプリケーションをホストしています。主な特徴：

- Python 3.13ランタイム
- メモリサイズ: 1024MB
- タイムアウト: 15分
- Lambda SnapStartのオプションサポート
- 環境変数による設定

### 4.3 IAMロールとポリシー

バックエンドAPIのLambda関数には、以下のようなアクセス権限が付与されています：

- DynamoDBテーブルへのアクセス
- Amazon Bedrockへのアクセス
- CodeBuildプロジェクトの起動
- CloudFormationスタックの操作
- API Gatewayの操作
- Athenaクエリの実行
- Cognitoユーザープールの操作
- OpenSearch Serverlessへのアクセス
- SecretsManagerへのアクセス
- S3バケットへのアクセス

## 5. 認証と認可

### 5.1 Cognito認証

Amazon Cognitoを使用してユーザー認証を行います。主な機能：

- ユーザープールとアプリクライアント
- 自己登録オプション
- 外部IDプロバイダー連携オプション
- ドメイン制限オプション

### 5.2 認可モデル

- ユーザーグループによる権限管理
  - `Admin`: 管理者権限
  - `CreatingBotAllowed`: ボット作成権限
  - `PublishAllowed`: API公開権限
- テーブルアクセスロールによる行レベルのアクセス制御

## 6. WebSocket API

リアルタイムのストリーミングレスポンスを提供するためのWebSocket APIです。主な特徴：

- API Gateway WebSocket API
- Lambda関数によるメッセージ処理
- DynamoDBによるセッション管理
- 32KBを超えるメッセージの分割処理

## 7. ドキュメント処理パイプライン

### 7.1 埋め込み処理

ドキュメントの埋め込み処理を行うためのパイプラインです。主なコンポーネント：

- EventBridge Pipes: DynamoDBストリームからのイベント処理
- Step Functions: 埋め込み処理のオーケストレーション
- Lambda関数: ドキュメント処理と埋め込み生成
- Amazon Bedrock: 埋め込みモデル
- OpenSearch Serverless: ベクトルデータの保存

### 7.2 ドキュメントストレージ

S3バケットを使用してドキュメントを保存します。主な特徴：

- サーバーサイド暗号化
- パブリックアクセスのブロック
- CORS設定
- アクセスログ記録

## 8. 使用状況分析

### 8.1 分析パイプライン

使用状況データを分析するためのパイプラインです。主なコンポーネント：

- DynamoDBエクスポート
- Glueデータカタログ
- Athenaクエリ
- S3結果出力バケット

### 8.2 分析データ

- 会話履歴
- ボット使用状況
- API使用状況

## 9. セキュリティ対策

- AWS WAFによるIPアドレス制限
- S3バケットのパブリックアクセスブロック
- サーバーサイド暗号化
- IAMロールの最小権限原則
- Cognitoによる認証
- SSL/TLS通信の強制
- アクセスログの記録

## 10. デプロイ設定オプション

- Lambda SnapStartの有効/無効
- クロスリージョン推論の有効/無効
- RAGレプリカの有効/無効
- ボットストアの有効/無効と言語設定
- 自己登録の有効/無効
- IPアドレス制限
- サインアップメールドメイン制限
- カスタムドメイン設定
