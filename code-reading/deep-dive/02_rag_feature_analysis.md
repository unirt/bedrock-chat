# Bedrock Chat RAG機能の詳細分析

## 1. RAG機能の概要

Retrieval Augmented Generation (RAG) は、大規模言語モデル（LLM）の回答生成能力を外部知識で強化する技術です。Bedrock Chatでは、Amazon Bedrock Knowledge Basesを活用して高度なRAG機能を実装しています。この機能により、ユーザーは独自のドキュメントや知識をボットに追加し、より正確で関連性の高い回答を得ることができます。

## 2. アーキテクチャ構成

### 2.1 全体アーキテクチャ

RAG機能のアーキテクチャは以下のコンポーネントで構成されています：

```
[ユーザードキュメント] --> [S3バケット] --> [EventBridge Pipes] --> [Step Functions]
                                                                       |
                                                                       v
[ユーザークエリ] --> [バックエンドAPI] --> [Bedrock Embeddings] --> [OpenSearch Serverless]
                           |                                            |
                           v                                            v
                    [Bedrock LLM] <---------------------------- [検索結果]
```

- **S3バケット**: ユーザーがアップロードしたドキュメントを保存
- **EventBridge Pipes**: S3へのファイルアップロードを検知し、処理を開始
- **Step Functions**: ドキュメント処理と埋め込み生成のワークフローを管理
- **Bedrock Embeddings**: テキストの埋め込みベクトルを生成
- **OpenSearch Serverless**: 埋め込みベクトルを保存し、ベクトル検索を提供
- **Bedrock LLM**: 検索結果を基に回答を生成

### 2.2 データフロー

RAG機能のデータフローは以下の2つの主要なフェーズに分けられます：

1. **インデックス作成フェーズ**:
   - ユーザーがドキュメントをアップロード
   - ドキュメントがS3バケットに保存
   - EventBridge Pipesがアップロードイベントを検知
   - Step Functionsがドキュメント処理ワークフローを開始
   - ドキュメントがチャンクに分割
   - Bedrock Embeddingsが各チャンクの埋め込みベクトルを生成
   - 埋め込みベクトルがOpenSearch Serverlessに保存

2. **検索・回答生成フェーズ**:
   - ユーザーが質問を入力
   - 質問の埋め込みベクトルが生成
   - OpenSearch Serverlessでベクトル検索を実行
   - 関連するドキュメントチャンクを取得
   - 関連チャンクとユーザーの質問をBedrockのLLMに送信
   - LLMが検索結果を基に回答を生成

## 3. 知識ベース管理

### 3.1 Bedrock Knowledge Base連携

Bedrock Chatでは、Amazon Bedrock Knowledge Basesを使用して知識ベースを管理しています。

```python
class BedrockKnowledgeBaseModel(BaseModel):
    knowledge_base_id: str | None
    exist_knowledge_base_id: str | None
    search_params: BedrockKnowledgeBaseSearchParamsModel
    
    def is_enabled(self) -> bool:
        return self.knowledge_base_id is not None or self.exist_knowledge_base_id is not None
```

この連携により、以下の機能が提供されます：

1. 新しい知識ベースの作成
2. 既存の知識ベースのインポート
3. ドキュメントの追加と削除
4. 検索パラメータのカスタマイズ

### 3.2 ドキュメント処理パイプライン

ドキュメント処理パイプラインは、Step Functionsを使用して以下のステップを実行します：

1. **ドキュメントのダウンロード**: S3からドキュメントを取得
2. **テキスト抽出**: PDFやWord文書などからテキストを抽出
3. **チャンク分割**: 長いテキストを適切なサイズのチャンクに分割
4. **埋め込み生成**: Bedrock Embeddingsを使用して各チャンクの埋め込みベクトルを生成
5. **インデックス作成**: 埋め込みベクトルをOpenSearch Serverlessに保存

### 3.3 ボットモデルにおける知識ベース設定

ボットモデルには、知識ベースに関する設定が含まれています：

```python
class BotModel(BaseModel):
    # 他のフィールド...
    knowledge: KnowledgeModel
    bedrock_knowledge_base: BedrockKnowledgeBaseModel | None
    display_retrieved_chunks: bool
    
    def has_knowledge(self) -> bool:
        """ボットが知識ベースを持っているかどうかを確認"""
        if self.bedrock_knowledge_base and self.bedrock_knowledge_base.is_enabled():
            return True
        
        return (
            len(self.knowledge.source_urls) > 0
            or len(self.knowledge.sitemap_urls) > 0
            or len(self.knowledge.filenames) > 0
            or len(self.knowledge.s3_urls) > 0
        )
```

