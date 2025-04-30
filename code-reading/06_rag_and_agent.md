# Bedrock Chat RAG機能とエージェント機能の実装

## 1. RAG (Retrieval Augmented Generation) 機能

### 1.1 概要

RAG機能は、ユーザーの質問に対して関連する知識ベースから情報を検索し、その情報を基にAIが回答を生成する機能です。Bedrock Chatでは、Amazon Bedrock Knowledge Basesを活用してRAG機能を実装しています。

### 1.2 知識ベース検索の実装 (`vector_search.py`)

#### 1.2.1 検索結果の型定義

```python
class SearchResult(TypedDict):
    bot_id: str
    content: str
    source_name: str
    source_link: str
    rank: int
    metadata: dict[str, Any]
    page_number: int | None
```

#### 1.2.2 Bedrock Knowledge Base検索

```python
def _bedrock_knowledge_base_search(bot: BotModel, query: str) -> list[SearchResult]:
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

#### 1.2.3 検索結果の変換

```python
def search_result_to_related_document(
    search_result: SearchResult,
    source_id_base: str,
) -> RelatedDocumentModel:
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

### 1.3 RAGプロンプトの構築 (`prompt.py`)

```python
def build_rag_prompt(
    search_results: list[SearchResult],
    model: type_model_name,
    display_citation: bool = True,
) -> str:
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
        # ...
    else:
        # 引用表示なしのプロンプト追加
        # ...
        
    return inserted_prompt
```

### 1.4 RAG機能の統合 (`usecases/chat.py`)

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
```

## 2. エージェント機能

### 2.1 概要

エージェント機能は、AIが外部ツールを使用して複雑なタスクを実行する機能です。Bedrock Chatでは、以下のようなツールをサポートしています：

1. 知識ベース検索ツール
2. インターネット検索ツール
3. Bedrock Agentツール

### 2.2 エージェントツールの基本構造 (`agents/tools/agent_tool.py`)

#### 2.2.1 AgentToolクラス

```python
class AgentTool(Generic[T]):
    def __init__(
        self,
        name: str,
        description: str,
        args_schema: type[T],
        function: Callable[
            [T, BotModel | None, type_model_name | None],
            ToolFunctionResult | list[ToolFunctionResult],
        ],
    ):
        self.name = name
        self.description = description
        self.args_schema = args_schema
        self.function = function
    
    def _generate_input_schema(self) -> dict[str, Any]:
        """Pydanticモデルをスキーマに変換"""
        return self.args_schema.model_json_schema(schema_generator=RemoveTitle)
    
    def to_converse_spec(self) -> ToolSpecificationTypeDef:
        """Converse APIのツール仕様に変換"""
        return ToolSpecificationTypeDef(
            name=self.name,
            description=self.description,
            inputSchema={"json": self._generate_input_schema()},
        )
    
    def run(
        self,
        tool_use_id: str,
        input: dict[str, JsonValue],
        model: type_model_name,
        bot: BotModel | None = None,
    ) -> ToolRunResult:
        """ツールを実行"""
        try:
            arg = self.args_schema.model_validate(input)
            res = self.function(arg, bot, model)
            # 結果を処理して返す
            # ...
        except Exception as e:
            # エラー処理
            # ...
```

### 2.3 知識ベース検索ツール (`agents/tools/knowledge.py`)

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

### 2.4 インターネット検索ツール (`agents/tools/internet_search.py`)

```python
class InternetSearchInput(BaseModel):
    query: str = Field(description="The query to search for on the internet.")
    country: str = Field(
        description="The country code you wish for search."
    )
    time_limit: str = Field(
        description="The time limit for the search. Options are 'd' (day), 'w' (week), 'm' (month), 'y' (year)."
    )

def _internet_search(
    tool_input: InternetSearchInput, bot: BotModel | None, model: type_model_name | None
) -> list:
    query = tool_input.query
    time_limit = tool_input.time_limit
    country = tool_input.country
    
    # ボットの設定に基づいて検索エンジンを選択
    if bot is None:
        return _search_with_duckduckgo(query, time_limit, country)
    
    internet_tool = next(
        (tool for tool in bot.agent.tools if isinstance(tool, InternetToolModel)),
        None,
    )
    
    if not internet_tool or internet_tool.search_engine == "duckduckgo":
        return _search_with_duckduckgo(query, time_limit, country)
    
    if internet_tool.search_engine == "firecrawl":
        # Firecrawlでの検索
        # ...
    
    # フォールバック
    return _search_with_duckduckgo(query, time_limit, country)

