# Bedrock Chat データモデル

## 1. 会話モデル

### 1.1 メッセージコンテンツモデル

メッセージの内容を表現するためのモデル群です。

#### 1.1.1 テキストコンテンツ (`TextContentModel`)

```python
class TextContentModel(BaseModel):
    content_type: Literal["text"]
    body: str
```

#### 1.1.2 画像コンテンツ (`ImageContentModel`)

```python
class ImageContentModel(BaseModel):
    content_type: Literal["image"]
    media_type: str
    body: Base64EncodedBytes
```

#### 1.1.3 添付ファイルコンテンツ (`AttachmentContentModel`)

```python
class AttachmentContentModel(BaseModel):
    content_type: Literal["attachment"]
    body: Base64EncodedBytes
    file_name: str
```

#### 1.1.4 ツール使用コンテンツ (`ToolUseContentModel`)

エージェントがツールを使用する際のコンテンツモデルです。

```python
class ToolUseContentModel(BaseModel):
    content_type: Literal["toolUse"]
    body: ToolUseContentModelBody
```

#### 1.1.5 ツール結果コンテンツ (`ToolResultContentModel`)

ツールの実行結果を表すコンテンツモデルです。

```python
class ToolResultContentModel(BaseModel):
    content_type: Literal["toolResult"]
    body: ToolResultContentModelBody
```

#### 1.1.6 推論コンテンツ (`ReasoningContentModel`)

AIの推論過程を表すコンテンツモデルです。

```python
class ReasoningContentModel(BaseModel):
    content_type: Literal["reasoning"]
    text: str
    signature: str
    redacted_content: Base64EncodedBytes
```

### 1.2 メッセージモデル (`MessageModel`)

会話内のメッセージを表すモデルです。

```python
class MessageModel(BaseModel):
    role: str
    content: list[ContentModel]
    model: type_model_name
    children: list[str]
    parent: str | None
    create_time: float
    feedback: FeedbackModel | None
    used_chunks: list[ChunkModel] | None
    thinking_log: list[SimpleMessageModel] | None
```

### 1.3 会話モデル (`ConversationModel`)

会話全体を表すモデルです。

```python
class ConversationModel(BaseModel):
    id: str
    create_time: float
    title: str
    total_price: float
    message_map: dict[str, MessageModel]
    last_message_id: str
    bot_id: str | None
    should_continue: bool
```

### 1.4 関連ドキュメントモデル (`RelatedDocumentModel`)

会話に関連するドキュメントを表すモデルです。

```python
class RelatedDocumentModel(BaseModel):
    content: ToolResultModel
    source_id: str
    source_name: str | None
    source_link: str | None
    page_number: int | None
```

## 2. ボットモデル

### 2.1 ボット設定モデル

#### 2.1.1 知識ベースモデル (`KnowledgeModel`)

ボットの知識ベースを表すモデルです。

```python
class KnowledgeModel(BaseModel):
    source_urls: list[str]
    sitemap_urls: list[str]
    filenames: list[str]
    s3_urls: list[str]
```

#### 2.1.2 生成パラメータモデル (`GenerationParamsModel`)

テキスト生成のパラメータを表すモデルです。

```python
class GenerationParamsModel(BaseModel):
    max_tokens: int
    top_k: int
    top_p: Float
    temperature: Float
    stop_sequences: list[str]
    reasoning_params: ReasoningParamsModel
```

#### 2.1.3 推論パラメータモデル (`ReasoningParamsModel`)

推論のパラメータを表すモデルです。

```python
class ReasoningParamsModel(BaseModel):
    budget_tokens: int
```

### 2.2 エージェントモデル

#### 2.2.1 ツールモデル

エージェントが使用するツールを表すモデル群です。

```python
# 基本ツール
class PlainToolModel(BaseModel):
    tool_type: Literal["plain"]
    name: str
    description: str

# インターネット検索ツール
class InternetToolModel(BaseModel):
    tool_type: Literal["internet"]
    name: str
    description: str
    search_engine: Optional[Literal["duckduckgo", "firecrawl"]]
    firecrawl_config: Optional[FirecrawlConfigModel] | None

# Bedrock エージェントツール
class BedrockAgentToolModel(BaseModel):
    tool_type: Literal["bedrock_agent"]
    name: str
    description: str
    bedrockAgentConfig: Optional[BedrockAgentConfigModel] | None
```

#### 2.2.2 エージェントモデル (`AgentModel`)

エージェントの設定を表すモデルです。

```python
class AgentModel(BaseModel):
    tools: list[ToolModel]
```

### 2.3 ボットモデル (`BotModel`)

カスタムボットを表すメインのモデルです。

```python
class BotModel(BaseModel):
    id: str
    owner_user_id: str
    title: str
    description: str
    instruction: str
    create_time: Float
    last_used_time: Float | None
    shared_scope: type_shared_scope
    shared_status: str
    allowed_cognito_groups: list[str]
    allowed_cognito_users: list[str]
    is_starred: bool
    generation_params: GenerationParamsModel
    agent: AgentModel
    knowledge: KnowledgeModel
    sync_status: type_sync_status
    sync_status_reason: str
    sync_last_exec_id: str
    published_api_stack_name: str | None
    published_api_datetime: int | None
    published_api_codebuild_id: str | None
    display_retrieved_chunks: bool
    conversation_quick_starters: list[ConversationQuickStarterModel]
    bedrock_knowledge_base: BedrockKnowledgeBaseModel | None
    bedrock_guardrails: BedrockGuardrailsModel | None
    active_models: ActiveModelsModel
    usage_stats: UsageStatsModel
```