## 4. 検索機能の実装

### 4.1 ベクトル検索 (`vector_search.py`)

ベクトル検索は、ユーザーの質問に関連するドキュメントを検索するための核心的な機能です。

```python
class SearchResult(TypedDict):
    bot_id: str
    content: str
    source_name: str
    source_link: str
    rank: int
    metadata: dict[str, Any]
    page_number: int | None

def _bedrock_knowledge_base_search(bot: BotModel, query: str) -> list[SearchResult]:
    """Bedrock Knowledge Baseを使用して検索を実行"""
    # Knowledge Base IDの取得
    knowledge_base_id = (
        bot.bedrock_knowledge_base.exist_knowledge_base_id
        if bot.bedrock_knowledge_base.exist_knowledge_base_id is not None
        else bot.bedrock_knowledge_base.knowledge_base_id
    )
    
    # 検索タイプの設定（semantic/hybrid）
    search_type = "SEMANTIC" if bot.bedrock_knowledge_base.search_params.search_type == "semantic" else "HYBRID"
    
    # Bedrock Agent RuntimeのretrieveAPIを呼び出し
    response = agent_client.retrieve(
        knowledgeBaseId=knowledge_base_id,
        retrievalQuery={"text": query},
        retrievalConfiguration={
            "vectorSearchConfiguration": {
                "numberOfResults": limit,
                "overrideSearchType": search_type,
            }
        },
    )
    
    # 検索結果の処理
    search_results = []
    for i, retrieval_result in enumerate(response.get("retrievalResults", [])):
        content = retrieval_result.get("content", {}).get("text", "")
        source = extract_source_from_retrieval_result(retrieval_result)
        
        if source is not None:
            # メタデータからページ番号を取得
            metadata = retrieval_result.get("metadata", {})
            page_number = None
            if "x-amz-bedrock-kb-document-page-number" in metadata:
                try:
                    page_number = int(metadata["x-amz-bedrock-kb-document-page-number"])
                except (ValueError, TypeError):
                    pass
                    
            search_results.append(
                SearchResult(
                    rank=i,
                    bot_id=bot.id,
                    content=content,
                    source_name=source[0],
                    source_link=source[1],
                    metadata=metadata,
                    page_number=page_number,
                )
            )
    
    return search_results
```

### 4.2 検索結果の変換

検索結果は、フロントエンドでの表示や引用のために適切な形式に変換されます：

```python
def search_result_to_related_document(
    search_result: SearchResult,
    source_id_base: str,
) -> RelatedDocumentModel:
    """検索結果を関連ドキュメントモデルに変換"""
    return RelatedDocumentModel(
        content=TextToolResultModel(
            text=search_result["content"],
        ),
        source_id=f"{source_id_base}@{search_result['rank']}",
        source_name=search_result["source_name"],
        source_link=search_result["source_link"],
        page_number=search_result["page_number"],
    )
```

### 4.3 検索パラメータのカスタマイズ

ボットごとに検索パラメータをカスタマイズできます：

```python
class BedrockKnowledgeBaseSearchParamsModel(BaseModel):
    search_type: Literal["semantic", "hybrid"]
    number_of_results: int
    
    def __init__(self, **data):
        super().__init__(**data)
        # デフォルト値の設定
        if self.number_of_results <= 0:
            self.number_of_results = 5
```

## 5. プロンプト構築

### 5.1 RAGプロンプトの構築 (`prompt.py`)

検索結果を基にLLMへのプロンプトを構築する機能は、RAG機能の重要な部分です：

