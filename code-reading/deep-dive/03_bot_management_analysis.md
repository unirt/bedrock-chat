# Bedrock Chat ボット管理機能の詳細分析

## 1. ボット管理機能の概要

Bedrock Chatのボット管理機能は、ユーザーがカスタムボットを作成、設定、共有、管理するための包括的な機能セットを提供します。この機能により、ユーザーは特定の目的や知識領域に特化したAIアシスタントを構築し、他のユーザーと共有することができます。

## 2. アーキテクチャ構成

### 2.1 全体アーキテクチャ

ボット管理機能のアーキテクチャは以下のコンポーネントで構成されています：

```
[フロントエンド (React)] <---> [バックエンドAPI (FastAPI)] <---> [DynamoDB]
       |                               |                            |
       v                               v                            v
[ボット設定UI] <-------------> [ボット管理API] <------------> [ボットテーブル]
       |                               |                            |
       v                               v                            v
[ボットストアUI] <------------> [ボットストアAPI] <---------> [共有ボット]
```

- **フロントエンド**: ボット作成・管理・共有のためのUI
- **バックエンドAPI**: ボット関連の操作を処理するRESTful API
- **DynamoDB**: ボット設定と共有情報を保存するデータベース
- **ボットストア**: 共有ボットの検索と利用のためのインターフェース

### 2.2 データフロー

ボット管理のデータフローは以下の主要なフローに分けられます：

1. **ボット作成フロー**:
   - ユーザーがボット設定（タイトル、説明、指示など）を入力
   - 設定がバックエンドAPIに送信
   - ボット情報がDynamoDBに保存
   - 知識ベースがある場合は埋め込み処理が開始

2. **ボット共有フロー**:
   - ボット所有者が共有設定を更新
   - 共有スコープと許可されたユーザー/グループが設定
   - 共有情報がDynamoDBに保存
   - 共有ボットがボットストアで利用可能に

3. **ボット使用フロー**:
   - ユーザーがボットストアからボットを選択
   - ボット情報がフロントエンドに読み込まれる
   - ユーザーがボットとチャット開始
   - ボットの使用統計が更新

## 3. ボットモデルとデータ構造

### 3.1 ボットモデル (`BotModel`)

ボットの設定と状態を表す中心的なデータモデルです：

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

### 3.2 ボットエイリアスモデル (`BotAliasModel`)

共有ボットへのエイリアスを表すモデルです：

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

### 3.3 ボット設定モデル

ボットの各種設定を表すモデル群です：

#### 3.3.1 生成パラメータモデル (`GenerationParamsModel`)

```python
class GenerationParamsModel(BaseModel):
    max_tokens: int
    top_k: int
    top_p: Float
    temperature: Float
    stop_sequences: list[str]
    reasoning_params: ReasoningParamsModel
```

#### 3.3.2 知識ベースモデル (`KnowledgeModel`)

```python
class KnowledgeModel(BaseModel):
    source_urls: list[str]
    sitemap_urls: list[str]
    filenames: list[str]
    s3_urls: list[str]
```

#### 3.3.3 エージェントモデル (`AgentModel`)

```python
class AgentModel(BaseModel):
    tools: list[ToolModel]
```

### 3.4 DynamoDBテーブル設計

ボットテーブルは以下の構造を持ちます：

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
  - その他のボット設定属性

## 4. ボット作成と設定

### 4.1 ボット作成API (`routers/bot.py`)

