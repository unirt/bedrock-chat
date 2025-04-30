# Bedrock Chat データフロー分析

## 1. 概要

Bedrock Chatアプリケーションにおけるデータの流れを分析し、エンドツーエンドのデータフローを理解します。このドキュメントでは、ユーザーの入力からAIの応答生成、データの保存までの一連の流れを説明します。

## 2. 主要なデータフローパターン

Bedrock Chatには、以下の主要なデータフローパターンがあります：

1. **チャットメッセージのデータフロー**: ユーザー入力からAI応答までの流れ
2. **ボット管理のデータフロー**: ボットの作成、更新、共有の流れ
3. **知識ベース（RAG）のデータフロー**: ドキュメントのアップロードから埋め込み、検索までの流れ
4. **エージェント機能のデータフロー**: ツール使用のリクエストから結果の表示までの流れ
5. **認証・認可のデータフロー**: ユーザー認証と権限管理の流れ

## 3. チャットメッセージのデータフロー

### 3.1 基本的なチャットフロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Frontend as フロントエンド
    participant API as バックエンドAPI
    participant Bedrock as Amazon Bedrock
    participant DynamoDB as DynamoDB

    User->>Frontend: メッセージ入力
    Frontend->>API: POST /conversation
    API->>Bedrock: Converse API呼び出し
    Bedrock-->>API: AI応答
    API->>DynamoDB: 会話履歴保存
    API-->>Frontend: 応答返却
    Frontend-->>User: 応答表示
```

### 3.2 ストリーミングチャットフロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Frontend as フロントエンド
    participant WebSocket as WebSocket API
    participant Lambda as Lambda関数
    participant Bedrock as Amazon Bedrock
    participant DynamoDB as DynamoDB

    User->>Frontend: メッセージ入力
    Frontend->>WebSocket: WebSocket接続
    WebSocket->>Lambda: メッセージ転送
    Lambda->>Bedrock: Converse Stream API呼び出し
    
    loop ストリーミング応答
        Bedrock-->>Lambda: 部分的な応答
        Lambda-->>WebSocket: 部分的な応答転送
        WebSocket-->>Frontend: 部分的な応答表示
        Frontend-->>User: リアルタイム表示
    end
    
    Lambda->>DynamoDB: 完全な会話履歴保存
```

### 3.3 データ構造の変換

```mermaid
flowchart TD
    A[ユーザー入力] --> B[フロントエンドの状態]
    B --> C[API リクエスト]
    C --> D[Bedrock リクエスト]
    D --> E[Bedrock レスポンス]
    E --> F[API レスポンス]
    F --> G[フロントエンドの状態更新]
    G --> H[UI表示]
    
    subgraph フロントエンド
    A
    B
    G
    H
    end
    
    subgraph バックエンド
    C
    F
    end
    
    subgraph Bedrock
    D
    E
    end
```

## 4. ボット管理のデータフロー

### 4.1 ボット作成フロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Frontend as フロントエンド
    participant API as バックエンドAPI
    participant DynamoDB as DynamoDB
    participant S3 as S3バケット
    participant EventBridge as EventBridge
    participant StepFunctions as Step Functions
    participant Bedrock as Bedrock Knowledge Base

    User->>Frontend: ボット作成フォーム入力
    Frontend->>API: POST /bot
    API->>DynamoDB: ボット情報保存
    
    alt ファイルアップロードあり
        Frontend->>API: GET /bot/{botId}/presigned-url
        API-->>Frontend: 署名付きURL
        Frontend->>S3: ファイルアップロード
        S3->>EventBridge: イベント発行
        EventBridge->>StepFunctions: 埋め込み処理開始
        StepFunctions->>Bedrock: 埋め込み生成
        Bedrock-->>StepFunctions: 埋め込み完了
        StepFunctions->>DynamoDB: ステータス更新
    end
    
    API-->>Frontend: ボット作成完了
    Frontend-->>User: 完了通知
