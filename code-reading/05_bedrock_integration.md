# Bedrock Chat 生成AI連携実装

## 1. Amazon Bedrock 連携の概要

Bedrock Chatは、Amazon Bedrockサービスを活用して複数の大規模言語モデル（LLM）と連携しています。主な連携方法は以下の通りです：

1. **Converse API**: 会話型インターフェースを提供するBedrockのConverseAPIを使用
2. **ストリーミングレスポンス**: リアルタイムでの応答生成のためのストリーミング機能
3. **クロスリージョン推論**: 複数のAWSリージョンにまたがるモデル推論
4. **ツール使用**: エージェント機能のためのツール使用（Tool Use）機能
5. **ガードレール**: コンテンツフィルタリングのためのBedrockガードレール機能

## 2. サポートされているモデル

アプリケーションは以下のモデルをサポートしています：

- **Anthropic Claude**:
  - claude-v3-haiku
  - claude-v3-opus
  - claude-v3.5-sonnet
  - claude-v3.5-sonnet-v2
  - claude-v3.7-sonnet
  - claude-v3.5-haiku

- **Mistral AI**:
  - mistral-7b-instruct
  - mixtral-8x7b-instruct
  - mistral-large
  - mistral-large-2

- **Amazon Nova**:
  - amazon-nova-pro
  - amazon-nova-lite
  - amazon-nova-micro

- **DeepSeek**:
  - deepseek-r1

- **Meta Llama**:
  - llama3-3-70b-instruct
  - llama3-2-1b-instruct
  - llama3-2-3b-instruct
  - llama3-2-11b-instruct
  - llama3-2-90b-instruct

## 3. Bedrock API連携の実装

### 3.1 `bedrock.py` - Bedrockとの基本連携

このモジュールはBedrockサービスとの基本的な連携を担当します。主な機能は以下の通りです：

#### 3.1.1 モデル識別と設定

```python
def is_nova_model(model: type_model_name) -> bool:
    """Amazon Novaモデルかどうかを確認"""
    return "amazon-nova" in model

def is_deepseek_model(model: type_model_name) -> bool:
    """DeepSeekモデルかどうかを確認"""
    return "deepseek" in model

def is_llama_model(model: type_model_name) -> bool:
    """Meta Llamaモデルかどうかを確認"""
    return "llama" in model

def is_mistral(model: type_model_name) -> bool:
    """Mistralモデルかどうかを確認"""
    return "mistral" in model

def is_tooluse_supported(model: type_model_name) -> bool:
    """ツール使用をサポートしているかどうかを確認"""
    return model not in [
        "deepseek-r1",
        "llama3-2-1b-instruct",
        "llama3-2-3b-instruct",
        "",
    ]
```

#### 3.1.2 モデル固有のパラメータ設定

各モデルファミリーに対して、適切な推論パラメータを設定する関数が実装されています：

```python
def _prepare_nova_model_params(model, generation_params) -> Tuple[InferenceConfigurationTypeDef, Dict[str, Any]]:
    """Amazon Novaモデル用のパラメータを準備"""
    # ...

def _prepare_deepseek_model_params(model, generation_params) -> Tuple[InferenceConfigurationTypeDef, None]:
    """DeepSeekモデル用のパラメータを準備"""
    # ...

def _prepare_llama_model_params(model, generation_params) -> Tuple[InferenceConfigurationTypeDef, None]:
    """Meta Llamaモデル用のパラメータを準備"""
    # ...

def _prepare_mistral_model_params(model, generation_params) -> Tuple[InferenceConfigurationTypeDef, Dict[str, int] | None]:
    """Mistralモデル用のパラメータを準備"""
    # ...
```

#### 3.1.3 Converse API呼び出し

