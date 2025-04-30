# Bedrock Chat バックエンド設計パターン分析

## 1. 概要

Bedrock Chatのバックエンドでは、保守性、拡張性、テスト容易性を高めるために様々な設計パターンが適用されています。このドキュメントでは、バックエンドで使用されている主要な設計パターンを整理し、それらの適用意図と効果を分析します。

## 2. アーキテクチャパターン

### 2.1 レイヤードアーキテクチャ

バックエンドは明確なレイヤー構造を持っています。

```
app/
├── routes/         # ルーティングとリクエスト処理
├── usecases/       # ビジネスロジック
├── repositories/   # データアクセス
├── models/         # データモデル
```

**適用意図**:
- 関心の分離による保守性の向上
- 各レイヤーの独立したテスト容易性
- 変更の影響範囲の局所化

**効果**:
- コードの構造が明確になり、新機能の追加が容易
- レイヤー間の依存関係が明確で、変更の影響が予測可能
- 各レイヤーを独立してテスト可能

### 2.2 サーバーレスアーキテクチャ

AWS Lambdaを使用したサーバーレスアーキテクチャを採用しています。

**適用意図**:
- インフラストラクチャ管理の簡素化
- スケーラビリティの向上
- コスト最適化

**効果**:
- サーバー管理が不要で運用負荷が軽減
- 需要に応じた自動スケーリング
- 使用量に基づく課金で効率的なコスト管理

## 3. APIデザインパターン

### 3.1 RESTful APIパターン

HTTPメソッドとリソースに基づくRESTful APIデザインを採用しています。

```python
@router.get("/conversation/{conversation_id}")
def get_conversation(request: Request, conversation_id: str):
    """Get a conversation history"""
    current_user: User = request.state.current_user
    output = fetch_conversation(current_user.id, conversation_id)
    return output

@router.post("/conversation")
def post_message(request: Request, chat_input: ChatInput):
    """Send chat message"""
    current_user: User = request.state.current_user
    conversation, message = chat(user=current_user, chat_input=chat_input)
    output = chat_output_from_message(conversation=conversation, message=message)
    return output
```

**適用意図**:
- 標準的なAPIインターフェースの提供
- クライアントとサーバーの疎結合
- キャッシュ可能性の向上

**効果**:
- APIの理解と使用が容易
- フロントエンドとバックエンドの独立した開発が可能
- 標準的なHTTPキャッシュメカニズムの活用

### 3.2 ミドルウェアパターン

リクエスト処理のパイプラインにミドルウェアを適用しています。

```python
@app.middleware("http")
def add_current_user_to_request(request: Request, call_next: ASGIApp):
    if is_running_on_lambda():
        if not is_published_api:
            authorization = request.headers.get("Authorization")
            if authorization:
                token_str = authorization.split(" ")[1]
                token = HTTPAuthorizationCredentials(
                    scheme="Bearer", credentials=token_str
                )
                request.state.current_user = get_current_user(token)
        else:
            assert PUBLISHED_API_ID is not None, "PUBLISHED_API_ID is not set."
            request.state.current_user = User.from_published_api_id(PUBLISHED_API_ID)
    # ...
    response = call_next(request)
    return response
```

**適用意図**:
- 横断的関心事の分離
- リクエスト処理の前後に共通処理を挿入
- コードの重複を避ける

**効果**:
- 認証、ロギングなどの共通処理が一元管理
- エンドポイントごとのコードが簡潔に
- 共通処理の変更が容易

## 4. データアクセスパターン

### 4.1 リポジトリパターン

データアクセスロジックをリポジトリクラスにカプセル化しています。

```python
# repositories/conversation.py
def find_conversation_by_id(user_id: str, conversation_id: str) -> ConversationModel:
    """Find conversation by id"""
    # ...

def store_conversation(user_id: str, conversation: ConversationModel) -> None:
    """Store conversation"""
    # ...
```

**適用意図**:
- データアクセスロジックの抽象化
- データソースの詳細からビジネスロジックを分離
- テスト容易性の向上

**効果**:
- データベース実装の詳細がビジネスロジックから隠蔽
- モックを使用したテストが容易
- データソースの変更が容易

### 4.2 データマッパーパターン

DynamoDBのデータ構造とアプリケーションのモデル間のマッピングを行っています。