```

### 4.2 ボット共有フロー

```mermaid
flowchart TD
    A[ボット所有者] --> B[共有設定変更]
    B --> C[API: PATCH /bot/{botId}/visibility]
    C --> D[DynamoDB更新]
    D --> E[共有ステータス変更]
    
    F[他のユーザー] --> G[ボットストア閲覧]
    G --> H[API: GET /store/search または /store/popular]
    H --> I[共有ボット一覧取得]
    I --> J[ボット使用]
    J --> K[エイリアス作成]
    K --> L[DynamoDBにエイリアス保存]
```

## 5. 知識ベース（RAG）のデータフロー

### 5.1 ドキュメント処理フロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Frontend as フロントエンド
    participant API as バックエンドAPI
    participant S3 as S3バケット
    participant EventBridge as EventBridge Pipes
    participant StepFunctions as Step Functions
    participant Bedrock as Bedrock Embeddings
    participant OpenSearch as OpenSearch Serverless

    User->>Frontend: ドキュメントアップロード
    Frontend->>API: GET /bot/{botId}/presigned-url
    API-->>Frontend: 署名付きURL
    Frontend->>S3: ドキュメントアップロード
    
    S3->>EventBridge: オブジェクト作成イベント
    EventBridge->>StepFunctions: 埋め込み処理開始
    
    StepFunctions->>S3: ドキュメント取得
    S3-->>StepFunctions: ドキュメント内容
    
    StepFunctions->>Bedrock: テキスト埋め込み生成
    Bedrock-->>StepFunctions: 埋め込みベクトル
    
    StepFunctions->>OpenSearch: ベクトル保存
    StepFunctions->>DynamoDB: ステータス更新
    
    API-->>Frontend: 処理状況更新
    Frontend-->>User: 完了通知
```

### 5.2 RAG検索フロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Frontend as フロントエンド
    participant API as バックエンドAPI
    participant Bedrock as Bedrock Embeddings
    participant OpenSearch as OpenSearch Serverless
    participant BedrockLLM as Bedrock LLM

    User->>Frontend: 質問入力
    Frontend->>API: POST /conversation
    
    API->>Bedrock: クエリ埋め込み生成
    Bedrock-->>API: クエリベクトル
    
    API->>OpenSearch: ベクトル検索
    OpenSearch-->>API: 関連ドキュメント
    
    API->>BedrockLLM: プロンプト + 関連ドキュメント
    BedrockLLM-->>API: 回答生成
    
    API-->>Frontend: 回答 + 引用情報
    Frontend-->>User: 回答表示
```

## 6. エージェント機能のデータフロー

### 6.1 エージェントツール使用フロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Frontend as フロントエンド
    participant WebSocket as WebSocket API
    participant Lambda as Lambda関数
    participant Bedrock as Bedrock LLM
    participant Tools as エージェントツール
    participant DynamoDB as DynamoDB

    User->>Frontend: 質問入力
    Frontend->>WebSocket: WebSocket接続
    WebSocket->>Lambda: メッセージ転送
    Lambda->>Bedrock: Converse Stream API呼び出し
    
    Bedrock-->>Lambda: ツール使用リクエスト
    Lambda-->>WebSocket: ツール使用状態通知
    WebSocket-->>Frontend: エージェント思考表示
    
    Lambda->>Tools: ツール実行
    Tools-->>Lambda: ツール実行結果
    
    Lambda->>Bedrock: ツール結果を送信
    Bedrock-->>Lambda: 最終応答
    
    Lambda-->>WebSocket: 応答転送
    WebSocket-->>Frontend: 応答表示
    Frontend-->>User: 結果表示
    
    Lambda->>DynamoDB: 会話履歴保存
```

### 6.2 エージェント状態遷移