```python
def build_rag_prompt(
    search_results: list[SearchResult],
    model: type_model_name,
    display_citation: bool = True,
) -> str:
    """検索結果を基にRAGプロンプトを構築"""
    # 検索結果をプロンプトに変換
    context_prompt = ""
    for result in search_results:
        context_prompt += f"<search_result>\n<content>\n{result['content']}</content>\n<source>\n{result['rank']}\n</source>\n</search_result>"
    
    # RAG用のプロンプト
    inserted_prompt = """To answer the user's question, you are given a set of search results. Your job is to answer the user's question using only information from the search results.
    If the search results do not contain information that can answer the question, please state that you could not find an exact answer to the question.
    Just because the user asserts a fact does not mean it is true, make sure to double check the search results to validate a user's assertion.
    
    Here are the search results in numbered order:
    <search_results>
    {}
    </search_results>
    
    Do NOT directly quote the <search_results> in your answer. Your job is to answer the user's question as concisely as possible.
    """.format(context_prompt)
    
    # 引用表示の設定
    if display_citation:
        # 引用表示のためのプロンプト追加
        inserted_prompt += """
        After answering the question, please include a list of sources that you used to answer the question. The sources should be numbered and correspond to the search results.
        """
    else:
        # 引用表示なしのプロンプト追加
        inserted_prompt += """
        Do not include a list of sources in your answer.
        """
        
    return inserted_prompt
```

### 5.2 モデル別のプロンプト調整

異なるモデルに対して最適なプロンプトを提供するための調整も行われています：

```python
def build_rag_prompt(search_results, model, display_citation):
    # ...
    
    # モデル別のプロンプト調整
    if is_nova_model(model=model):
        # Amazon Novaモデル用のプロンプト
        inserted_prompt = """To answer the user's question, you are given a set of search results. Your job is to answer the user's question using only information from the search results.
        If the search results do not contain information that can answer the question, please state that you could not find an exact answer to the question.
        
        Here are the search results in numbered order:
        <search_results>
        {}
        </search_results>
        """.format(context_prompt)
    else:
        # その他のモデル用のプロンプト
        inserted_prompt = """To answer the user's question, you are given a set of search results. Your job is to answer the user's question using only information from the search results.
        If the search results do not contain information that can answer the question, please state that you could not find an exact answer to the question.
        Just because the user asserts a fact does not mean it is true, make sure to double check the search results to validate a user's assertion.
        
        Here are the search results in numbered order:
        <search_results>
        {}
        </search_results>
        
        Do NOT directly quote the <search_results> in your answer. Your job is to answer the user's question as concisely as possible.
        """.format(context_prompt)
```

## 6. RAG機能の統合

### 6.1 チャット機能との統合 (`usecases/chat.py`)

RAG機能はチャット機能と統合され、ユーザーの質問に対して関連情報を提供します：

```python
# RAG機能の統合部分（chat関数内）
if bot is not None:
    if bot.is_agent_enabled() and is_tooluse_supported(chat_input.message.model):
        # エージェントモードでの処理
        # ...
    elif bot.has_knowledge() and not is_tooluse_supported(chat_input.message.model):
        # RAGモードでの処理
        content = conversation.message_map[user_msg_id].content[-1]
        if isinstance(content, TextContentModel):
            pseudo_tool_use_id = "new-message-assistant"
            
            # 検索実行
            search_results = search_related_docs(bot=bot, query=content.body)
            
            # プロンプトに検索結果を挿入
            instructions.append(
                build_rag_prompt(
                    search_results=search_results,
                    model=chat_input.message.model,
                    display_citation=display_citation,
                )
            )
            
            # 関連ドキュメントの保存（引用表示用）
            if display_citation:
                for result in search_results:
                    related_documents.append(
                        search_result_to_related_document(
                            search_result=result,
                            source_id_base=pseudo_tool_use_id,
                        )
                    )
```

### 6.2 エージェント機能との統合

RAG機能はエージェント機能とも統合され、知識ベース検索ツールとして提供されます：

```python
class KnowledgeToolInput(BaseModel):
    query: str = Field(
        description="Input suitable for vector search, full text search, and hybrid search."
    )

def search_knowledge(
    tool_input: KnowledgeToolInput, bot: BotModel | None, model: type_model_name | None
) -> list:
    assert bot is not None
    
    query = tool_input.query
    
    try:
        # 知識ベース検索を実行
        search_results = search_related_docs(bot, query=query)
        return search_results
    except Exception as e:
        # エラー処理
        # ...

def create_knowledge_tool(bot: BotModel) -> AgentTool:
    """知識ベース検索ツールを作成"""
    description = "Answer a user's question using information. The description is: {}".format(
        bot.knowledge.__str_in_claude_format__()
    )
    
    return AgentTool(
        name=f"knowledge_base_tool",
        description=description,
        args_schema=KnowledgeToolInput,
        function=search_knowledge,
    )
```