internet_search_tool = AgentTool(
    name="internet_search",
    description="Search the internet for information.",
    args_schema=InternetSearchInput,
    function=_internet_search,
)
```

### 2.5 ツール管理 (`agents/utils.py`)

```python
def get_available_tools() -> list[AgentTool]:
    """利用可能なツールのリストを取得"""
    tools: list[AgentTool] = []
    tools.append(internet_search_tool)
    tools.append(bedrock_agent_tool)
    return tools

def get_tools(bot: BotModel | None) -> Dict[str, AgentTool]:
    """ボットの設定に基づいてツールの辞書を取得"""
    tools: Dict[str, AgentTool] = {}
    
    # ボットがない場合や、エージェントが有効でない場合は空の辞書を返す
    if not bot or not bot.is_agent_enabled():
        return tools
    
    # 利用可能なツールを名前をキーとする辞書に変換
    available_tools = {tool.name: tool for tool in get_available_tools()}
    
    # ボットのツール設定に基づいてツールを取得
    for tool_config in bot.agent.tools:
        try:
            # ツールが利用可能でない場合はスキップ
            if tool_config.name not in available_tools:
                continue
                
            tools[tool_config.name] = available_tools[tool_config.name]
            
            # Bedrock Agentツールの説明を更新
            if (
                tool_config.name == "bedrock_agent"
                and tool_config.tool_type == "bedrock_agent"
                and tool_config.bedrockAgentConfig
                and tool_config.bedrockAgentConfig.agent_id
            ):
                # ...
        except Exception as e:
            # エラー処理
            # ...
    
    # ボットに知識ベースがある場合は知識ベース検索ツールを追加
    if bot.has_knowledge():
        knowledge_tool = create_knowledge_tool(bot=bot)
        tools[knowledge_tool.name] = knowledge_tool
    
    return tools
```

### 2.6 エージェント機能の統合 (`usecases/chat.py`)

```python
# エージェント機能の統合部分（chat関数内）
tools: Dict[str, AgentTool] = {}
if is_tooluse_supported(chat_input.message.model):
    tools = get_tools(bot)

# ...

stream_handler = ConverseApiStreamHandler(
    model=chat_input.message.model,
    instructions=instructions,
    generation_params=generation_params,
    guardrail=guardrail,
    tools=tools,  # ツールを設定
    on_stream=on_stream,
    on_thinking=on_thinking,
    on_reasoning=on_reasoning,
)

thinking_log: list[SimpleMessageModel] = []
while True:
    result = stream_handler.run(
        messages=messages,
        grounding_source=grounding_source,
        message_for_continue_generate=message_for_continue_generate,
        enable_reasoning=chat_input.enable_reasoning,
    )
    
    message = result["message"]
    stop_reason = result["stop_reason"]
    
    # ...
    
    if stop_reason != "tool_use":  # ツール使用が完了した場合
        # メッセージ処理
        # ...
        break
    
    # ツール使用の処理
    tool_use_message = SimpleMessageModel.from_message_model(message=message)
    # ...
    
    thinking_log.append(tool_use_message)
    
    tool_use_contents = [
        content
        for content in tool_use_message.content
        if isinstance(content, ToolUseContentModel)
    ]
    
    run_results: list[ToolRunResult] = []
    for content in tool_use_contents:
        tool = tools[content.body.name]
        run_result = tool.run(
            tool_use_id=content.body.tool_use_id,
            input=content.body.input,
            model=chat_input.message.model,
            bot=bot,
        )
        run_results.append(run_result)
        
        if run_result["status"] == "success":
            related_documents.extend(run_result["related_documents"])
        
        if on_tool_result:
            on_tool_result(run_result)
    
    # ツール結果をメッセージに追加
    tool_result_message = SimpleMessageModel(
        role="user",
        content=[
            ToolResultContentModel.from_tool_run_result(
                run_result=result,
                model=chat_input.message.model,
                display_citation=display_citation,
            )
            for result in run_results
        ],
    )
    messages.append(tool_result_message)
    thinking_log.append(tool_result_message)