```mermaid
stateDiagram-v2
    [*] --> sleeping
    sleeping --> thinking: wakeup
    thinking --> thinking: thought
    thinking --> thinking: go-on
    thinking --> thinking: tool-result
    thinking --> thinking: related-document
    thinking --> leaving: goodbye
    leaving --> sleeping: after 2.5s
```

## 7. 認証・認可のデータフロー

### 7.1 認証フロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Frontend as フロントエンド
    participant Cognito as Amazon Cognito
    participant API as バックエンドAPI

    User->>Frontend: ログイン情報入力
    Frontend->>Cognito: 認証リクエスト
    Cognito-->>Frontend: JWT トークン
    
    Frontend->>API: APIリクエスト + JWT
    API->>Cognito: トークン検証
    Cognito-->>API: 検証結果
    
    alt 検証成功
        API-->>Frontend: APIレスポンス
    else 検証失敗
        API-->>Frontend: 401 Unauthorized
        Frontend-->>User: 再ログイン要求
    end
```

### 7.2 権限管理フロー

```mermaid
flowchart TD
    A[ユーザーリクエスト] --> B{ユーザーグループ確認}
    B -->|Admin| C[管理者機能アクセス許可]
    B -->|CreatingBotAllowed| D[ボット作成許可]
    B -->|PublishAllowed| E[API公開許可]
    B -->|なし| F[基本機能のみ許可]
    
    C --> G[管理者API]
    D --> H[ボット作成API]
    E --> I[API公開機能]
    F --> J[チャット機能]
```

## 8. エンドツーエンドのデータフロー

### 8.1 RAGを使用したチャットの完全なフロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Frontend as フロントエンド
    participant API as バックエンドAPI
    participant Cognito as Amazon Cognito
    participant DynamoDB as DynamoDB
    participant S3 as S3バケット
    participant Bedrock as Amazon Bedrock
    participant OpenSearch as OpenSearch Serverless

    User->>Frontend: ログイン
    Frontend->>Cognito: 認証リクエスト
    Cognito-->>Frontend: JWT トークン
    
    User->>Frontend: ボット選択
    Frontend->>API: GET /bot/{botId}
    API->>DynamoDB: ボット情報取得
    DynamoDB-->>API: ボット情報
    API-->>Frontend: ボット情報
    
    User->>Frontend: 質問入力
    Frontend->>API: POST /conversation
    API->>Cognito: トークン検証
    Cognito-->>API: 検証結果
    
    API->>Bedrock: クエリ埋め込み生成
    Bedrock-->>API: クエリベクトル
    
    API->>OpenSearch: ベクトル検索
    OpenSearch-->>API: 関連ドキュメント
    
    API->>S3: ドキュメント取得（必要な場合）
    S3-->>API: ドキュメント内容
    
    API->>Bedrock: Converse API呼び出し（プロンプト + 関連ドキュメント）
    Bedrock-->>API: AI応答
    
    API->>DynamoDB: 会話履歴保存
    API-->>Frontend: 応答 + 引用情報
    Frontend-->>User: 応答表示
```

### 8.2 エージェント機能を使用したチャットの完全なフロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Frontend as フロントエンド
    participant WebSocket as WebSocket API
    participant Lambda as Lambda関数
    participant Cognito as Amazon Cognito
    participant DynamoDB as DynamoDB
    participant Bedrock as Amazon Bedrock
    participant ExternalAPI as 外部API/ツール

    User->>Frontend: ログイン
    Frontend->>Cognito: 認証リクエスト
    Cognito-->>Frontend: JWT トークン
    
    User->>Frontend: エージェントボット選択
    Frontend->>API: GET /bot/{botId}
    API->>DynamoDB: ボット情報取得
    DynamoDB-->>API: ボット情報
    API-->>Frontend: ボット情報
    
    User->>Frontend: 質問入力
    Frontend->>WebSocket: WebSocket接続 + JWT
    WebSocket->>Lambda: メッセージ転送
    Lambda->>Cognito: トークン検証
    Cognito-->>Lambda: 検証結果
    
    Lambda->>DynamoDB: ボット設定取得
    DynamoDB-->>Lambda: ツール設定
    
    Lambda->>Bedrock: Converse Stream API呼び出し
    
    loop ストリーミング応答
        Bedrock-->>Lambda: 思考プロセス
        Lambda-->>WebSocket: 思考状態通知
        WebSocket-->>Frontend: エージェント思考表示
        Frontend-->>User: 思考プロセス表示
        
        Bedrock-->>Lambda: ツール使用リクエスト
        Lambda->>ExternalAPI: ツール実行
        ExternalAPI-->>Lambda: ツール結果
        Lambda->>Bedrock: ツール結果送信
    end
    
    Bedrock-->>Lambda: 最終応答
    Lambda-->>WebSocket: 応答転送
    WebSocket-->>Frontend: 応答表示
    Frontend-->>User: 結果表示
    
    Lambda->>DynamoDB: 会話履歴保存