```python
def _to_model(item: dict) -> ConversationModel:
    """Convert DynamoDB item to ConversationModel"""
    # ...

def _to_item(user_id: str, conversation: ConversationModel) -> dict:
    """Convert ConversationModel to DynamoDB item"""
    # ...
```

**適用意図**:
- データベース構造とドメインモデルの分離
- データ変換ロジックの一元管理
- ドメインモデルの純粋性の維持

**効果**:
- データベーススキーマの変更がドメインモデルに影響しない
- ドメインモデルがデータベース依存から解放
- データ変換ロジックのテストが容易

## 5. ドメインモデリングパターン

### 5.1 値オブジェクトパターン

不変の値オブジェクトを使用してドメインの概念を表現しています。

```python
class FeedbackModel(BaseModel):
    thumbs_up: bool
    category: str
    comment: str
```

**適用意図**:
- ドメイン概念の明確な表現
- 不変性による予測可能性
- 自己検証による整合性の確保

**効果**:
- ドメインロジックが明確に表現され理解しやすい
- 不変性により副作用が減少
- バリデーションが一元管理され整合性が向上

### 5.2 エンティティパターン

識別子を持つエンティティを使用してドメインオブジェクトを表現しています。

```python
class BotModel(BaseModel):
    id: str
    owner_user_id: str
    title: str
    description: str
    instruction: str
    # ...
```

**適用意図**:
- 一意に識別可能なドメインオブジェクトの表現
- ライフサイクル管理の明確化
- ドメインロジックのカプセル化

**効果**:
- オブジェクトの同一性が明確
- 状態変更の追跡が容易
- ドメインルールがエンティティに集約

## 6. ビジネスロジックパターン

### 6.1 ユースケースパターン

ビジネスロジックをユースケースとして分離しています。

```python
# usecases/chat.py
def chat(
    user: User,
    chat_input: ChatInput,
    on_stream: Callable[[str], None] | None = None,
    on_stop: Callable[[OnStopInput], None] | None = None,
    on_thinking: Callable[[OnThinking], None] | None = None,
    on_tool_result: Callable[[ToolRunResult], None] | None = None,
    on_reasoning: Callable[[str], None] | None = None,
) -> tuple[ConversationModel, MessageModel]:
    # ビジネスロジックの実装
    # ...
```

**適用意図**:
- ビジネスロジックの明確な境界設定
- 単一責任の原則の適用
- テスト容易性の向上

**効果**:
- ビジネスロジックが明確に分離され理解しやすい
- 各ユースケースが独立してテスト可能
- 変更の影響範囲が局所化

### 6.2 サービスパターン

共通のビジネスロジックをサービスとして提供しています。

```python
# bedrock.py
def calculate_price(
    model: type_model_name,
    input_tokens: int,
    output_tokens: int,
    region: str = BEDROCK_REGION,
) -> float:
    # ...

def get_model_id(
    model: type_model_name,
    enable_cross_region: bool = ENABLE_BEDROCK_CROSS_REGION_INFERENCE,
    bedrock_region: str = BEDROCK_REGION,
) -> str:
    # ...
```

**適用意図**:
- 共通ロジックの再利用
- ドメインサービスの提供
- 関心の分離

**効果**:
- コードの重複が減少
- 共通ロジックの一元管理
- ビジネスルールの一貫性が向上

## 7. エラーハンドリングパターン

### 7.1 例外ベースのエラーハンドリング

ドメイン固有の例外クラスを定義し、集中的にハンドリングしています。

```python
# repositories/common.py
class RecordNotFoundError(Exception):
    """Record not found error"""
    pass

class RecordAccessNotAllowedError(Exception):
    """Record access not allowed error"""
    pass

# main.py
app.add_exception_handler(RecordNotFoundError, error_handler_factory(404))
app.add_exception_handler(RecordAccessNotAllowedError, error_handler_factory(403))
```

**適用意図**:
- エラー処理の一元化
- ドメイン固有のエラー状態の明確な表現
- エラーレスポンスの標準化

**効果**:
- エラー処理が一貫して行われる
- エラーの種類と原因が明確
- クライアントへの適切なエラー情報の提供

### 7.2 結果オブジェクトパターン

一部の操作では、例外の代わりに結果オブジェクトを返しています。

```python
def search_related_docs(bot: BotModel, query: str) -> list[SearchResult]:
    try:
        # 検索処理
        return search_results
    except ClientError as e:
        logger.error(f"Error querying Bedrock Knowledge Base: {e}")
        raise e
```