```

## 3. RAGとエージェント機能の連携

### 3.1 知識ベースとエージェントの統合

Bedrock Chatでは、知識ベースとエージェント機能を統合する2つの方法があります：

1. **RAGモード**: ツール使用をサポートしていないモデルの場合、検索結果をプロンプトに直接挿入
2. **エージェントモード**: ツール使用をサポートするモデルの場合、知識ベース検索をツールとして提供

```python
if bot is not None:
    if bot.is_agent_enabled() and is_tooluse_supported(chat_input.message.model):
        # エージェントモード
        if bot.has_knowledge():
            # 知識ベース検索ツールを追加
            knowledge_tool = create_knowledge_tool(bot=bot)
            tools[knowledge_tool.name] = knowledge_tool
            
        if display_citation:
            instructions.append(
                get_prompt_to_cite_tool_results(
                    model=chat_input.message.model,
                )
            )
    elif bot.has_knowledge() and not is_tooluse_supported(chat_input.message.model):
        # RAGモード
        content = conversation.message_map[user_msg_id].content[-1]
        if isinstance(content, TextContentModel):
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
```

### 3.2 引用表示の設定

検索結果やツール結果を引用表示するための設定があります：

```python
# ボットの設定から引用表示の有無を取得
display_citation = bot is not None and bot.display_retrieved_chunks

# 引用表示のためのプロンプト
if display_citation:
    instructions.append(
        get_prompt_to_cite_tool_results(
            model=chat_input.message.model,
        )
    )
```

## 4. ストリーミング処理とツール使用

### 4.1 ストリーミングハンドラ (`stream.py`)

```python
class ConverseApiStreamHandler:
    def run(
        self,
        messages: list[SimpleMessageModel],
        grounding_source: GuardrailConverseContentBlockTypeDef | None = None,
        message_for_continue_generate: SimpleMessageModel | None = None,
        enable_reasoning: bool = False,
    ) -> OnStopInput:
        # Bedrockへのリクエスト作成
        args = compose_args_for_converse_api(
            messages=messages,
            model=self.model,
            instructions=self.instructions,
            generation_params=self.generation_params,
            guardrail=self.guardrail,
            grounding_source=grounding_source,
            tools=self.tools,
            enable_reasoning=enable_reasoning,
        )
        
        # ストリーミングレスポンスの処理
        response = client.converse_stream(**args)
        
        # イベント処理
        for event in response["stream"]:
            if "contentBlockDelta" in event:
                # テキスト、ツール使用、推論コンテンツの処理
                # ...
            elif "contentBlockStop" in event:
                # ツール使用の完了処理
                # ...
            elif "messageStop" in event:
                # メッセージの完了処理
                # ...
```

### 4.2 ツール使用のループ処理

```python
# ツール使用のループ処理（chat関数内）
while True:
    result = stream_handler.run(...)
    
    if stop_reason != "tool_use":  # ツール使用が完了した場合
        break
    
    # ツール使用の処理
    tool_use_message = SimpleMessageModel.from_message_model(message=message)
    # ...
    
    # ツールの実行
    run_results = []
    for content in tool_use_contents:
        tool = tools[content.body.name]
        run_result = tool.run(...)
        run_results.append(run_result)
    
    # ツール結果をメッセージに追加
    tool_result_message = SimpleMessageModel(...)
    messages.append(tool_result_message)
    
    # 次のイテレーションへ
```

## 5. モデル別の対応

### 5.1 モデル別のツール使用サポート

```python
def is_tooluse_supported(model: type_model_name) -> bool:
    """ツール使用をサポートしているかどうかを確認"""
    return model not in [
        "deepseek-r1",
        "llama3-2-1b-instruct",
        "llama3-2-3b-instruct",
        "",
    ]
```

### 5.2 モデル別のプロンプト調整

```python
def build_rag_prompt(search_results, model, display_citation):
    # ...
    
    # モデル別のプロンプト調整
    if is_nova_model(model=model):
        # Amazon Novaモデル用のプロンプト
        # ...
    else:
        # その他のモデル用のプロンプト
        # ...
```

## 6. エラーハンドリング

```python
def search_knowledge(tool_input, bot, model):
    try:
        search_results = search_related_docs(bot, query=query)
        return search_results
    except Exception as e:
        error_traceback = traceback.format_exc()
        logger.error(
            f"Failed to run AnswerWithKnowledgeTool: {e}\nTraceback: {error_traceback}"
        )
        raise e

def run(self, tool_use_id, input, model, bot):
    try:
        arg = self.args_schema.model_validate(input)
        res = self.function(arg, bot, model)
        # ...
    except Exception as e:
        return ToolRunResult(
            tool_use_id=tool_use_id,
            status="error",
            related_documents=[
                _function_result_to_related_document(
                    tool_name=self.name,
                    res=str(e),
                    source_id_base=tool_use_id,
                )
            ],
        )
```