```python
@router.post("/bot")
async def create_bot(
    request: Request,
    bot_input: BotInput,
    user: User = Depends(get_current_user),
) -> BotResponse:
    """新しいプライベートボットを作成"""
    # 作成権限のチェック
    check_creating_bot_allowed(user)
    
    # ボットIDの生成
    bot_id = str(uuid.uuid4())
    
    # ボットモデルの作成
    bot = BotModel(
        id=bot_id,
        owner_user_id=user.id,
        title=bot_input.title,
        description=bot_input.description,
        instruction=bot_input.instruction,
        create_time=time.time(),
        last_used_time=None,
        shared_scope="private",
        shared_status="active",
        allowed_cognito_groups=[],
        allowed_cognito_users=[],
        is_starred=False,
        generation_params=bot_input.generation_params or DEFAULT_GENERATION_PARAMS,
        agent=bot_input.agent or DEFAULT_AGENT,
        knowledge=bot_input.knowledge or DEFAULT_KNOWLEDGE,
        sync_status="NOT_STARTED",
        sync_status_reason="",
        sync_last_exec_id="",
        published_api_stack_name=None,
        published_api_datetime=None,
        published_api_codebuild_id=None,
        display_retrieved_chunks=bot_input.display_retrieved_chunks,
        conversation_quick_starters=bot_input.conversation_quick_starters or [],
        bedrock_knowledge_base=bot_input.bedrock_knowledge_base,
        bedrock_guardrails=bot_input.bedrock_guardrails,
        active_models=bot_input.active_models or DEFAULT_ACTIVE_MODELS,
        usage_stats=UsageStatsModel(
            total_conversations=0,
            total_messages=0,
            total_tokens=0,
            total_price=0,
        ),
    )
    
    # ボットの保存
    await save_bot(bot)
    
    # 知識ベースの処理開始（必要な場合）
    if bot.has_knowledge():
        # 知識ベース処理の開始
        # ...
    
    return BotResponse.from_bot_model(bot)
```

### 4.2 ボット更新API

```python
@router.patch("/bot/{bot_id}")
async def update_bot(
    request: Request,
    bot_id: str,
    bot_update: BotUpdate,
    user: User = Depends(get_current_user),
) -> BotResponse:
    """所有ボットのタイトル、指示、説明を更新"""
    # ボットの取得
    bot = await get_bot_by_id(bot_id, user.id)
    
    # 所有権チェック
    if bot.owner_user_id != user.id:
        raise RecordAccessNotAllowedError(f"Bot {bot_id} is not owned by user {user.id}")
    
    # フィールドの更新
    if bot_update.title is not None:
        bot.title = bot_update.title
    if bot_update.description is not None:
        bot.description = bot_update.description
    if bot_update.instruction is not None:
        bot.instruction = bot_update.instruction
    # その他のフィールド更新
    
    # ボットの保存
    await save_bot(bot)
    
    return BotResponse.from_bot_model(bot)
```

### 4.3 ボット設定のフロントエンド実装

ボット設定のフロントエンド実装は、以下のコンポーネントで構成されています：

1. **ボット作成フォーム**: タイトル、説明、指示などの基本情報を入力
2. **生成パラメータ設定**: 温度、トークン数などのパラメータを設定
3. **知識ベース設定**: ドキュメントのアップロードと検索設定
4. **エージェント設定**: ツールの選択と設定
5. **共有設定**: 共有スコープとアクセス権限の設定

## 5. ボット共有と権限管理

### 5.1 共有スコープの設定

```python
@router.patch("/bot/{bot_id}/visibility")
async def update_bot_visibility(
    request: Request,
    bot_id: str,
    visibility: BotVisibility,
    user: User = Depends(get_current_user),
) -> BotResponse:
    """ボットの可視性を切り替え"""
    # ボットの取得
    bot = await get_bot_by_id(bot_id, user.id)
    
    # 所有権チェック
    if bot.owner_user_id != user.id:
        raise RecordAccessNotAllowedError(f"Bot {bot_id} is not owned by user {user.id}")
    
    # 共有スコープの更新
    bot.shared_scope = visibility.shared_scope
    
    # 許可されたユーザーとグループの更新
    if visibility.allowed_cognito_users is not None:
        bot.allowed_cognito_users = visibility.allowed_cognito_users
    if visibility.allowed_cognito_groups is not None:
        bot.allowed_cognito_groups = visibility.allowed_cognito_groups
    
    # ボットの保存
    await save_bot(bot)
    
    return BotResponse.from_bot_model(bot)
```