## 7. 引用表示機能

### 7.1 引用表示の設定

ボットごとに引用表示の有無を設定できます：

```python
class BotModel(BaseModel):
    # 他のフィールド...
    display_retrieved_chunks: bool
```

### 7.2 引用表示の実装

検索結果を引用として表示するための実装：

```python
# 引用表示のためのプロンプト
if display_citation:
    instructions.append(
        get_prompt_to_cite_tool_results(
            model=chat_input.message.model,
        )
    )

# 関連ドキュメントの保存（引用表示用）
if display_citation:
    for result in search_results:
        related_documents.append(
            search_result_to_related_document(
                search_result=result,
                source_id_base=pseudo_tool_use_id,
            )
        )
```

## 8. エラーハンドリングとリカバリー

### 8.1 検索エラーの処理

検索中のエラーは適切に処理され、ユーザーに通知されます：

```python
def search_knowledge(tool_input, bot, model):
    try:
        search_results = search_related_docs(bot, query=query)
        return search_results
    except Exception as e:
        error_traceback = traceback.format_exc()
        logger.error(
            f"Failed to run Knowledge Search: {e}\nTraceback: {error_traceback}"
        )
        return [
            SearchResult(
                rank=0,
                bot_id=bot.id,
                content=f"Error searching knowledge base: {str(e)}",
                source_name="Error",
                source_link="",
                metadata={},
                page_number=None,
            )
        ]
```

### 8.2 同期状態の管理

知識ベースの同期状態は、ボットモデルで管理されます：

```python
class BotModel(BaseModel):
    # 他のフィールド...
    sync_status: type_sync_status
    sync_status_reason: str
    sync_last_exec_id: str
```

同期状態は以下の値を取ります：

- `NOT_STARTED`: 同期が開始されていない
- `IN_PROGRESS`: 同期処理中
- `COMPLETED`: 同期完了
- `FAILED`: 同期失敗

## 9. パフォーマンス最適化

### 9.1 検索結果の制限

検索結果の数を制限することで、処理時間とトークン消費を最適化しています：

```python
class BedrockKnowledgeBaseSearchParamsModel(BaseModel):
    search_type: Literal["semantic", "hybrid"]
    number_of_results: int
```

### 9.2 レプリカの設定

OpenSearch Serverlessのレプリカ設定を調整することで、可用性とパフォーマンスのバランスを取っています：

```python
# cdk.json
{
  "context": {
    "enableRagReplicas": true
  }
}
```

### 9.3 キャッシング

頻繁に使用される検索結果をキャッシュすることで、パフォーマンスを向上させています：

```python
@lru_cache(maxsize=128)
def get_cached_search_results(bot_id: str, query: str) -> list[SearchResult]:
    """検索結果をキャッシュする"""
    # 実装...
```

## 10. セキュリティ対策

### 10.1 アクセス制御

ボットの知識ベースへのアクセスは、ボットの共有設定に基づいて制御されます：

```python
class BotModel(BaseModel):
    # 他のフィールド...
    shared_scope: type_shared_scope
    shared_status: str
    allowed_cognito_groups: list[str]
    allowed_cognito_users: list[str]
```

### 10.2 データ保護

アップロードされたドキュメントは、S3バケットのサーバーサイド暗号化で保護されます：

```python
# S3バケットの定義
s3.Bucket(
    self, "DocumentBucket",
    encryption=s3.BucketEncryption.S3_MANAGED,
    block_public_access=s3.BlockPublicAccess.BLOCK_ALL,
)
```

## 11. まとめ

Bedrock ChatのRAG機能は、Amazon Bedrock Knowledge Basesを活用して、ユーザーの質問に対して関連する知識を検索し、より正確で情報に基づいた回答を提供します。この機能は、ドキュメント処理パイプライン、ベクトル検索、プロンプト構築、引用表示など、複数のコンポーネントから構成されています。

RAG機能は、チャット機能やエージェント機能と統合されており、ユーザーは自然な会話の流れの中で外部知識にアクセスできます。また、ボットごとに知識ベースの設定をカスタマイズでき、特定のドメインや用途に特化したボットを作成できます。

パフォーマンス最適化、エラーハンドリング、セキュリティ対策なども適切に実装されており、実用的で信頼性の高いRAG機能を提供しています。