**適用意図**:
- エラー状態の明示的な表現
- 例外に頼らないエラーフロー
- 予測可能なエラー処理

**効果**:
- エラーフローが明示的で追跡しやすい
- 呼び出し元がエラー処理を強制される
- 例外の乱用を防止

## 8. 非同期処理パターン

### 8.1 イベント駆動パターン

EventBridge PipesとStep Functionsを使用したイベント駆動アーキテクチャを採用しています。

**適用意図**:
- 処理の非同期化によるレスポンス時間の短縮
- システムコンポーネント間の疎結合
- スケーラビリティの向上

**効果**:
- ユーザー体験の向上（長時間処理をブロックしない）
- システムコンポーネントの独立した進化が可能
- 負荷分散と耐障害性の向上

### 8.2 ストリーミングレスポンスパターン

WebSocketを使用したストリーミングレスポンスを実装しています。

```python
# websocket.py
def on_message(event, context):
    # WebSocketメッセージ処理
    # ...
    
    # ストリーミングレスポンス送信
    for chunk in response_chunks:
        send_to_connection(connection_id, chunk)
```

**適用意図**:
- リアルタイムのユーザーフィードバック
- 長時間処理の進捗表示
- ユーザー体験の向上

**効果**:
- AIの思考プロセスがリアルタイムで表示される
- ユーザーが応答を待つ間のフィードバックが提供される
- 大きなレスポンスが効率的に処理される

## 9. セキュリティパターン

### 9.1 認証・認可パターン

JWTベースの認証と役割ベースのアクセス制御を実装しています。

```python
# dependencies.py
def check_admin(request: Request):
    current_user: User = request.state.current_user
    if not current_user.is_admin():
        raise PermissionError("Admin permission required")
    return True

# routes/admin.py
@router.get("/admin/published-bots", response_model=PublishedBotOutputsWithNextToken)
def get_all_published_bots(
    next_token: str | None = None,
    limit: int = 1000,
    admin_check=Depends(check_admin),
):
    # 管理者のみアクセス可能なエンドポイント
    # ...
```

**適用意図**:
- アクセス制御の一元管理
- 権限チェックの標準化
- セキュリティポリシーの明確な実装

**効果**:
- 認証・認可ロジックが一貫して適用される
- 権限チェックの漏れを防止
- セキュリティポリシーの変更が容易

### 9.2 最小権限の原則

IAMロールとポリシーに最小権限の原則を適用しています。

```python
# api.ts (CDK)
handlerRole.addToPolicy(
  new iam.PolicyStatement({
    actions: ["bedrock:*"],
    resources: ["*"],
  })
);

handlerRole.addToPolicy(
  new iam.PolicyStatement({
    effect: iam.Effect.ALLOW,
    actions: ["codebuild:StartBuild"],
    resources: [
      props.apiPublishProject.projectArn,
      props.bedrockCustomBotProject.projectArn,
    ],
  })
);
```

**適用意図**:
- セキュリティリスクの最小化
- 権限の明示的な付与
- 意図しない権限昇格の防止

**効果**:
- セキュリティ侵害の影響範囲が限定される
- 権限が明示的で監査可能
- 最小限の権限で運用可能

## 10. まとめ

Bedrock Chatのバックエンドでは、以下の設計パターンが効果的に適用されています：

1. **アーキテクチャパターン**: レイヤードアーキテクチャとサーバーレスアーキテクチャによる関心の分離と運用の簡素化
2. **APIデザインパターン**: RESTful APIとミドルウェアパターンによる標準的で拡張性の高いAPI設計
3. **データアクセスパターン**: リポジトリパターンとデータマッパーによるデータアクセスの抽象化
4. **ドメインモデリングパターン**: 値オブジェクトとエンティティによるドメイン概念の明確な表現
5. **ビジネスロジックパターン**: ユースケースとサービスパターンによるビジネスロジックの分離と再利用
6. **エラーハンドリングパターン**: 例外ベースと結果オブジェクトによる一貫したエラー処理
7. **非同期処理パターン**: イベント駆動とストリーミングレスポンスによるユーザー体験の向上
8. **セキュリティパターン**: 認証・認可と最小権限の原則による堅牢なセキュリティ

これらの設計パターンの適用により、Bedrock Chatのバックエンドは保守性、拡張性、テスト容易性、セキュリティ性に優れたアーキテクチャを実現しています。