### 5.2 共有スコープの種類

ボットの共有スコープには以下の種類があります：

- `private`: 所有者のみがアクセス可能
- `shared`: 特定のユーザーやグループとのみ共有
- `public`: すべてのユーザーがアクセス可能

```python
type_shared_scope = Literal["private", "shared", "public"]
```

### 5.3 アクセス権限のチェック

```python
def check_bot_access(bot: BotModel, user: User) -> bool:
    """ユーザーがボットにアクセスできるかどうかをチェック"""
    # 所有者は常にアクセス可能
    if bot.owner_user_id == user.id:
        return True
    
    # 公開ボットは誰でもアクセス可能
    if bot.shared_scope == "public":
        return True
    
    # 共有ボットの場合、許可されたユーザーやグループをチェック
    if bot.shared_scope == "shared":
        # ユーザーIDによるチェック
        if user.id in bot.allowed_cognito_users:
            return True
        
        # グループによるチェック
        for group in user.groups:
            if group in bot.allowed_cognito_groups:
                return True
    
    return False
```

## 6. ボットストア機能

### 6.1 ボットストアAPI (`routers/store.py`)

```python
@router.get("/store/search")
async def search_bots(
    request: Request,
    q: str = "",
    user: User = Depends(get_current_user),
) -> list[BotSummaryResponse]:
    """クエリ文字列でボットを検索"""
    # 検索クエリの準備
    search_query = {
        "query": {
            "bool": {
                "must": [
                    {
                        "multi_match": {
                            "query": q,
                            "fields": ["title^3", "description^2", "instruction"],
                            "fuzziness": "AUTO",
                        }
                    }
                ],
                "filter": [
                    {"term": {"shared_scope": "public"}}
                ]
            }
        },
        "sort": [{"_score": {"order": "desc"}}],
        "size": 50,
    }
    
    # OpenSearchで検索実行
    response = await opensearch_client.search(
        index=BOT_STORE_INDEX,
        body=search_query,
    )
    
    # 検索結果の処理
    bots = []
    for hit in response["hits"]["hits"]:
        source = hit["_source"]
        bots.append(
            BotSummaryResponse(
                id=source["id"],
                title=source["title"],
                description=source["description"],
                owner_user_id=source["owner_user_id"],
                owner_name=source.get("owner_name", ""),
                create_time=source["create_time"],
                has_knowledge=source["has_knowledge"],
                has_agent=source["has_agent"],
                usage_stats=UsageStatsModel(
                    total_conversations=source.get("total_conversations", 0),
                    total_messages=source.get("total_messages", 0),
                    total_tokens=source.get("total_tokens", 0),
                    total_price=source.get("total_price", 0),
                ),
            )
        )
    
    return bots
```

### 6.2 人気ボットの取得

```python
@router.get("/store/popular")
async def get_popular_bots(
    request: Request,
    user: User = Depends(get_current_user),
) -> list[BotSummaryResponse]:
    """人気のボットを取得"""
    # 人気ボットの検索クエリ
    search_query = {
        "query": {
            "bool": {
                "filter": [
                    {"term": {"shared_scope": "public"}}
                ]
            }
        },
        "sort": [{"total_conversations": {"order": "desc"}}],
        "size": 10,
    }
    
    # OpenSearchで検索実行
    response = await opensearch_client.search(
        index=BOT_STORE_INDEX,
        body=search_query,
    )
    
    # 検索結果の処理
    # ...
    
    return bots
```

### 6.3 ボットストアのインデックス更新