### 2.4 ボットエイリアスモデル (`BotAliasModel`)

共有されたボットへのエイリアスを表すモデルです。

```python
class BotAliasModel(BaseModel):
    original_bot_id: str
    owner_user_id: str
    title: str
    description: str
    is_origin_accessible: bool
    create_time: Float
    last_used_time: Float
    is_starred: bool
    sync_status: type_sync_status
    has_knowledge: bool
    has_agent: bool
    conversation_quick_starters: list[ConversationQuickStarterModel]
    active_models: ActiveModelsModel
```

## 3. API公開モデル

### 3.1 API使用プランモデル (`ApiUsagePlanModel`)

API使用プランを表すモデルです。

```python
class ApiUsagePlanModel(BaseModel):
    id: str
    name: str
    quota: ApiUsagePlanQuotaModel
    throttle: ApiUsagePlanThrottleModel
    key_ids: list[str]
```

### 3.2 APIキーモデル (`ApiKeyModel`)

APIキーを表すモデルです。

```python
class ApiKeyModel(BaseModel):
    id: str
    description: str
    value: str
    enabled: bool
    created_date: int
```

### 3.3 公開APIスタックモデル (`PublishedApiStackModel`)

公開されたAPIのCloudFormationスタック情報を表すモデルです。

```python
class PublishedApiStackModel(BaseModel):
    stack_id: str
    stack_name: str
    stack_status: str
    api_id: str | None
    api_name: str | None
    api_usage_plan_id: str | None
    api_allowed_origins: list[str] | None
    api_stage: str | None
    create_time: int
```

## 4. DynamoDBテーブル設計

### 4.1 会話テーブル

- パーティションキー: `PK` (ユーザーID)
- ソートキー: `SK` (会話ID)
- 属性:
  - `Title`: 会話のタイトル
  - `CreateTime`: 作成時間
  - `TotalPrice`: 合計価格
  - `MessageMap`: メッセージマップ
  - `LastMessageId`: 最後のメッセージID
  - `BotId`: ボットID
  - `ShouldContinue`: 続行フラグ

### 4.2 ボットテーブル

- パーティションキー: `PK` (ユーザーID)
- ソートキー: `SK` (アイテムタイプ)
- 属性:
  - `Id`: ボットID
  - `Title`: ボットのタイトル
  - `Description`: ボットの説明
  - `Instruction`: ボットの指示
  - `CreateTime`: 作成時間
  - `LastUsedTime`: 最終使用時間
  - `SharedScope`: 共有スコープ
  - `SharedStatus`: 共有状態
  - `AllowedCognitoGroups`: 許可されたCognitoグループ
  - `AllowedCognitoUsers`: 許可されたCognitoユーザー
  - `IsStarred`: スター付きフラグ
  - `GenerationParams`: 生成パラメータ
  - `Agent`: エージェント設定
  - `Knowledge`: 知識ベース設定
  - `SyncStatus`: 同期状態
  - `SyncStatusReason`: 同期状態の理由
  - `SyncLastExecId`: 最後の実行ID
  - `PublishedApiStackName`: 公開APIスタック名
  - `PublishedApiDatetime`: API公開日時
  - `PublishedApiCodebuildId`: API公開CodebuildID
  - `DisplayRetrievedChunks`: チャンク表示フラグ
  - `ConversationQuickStarters`: 会話クイックスターター
  - `BedrockKnowledgeBase`: Bedrock知識ベース設定
  - `BedrockGuardrails`: Bedrockガードレール設定
  - `ActiveModels`: アクティブなモデル
  - `UsageStats`: 使用統計

## 5. データモデル間の関係

1. **ユーザーとボット**: ユーザーは複数のボットを所有できます（1対多）
2. **ユーザーと会話**: ユーザーは複数の会話を持つことができます（1対多）
3. **ボットと会話**: 会話は1つのボットに関連付けられます（多対1）
4. **ボットとエイリアス**: 共有されたボットに対して複数のエイリアスが存在する可能性があります（1対多）
5. **会話とメッセージ**: 会話は複数のメッセージを含みます（1対多）
6. **メッセージとコンテンツ**: メッセージは複数のコンテンツを含みます（1対多）
7. **ボットとAPI公開**: ボットは1つのAPI公開設定を持つことができます（1対1）

## 6. 主要なデータフロー

1. **会話の作成と更新**:
   - ユーザーがメッセージを送信
   - `MessageModel`が作成され、`ConversationModel`に追加
   - 会話テーブルに保存

2. **ボットの作成と更新**:
   - ユーザーがボット設定を入力
   - `BotModel`が作成され、ボットテーブルに保存
   - 知識ベースがある場合は埋め込み処理が開始

3. **ボットの共有**:
   - ボット所有者が共有設定を更新
   - `BotModel`の`shared_scope`と`shared_status`が更新
   - 他のユーザーがアクセス可能になる

4. **API公開**:
   - ボット所有者がAPI公開を要求
   - CloudFormationスタックが作成され、APIがデプロイ
   - `PublishedApiStackModel`情報がボットに関連付けられる