```python
def compose_args_for_converse_api(
    messages: list[SimpleMessageModel],
    model: type_model_name,
    instructions: list[str] = [],
    generation_params: GenerationParamsModel | None = None,
    guardrail: BedrockGuardrailsModel | None = None,
    grounding_source: GuardrailConverseContentBlockTypeDef | None = None,
    tools: dict[str, AgentTool] | None = None,
    stream: bool = True,
    enable_reasoning: bool = False,
) -> ConverseStreamRequestTypeDef:
    """Converse APIの呼び出し引数を構築"""
    # ...

@retry(exceptions=(BedrockThrottlingException,), tries=3, delay=60, backoff=2, jitter=(0, 2), logger=logger)
def call_converse_api(args: ConverseStreamRequestTypeDef) -> ConverseResponseTypeDef:
    """Converse APIを呼び出し、スロットリング時にリトライ"""
    # ...
```

#### 3.1.4 料金計算とモデルID解決

```python
def calculate_price(model: type_model_name, input_tokens: int, output_tokens: int, region: str = BEDROCK_REGION) -> float:
    """トークン数に基づいて料金を計算"""
    # ...

def get_model_id(model: type_model_name, enable_cross_region: bool = ENABLE_BEDROCK_CROSS_REGION_INFERENCE, bedrock_region: str = BEDROCK_REGION) -> str:
    """モデル名からBedrockのモデルIDを取得"""
    # ...
```

### 3.2 `stream.py` - ストリーミングレスポンス処理

このモジュールはBedrockのストリーミングレスポンスを処理します。主な機能は以下の通りです：

#### 3.2.1 ストリームハンドラ

```python
class ConverseApiStreamHandler:
    """Converse APIのストリームを処理するハンドラ"""
    
    def __init__(
        self,
        model: type_model_name,
        instructions: list[str] = [],
        generation_params: GenerationParamsModel | None = None,
        guardrail: BedrockGuardrailsModel | None = None,
        tools: dict[str, AgentTool] | None = None,
        on_stream: Callable[[str], None] | None = None,
        on_thinking: Callable[[OnThinking], None] | None = None,
        on_reasoning: Callable[[str], None] | None = None,
    ):
        # ...
    
    def run(
        self,
        messages: list[SimpleMessageModel],
        grounding_source: GuardrailConverseContentBlockTypeDef | None = None,
        message_for_continue_generate: SimpleMessageModel | None = None,
        enable_reasoning: bool = False,
    ) -> OnStopInput:
        """ストリーミングレスポンスを実行して処理"""
        # ...
```

#### 3.2.2 コンテンツ処理

```python
def _content_model_from_partial_content(content: _PartialTextContent | _PartialToolUseContent) -> ContentModel:
    """部分的なコンテンツからコンテンツモデルを作成"""
    # ...

def _content_model_to_partial_content(content: ContentModel) -> _PartialTextContent | _PartialToolUseContent | _PartialReasoningContent:
    """コンテンツモデルから部分的なコンテンツを作成"""
    # ...
```

## 4. チャット機能の実装 (`usecases/chat.py`)

### 4.1 会話の準備と処理

```python
def prepare_conversation(user: User, chat_input: ChatInput) -> tuple[str, ConversationModel, BotModel | None]:
    """会話を準備し、必要に応じて新しい会話を作成"""
    # ...

def trace_to_root(node_id: str | None, message_map: dict[str, MessageModel]) -> list[SimpleMessageModel]:
    """メッセージツリーをルートノードまでトレース"""
    # ...
```

### 4.2 チャット実行のメインフロー

```python
def chat(
    user: User,
    chat_input: ChatInput,
    on_stream: Callable[[str], None] | None = None,
    on_stop: Callable[[OnStopInput], None] | None = None,
    on_thinking: Callable[[OnThinking], None] | None = None,
    on_tool_result: Callable[[ToolRunResult], None] | None = None,
    on_reasoning: Callable[[str], None] | None = None,
) -> tuple[ConversationModel, MessageModel]:
    """チャットの実行とレスポンス処理"""
    # ...
```

このメソッドは以下のステップで実行されます：

1. 会話の準備（既存の会話を取得または新規作成）
2. ボットの取得と設定
3. ツールの設定（エージェント機能）
4. 関連ドキュメントの検索（RAG機能）
5. メッセージの構築
6. Bedrockへのリクエスト送信
7. ストリーミングレスポンスの処理
8. ツール使用の処理（必要に応じて）
9. 会話の保存
10. ボットの使用統計更新