```python
async def update_bot_store_index(bot: BotModel, user_name: str | None = None):
    """ボットストアのインデックスを更新"""
    # 公開ボットのみインデックスに追加
    if bot.shared_scope != "public":
        # 既存のインデックスから削除
        try:
            await opensearch_client.delete(
                index=BOT_STORE_INDEX,
                id=bot.id,
                ignore=[404],
            )
        except Exception as e:
            logger.error(f"Failed to delete bot {bot.id} from bot store index: {e}")
        return
    
    # インデックスに追加するドキュメントの作成
    document = {
        "id": bot.id,
        "title": bot.title,
        "description": bot.description,
        "instruction": bot.instruction,
        "owner_user_id": bot.owner_user_id,
        "owner_name": user_name or "",
        "create_time": bot.create_time,
        "shared_scope": bot.shared_scope,
        "has_knowledge": bot.has_knowledge(),
        "has_agent": bot.is_agent_enabled(),
        "total_conversations": bot.usage_stats.total_conversations,
        "total_messages": bot.usage_stats.total_messages,
        "total_tokens": bot.usage_stats.total_tokens,
        "total_price": bot.usage_stats.total_price,
    }
    
    # OpenSearchにドキュメントを追加/更新
    try:
        await opensearch_client.index(
            index=BOT_STORE_INDEX,
            id=bot.id,
            body=document,
        )
    except Exception as e:
        logger.error(f"Failed to index bot {bot.id} to bot store index: {e}")
```

## 7. ボット使用統計

### 7.1 使用統計モデル

```python
class UsageStatsModel(BaseModel):
    total_conversations: int
    total_messages: int
    total_tokens: int
    total_price: float
```

### 7.2 使用統計の更新

```python
async def update_bot_usage_stats(
    bot_id: str,
    owner_user_id: str,
    tokens: int,
    price: float,
):
    """ボットの使用統計を更新"""
    # ボットの取得
    bot = await get_bot_by_id(bot_id, owner_user_id)
    
    # 統計の更新
    bot.usage_stats.total_messages += 1
    bot.usage_stats.total_tokens += tokens
    bot.usage_stats.total_price += price
    
    # 最終使用時間の更新
    bot.last_used_time = time.time()
    
    # ボットの保存
    await save_bot(bot)
    
    # ボットストアのインデックスも更新
    if bot.shared_scope == "public":
        await update_bot_store_index(bot)
```

## 8. ボットAPI公開機能

### 8.1 API公開モデル

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

### 8.2 API公開エンドポイント

```python
@router.post("/bot/{bot_id}/publication")
async def publish_bot(
    request: Request,
    bot_id: str,
    publication: BotPublicationInput,
    user: User = Depends(get_current_user),
) -> BotPublicationResponse:
    """ボットを公開"""
    # 公開権限のチェック
    check_publish_allowed(user)
    
    # ボットの取得
    bot = await get_bot_by_id(bot_id, user.id)
    
    # 所有権チェック
    if bot.owner_user_id != user.id:
        raise RecordAccessNotAllowedError(f"Bot {bot_id} is not owned by user {user.id}")
    
    # CloudFormationスタックの作成
    stack_name = f"BrChatApiBot-{bot_id}"
    
    # CodeBuildプロジェクトの開始
    codebuild_id = await start_api_publication_build(
        bot_id=bot_id,
        stack_name=stack_name,
        allowed_origins=publication.allowed_origins,
    )
    
    # ボットの公開情報を更新
    bot.published_api_stack_name = stack_name
    bot.published_api_datetime = int(time.time())
    bot.published_api_codebuild_id = codebuild_id
    
    # ボットの保存
    await save_bot(bot)
    
    return BotPublicationResponse(
        bot_id=bot_id,
        stack_name=stack_name,
        codebuild_id=codebuild_id,
        status="IN_PROGRESS",
        create_time=bot.published_api_datetime,
    )
```

## 9. エラーハンドリングとリカバリー

### 9.1 ボット作成・更新のエラーハンドリング