```

## 9. データの永続化と状態管理

### 9.1 データ永続化の流れ

```mermaid
flowchart TD
    A[ユーザーアクション] --> B[フロントエンド状態更新]
    B --> C[API呼び出し]
    C --> D[バックエンド処理]
    D --> E{データ種別}
    
    E -->|会話| F[DynamoDB会話テーブル]
    E -->|ボット設定| G[DynamoDBボットテーブル]
    E -->|ドキュメント| H[S3バケット]
    E -->|埋め込み| I[OpenSearch]
    
    F --> J[会話履歴]
    G --> K[ボット設定]
    H --> L[ドキュメント保存]
    I --> M[検索インデックス]
```

### 9.2 フロントエンド状態管理の流れ

```mermaid
flowchart TD
    A[ユーザーアクション] --> B[Reactコンポーネント]
    B --> C[カスタムフック]
    C --> D{状態管理}
    
    D -->|チャット| E[useChatState]
    D -->|ボット| F[useBotState]
    D -->|認証| G[useAuthState]
    
    E --> H[Zustand Store]
    F --> H
    G --> H
    
    H --> I[永続化ストレージ]
    H --> J[UI更新]
```

## 10. エラーハンドリングとリカバリーフロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Frontend as フロントエンド
    participant API as バックエンドAPI
    participant Bedrock as Amazon Bedrock

    User->>Frontend: アクション実行
    Frontend->>API: APIリクエスト
    
    alt APIエラー
        API-->>Frontend: エラーレスポンス
        Frontend-->>User: エラー通知
        User->>Frontend: リトライ
        Frontend->>API: 再リクエスト
    end
    
    API->>Bedrock: Bedrockリクエスト
    
    alt Bedrockエラー
        Bedrock-->>API: エラーレスポンス
        API-->>Frontend: エラー情報
        Frontend-->>User: エラー通知
        User->>Frontend: 別のモデルで試す
        Frontend->>API: 新しいリクエスト
        API->>Bedrock: 新しいモデルでリクエスト
    end
    
    Bedrock-->>API: 正常レスポンス
    API-->>Frontend: 成功レスポンス
    Frontend-->>User: 結果表示
```

## 11. まとめ

Bedrock Chatアプリケーションのデータフローは、以下の特徴を持っています：

1. **多層アーキテクチャ**: フロントエンド、バックエンドAPI、AWSサービスの多層構造
2. **イベント駆動型処理**: EventBridge PipesとStep Functionsによる非同期処理
3. **リアルタイム通信**: WebSocketを使用したストリーミングレスポンス
4. **状態の分離**: フロントエンドとバックエンドでの明確な状態管理
5. **マイクロサービス的アプローチ**: 機能ごとに分離された処理フロー

これらのデータフローパターンにより、Bedrock Chatは拡張性が高く、保守性の良いアーキテクチャを実現しています。特に、RAG機能とエージェント機能のデータフローは、複雑な処理を効率的に実行するために最適化されています。