### 4.3 会話タイトルの提案

```python
def propose_conversation_title(user_id: str, conversation_id: str, model: type_model_name = "claude-v3-haiku") -> str:
    """会話のタイトルを提案"""
    # ...
```

### 4.4 会話の取得と検索

```python
def fetch_conversation(user_id: str, conversation_id: str) -> Conversation:
    """会話を取得してスキーマに変換"""
    # ...

def search_conversations(query: str, user: User) -> list[ConversationSearchResult]:
    """キーワードで会話を検索"""
    # ...
```

## 5. クロスリージョン推論

アプリケーションはクロスリージョン推論をサポートしており、以下の実装があります：

```python
def get_model_id(
    model: type_model_name,
    enable_cross_region: bool = ENABLE_BEDROCK_CROSS_REGION_INFERENCE,
    bedrock_region: str = BEDROCK_REGION,
) -> str:
    # ...
    if enable_cross_region:
        if (
            bedrock_region in supported_regions
            and model in supported_regions[bedrock_region]["models"]
        ):
            region_prefix = supported_regions[bedrock_region]["area"]
            model_id = f"{region_prefix}.{base_model_id}"
            # ...
```

サポートされているリージョンとモデルの組み合わせは、`supported_regions`辞書で定義されています。

## 6. エラーハンドリングとリトライ

Bedrockへのリクエストには、スロットリングなどのエラーに対するリトライメカニズムが実装されています：

```python
@retry(
    exceptions=(BedrockThrottlingException,),
    tries=3,
    delay=60,
    backoff=2,
    jitter=(0, 2),
    logger=logger,
)
def call_converse_api(args: ConverseStreamRequestTypeDef) -> ConverseResponseTypeDef:
    # ...
```

また、ストリーミング中のエラーも適切に処理されます：

```python
# ストリーミング中のエラー処理
elif "modelStreamErrorException" in event:
    # ...
elif "throttlingException" in event:
    # ...
elif "internalServerException" in event:
    # ...
elif "serviceUnavailableException" in event:
    # ...
elif "validationException" in event:
    # ...
```

## 7. 料金計算

各モデルの使用料金は、入力トークンと出力トークンの数に基づいて計算されます：

```python
def calculate_price(
    model: type_model_name,
    input_tokens: int,
    output_tokens: int,
    region: str = BEDROCK_REGION,
) -> float:
    input_price = (
        BEDROCK_PRICING.get(region, {})
        .get(model, {})
        .get("input", BEDROCK_PRICING["default"][model]["input"])
    )
    output_price = (
        BEDROCK_PRICING.get(region, {})
        .get(model, {})
        .get("output", BEDROCK_PRICING["default"][model]["output"])
    )

    return input_price * input_tokens / 1000.0 + output_price * output_tokens / 1000.0
```

料金情報は`BEDROCK_PRICING`定数に定義されています。

## 8. 推論パラメータの設定

各モデルファミリーに対して、適切なデフォルト推論パラメータが設定されています：

```python
DEFAULT_GENERATION_CONFIG = {
    "max_tokens": 4096,
    "top_p": 0.9,
    "temperature": 0.7,
    "stop_sequences": [],
    "top_k": 250,
    "reasoning_params": {
        "budget_tokens": 1024,
    },
}

DEFAULT_DEEP_SEEK_GENERATION_CONFIG = {
    "max_tokens": 4096,
    "top_p": 0.9,
    "temperature": 0.7,
    "stop_sequences": [],
}

DEFAULT_LLAMA_GENERATION_CONFIG = {
    "max_tokens": 4096,
    "top_p": 0.9,
    "temperature": 0.7,
    "stop_sequences": [],
}

DEFAULT_MISTRAL_GENERATION_CONFIG = {
    "max_tokens": 4096,
    "top_p": 0.9,
    "temperature": 0.7,
    "stop_sequences": [],
}
```

これらのデフォルト値は、ユーザーがカスタムボットで設定したパラメータでオーバーライドできます。