```python
@router.post("/bot")
async def create_bot(
    request: Request,
    bot_input: BotInput,
    user: User = Depends(get_current_user),
) -> BotResponse:
    try:
        # 作成権限のチェック
        check_creating_bot_allowed(user)
        
        # ボットの作成と保存
        # ...
        
        return BotResponse.from_bot_model(bot)
    except PermissionError as e:
        # 権限エラー
        raise HTTPException(status_code=403, detail=str(e))
    except ValueError as e:
        # 入力値エラー
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        # その他のエラー
        logger.error(f"Failed to create bot: {e}")
        raise HTTPException(status_code=500, detail="Failed to create bot")
```

### 9.2 知識ベース同期エラーの処理

```python
async def handle_knowledge_sync_error(bot_id: str, owner_user_id: str, error: str):
    """知識ベース同期エラーの処理"""
    try:
        # ボットの取得
        bot = await get_bot_by_id(bot_id, owner_user_id)
        
        # 同期状態の更新
        bot.sync_status = "FAILED"
        bot.sync_status_reason = error
        
        # ボットの保存
        await save_bot(bot)
    except Exception as e:
        logger.error(f"Failed to handle knowledge sync error for bot {bot_id}: {e}")
```

## 10. パフォーマンス最適化

### 10.1 ボット情報のキャッシング

```python
@lru_cache(maxsize=128)
async def get_cached_bot(bot_id: str, user_id: str) -> BotModel:
    """ボット情報をキャッシュする"""
    return await get_bot_by_id(bot_id, user_id)
```

### 10.2 ボットストアのインデックス最適化

```python
async def optimize_bot_store_index():
    """ボットストアのインデックスを最適化"""
    try:
        await opensearch_client.indices.forcemerge(
            index=BOT_STORE_INDEX,
            max_num_segments=1,
        )
    except Exception as e:
        logger.error(f"Failed to optimize bot store index: {e}")
```

## 11. セキュリティ対策

### 11.1 ボット作成権限の確認

```python
def check_creating_bot_allowed(user: User):
    """ボット作成権限をチェック"""
    if "CreatingBotAllowed" not in user.groups:
        raise PermissionError("User is not allowed to create bots")
```

### 11.2 API公開権限の確認

```python
def check_publish_allowed(user: User):
    """API公開権限をチェック"""
    if "PublishAllowed" not in user.groups:
        raise PermissionError("User is not allowed to publish APIs")
```

### 11.3 ボットアクセス権限の確認

```python
async def get_accessible_bot(bot_id: str, user: User) -> BotModel:
    """アクセス可能なボットを取得"""
    # ボットの取得
    bot = await get_bot_by_id(bot_id, user.id)
    
    # アクセス権限のチェック
    if not check_bot_access(bot, user):
        raise RecordAccessNotAllowedError(f"Bot {bot_id} is not accessible by user {user.id}")
    
    return bot
```

## 12. まとめ

Bedrock Chatのボット管理機能は、ユーザーがカスタムボットを作成、設定、共有、管理するための包括的な機能セットを提供します。この機能は、以下の主要なコンポーネントで構成されています：

1. **ボットモデルとデータ構造**: ボットの設定と状態を表すデータモデル
2. **ボット作成と設定**: ボットの基本情報、生成パラメータ、知識ベース、エージェント設定などを管理
3. **ボット共有と権限管理**: ボットの共有スコープとアクセス権限を管理
4. **ボットストア機能**: 共有ボットの検索と利用のためのインターフェース
5. **ボット使用統計**: ボットの使用状況を追跡
6. **ボットAPI公開機能**: ボットをAPIとして公開

これらの機能により、ユーザーは特定の目的や知識領域に特化したAIアシスタントを構築し、他のユーザーと共有することができます。また、ボットストア機能により、組織内での知識共有と再利用が促進されます。

セキュリティ面では、適切な権限管理とアクセス制御が実装されており、ボットの作成、共有、API公開などの操作が適切に制御されています。また、パフォーマンス最適化のための仕組みも導入されており、効率的なボット管理が可能となっています。
