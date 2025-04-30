# Bedrock Chat エージェント機能とXState

## 1. エージェント機能の概要

Bedrock Chatのエージェント機能は、AIが外部ツールを使用して複雑なタスクを実行する機能です。この機能により、AIは検索、知識ベースの参照、計算などのツールを使用して、より正確で有用な回答を提供できます。

エージェント機能の状態管理には、XStateという状態マシンライブラリが使用されています。XStateを使用することで、複雑な状態遷移を宣言的に定義し、エージェントの動作を予測可能で管理しやすいものにしています。

## 2. XStateによる状態管理

### 2.1 XStateの基本概念

XStateは、有限状態マシン（FSM）のパラダイムに基づいた状態管理ライブラリです。主な概念は以下の通りです：

- **状態（State）**: システムが取りうる状態
- **イベント（Event）**: 状態遷移のトリガーとなるアクション
- **遷移（Transition）**: あるイベントによって、ある状態から別の状態への変化
- **アクション（Action）**: 遷移時に実行される副作用
- **コンテキスト（Context）**: 状態マシン内で共有されるデータ

### 2.2 エージェント状態マシンの定義

```typescript
export const agentThinkingState = setup({
  types: {
    context: {} as {
      tools: AgentToolsProps[];
      relatedDocuments: RelatedDocument[];
    },
    events: {} as AgentEvent,
  },
  actions: {
    reset: assign({
      tools: [],
      relatedDocuments: [],
    }),
    updateThought: assign({
      tools: ({ context, event }) => produce(context.tools, draft => {
        if (event.type === 'thought') {
          // 思考内容の更新処理
        }
      }),
    }),
    addTool: assign({
      tools: ({ context, event }) => produce(context.tools, draft => {
        if (event.type === 'go-on') {
          // ツールの追加処理
        }
      }),
    }),
    updateToolResult: assign({
      tools: ({ context, event }) => produce(context.tools, draft => {
        if (event.type === 'tool-result') {
          // ツール結果の更新処理
        }
      }),
    }),
    addRelatedDocument: assign(({ context, event }) => produce(context, draft => {
      if (event.type === 'related-document') {
        // 関連ドキュメントの追加処理
      }
    })),
    close: assign({
      tools: [],
      relatedDocuments: [],
    }),
  },
}).createMachine({
  context: {
    tools: [],
    relatedDocuments: [],
    areAllToolsSuccessful: false,
  },
  initial: 'sleeping',
  states: {
    sleeping: {
      on: {
        wakeup: {
          actions: 'reset',
          target: 'thinking',
        },
      },
    },
    thinking: {
      on: {
        'thought': {
          actions: 'updateThought',
        },
        'go-on': {
          actions: 'addTool',
        },
        'tool-result': {
          actions: ['updateToolResult'],
        },
        'related-document': {
          actions: ['addRelatedDocument'],
        },
        goodbye: {
          actions: 'close',
          target: 'leaving',
        },
      },
    },
    leaving: {
      after: {
        2500: { target: 'sleeping' },
      },
    },
  },
});
```

### 2.3 状態の定義

エージェント状態マシンは、以下の3つの状態を持っています：

1. **sleeping**: 待機状態。エージェントがアクティブでない状態
2. **thinking**: 思考中状態。エージェントがアクティブで、思考やツール使用を行っている状態
3. **leaving**: 終了中状態。エージェントが処理を完了し、待機状態に戻る前の一時的な状態

### 2.4 イベントの定義

エージェント状態マシンは、以下のイベントを処理します：

```typescript
export type AgentEvent =
  | { type: 'wakeup' }
  | {
      type: 'thought';
      thought: string;
    }
  | {
      type: 'go-on';
      toolUseId: string;
      name: string;
      input: { [key: string]: any };
    }
  | {
      type: 'tool-result';
      toolUseId: string;
      status: AgentToolState;
    }
  | {
      type: 'related-document';
      toolUseId: string;
      relatedDocument: RelatedDocument;
    }
  | { type: 'goodbye' };
```

1. **wakeup**: エージェントを起動するイベント
2. **thought**: エージェントの思考内容を更新するイベント
3. **go-on**: エージェントがツールを使用するイベント
4. **tool-result**: ツールの実行結果を受け取るイベント
5. **related-document**: 関連ドキュメントを追加するイベント
6. **goodbye**: エージェントを終了するイベント

### 2.5 アクションの定義

状態遷移時に実行されるアクションは、以下のように定義されています：

1. **reset**: コンテキストをリセット
2. **updateThought**: 思考内容を更新
3. **addTool**: ツールを追加
4. **updateToolResult**: ツール結果を更新
5. **addRelatedDocument**: 関連ドキュメントを追加
6. **close**: コンテキストをクリア

## 3. 推論機能の状態管理

エージェント機能と並行して、AIの推論プロセスを管理するための状態マシンも実装されています。

### 3.1 推論状態マシンの定義

```typescript
export const reasoningState = setup({
  types: {
    context: {} as ReasoningContext,
    events: {} as ReasoningEvent,
  },
  actions: {
    reset: assign({
      content: '',
    }),
    appendContent: assign({
      content: ({ context, event }) =>
        event.type === 'write'
          ? context.content + event.content
          : context.content,
    }),
    clear: assign({
      content: '',
    }),
  },
}).createMachine({
  id: 'reasoning',
  context: {
    content: '',
  },
  initial: 'inactive',
  states: {
    inactive: {
      on: {
        start: {
          actions: 'reset',
          target: 'active',
        },
      },
    },
    active: {
      on: {
        write: {
          actions: 'appendContent',
        },
        end: {
          actions: 'clear',
          target: 'inactive',
        },
      },
    },
  },
});
```

### 3.2 推論状態の定義

推論状態マシンは、以下の2つの状態を持っています：

1. **inactive**: 非アクティブ状態。推論が行われていない状態
2. **active**: アクティブ状態。推論が行われている状態

### 3.3 推論イベントの定義

```typescript
export type ReasoningEvent =
  | { type: 'start' }
  | { type: 'write'; content: string }
  | { type: 'end' };
```

1. **start**: 推論を開始するイベント
2. **write**: 推論内容を追加するイベント
3. **end**: 推論を終了するイベント

## 4. XStateとReactの統合

### 4.1 `useMachine` フックの使用

XStateの状態マシンは、`@xstate/react`パッケージの`useMachine`フックを使用してReactコンポーネントと統合されています。

```typescript
const useChat = () => {
  const [agentThinking, agentSend] = useMachine(agentThinkingState);
  const [reasoningThinking, reasoningSend] = useMachine(reasoningState);
  
  // ...
  
  return {
    agentThinking,
    reasoningThinking,
    // ...
  };
};
```

### 4.2 イベントのディスパッチ

状態マシンにイベントをディスパッチするには、`send`関数を使用します。

```typescript
agentSend({ type: 'wakeup' });
reasoningSend({ type: 'start' });

postStreaming({
  input,
  dispatch: (c: string) => {
    editMessage(conversationId, NEW_MESSAGE_ID.ASSISTANT, c);
  },
  thinkingDispatch: (event) => {
    agentSend(event);
  },
  reasoningDispatch: (event) => {
    reasoningSend(event);
  },
})
```

## 5. エージェントツールの実装

### 5.1 ツールの基本構造

エージェントが使用するツールは、`AgentTool`クラスとして実装されています。

```typescript
export class AgentTool<T> {
  constructor(
    public name: string,
    public description: string,
    public args_schema: type<T>,
    public function: (args: T, bot?: BotModel, model?: type_model_name) => ToolFunctionResult | ToolFunctionResult[],
  ) {}

  to_converse_spec(): ToolSpecificationTypeDef {
    return {
      name: this.name,
      description: this.description,
      inputSchema: { json: this._generate_input_schema() },
    };
  }

  run(
    tool_use_id: string,
    input: Record<string, JsonValue>,
    model: type_model_name,
    bot?: BotModel,
  ): ToolRunResult {
    try {
      const args = this.args_schema.model_validate(input);
      const result = this.function(args, bot, model);
      // 結果の処理
      return {
        tool_use_id,
        status: 'success',
        related_documents: [...],
      };
    } catch (e) {
      // エラー処理
      return {
        tool_use_id,
        status: 'error',
        related_documents: [...],
      };
    }
  }
}
```

### 5.2 知識ベースツール

知識ベースを検索するためのツールです。

```typescript
function create_knowledge_tool(bot: BotModel): AgentTool {
  const description = `Answer a user's question using information. The description is: ${bot.knowledge.__str_in_claude_format__()}`;
  
  return new AgentTool(
    'knowledge_base_tool',
    description,
    KnowledgeToolInput,
    search_knowledge,
  );
}

function search_knowledge(
  tool_input: KnowledgeToolInput,
  bot: BotModel | null,
  model: type_model_name | null,
): ToolFunctionResult[] {
  assert(bot != null);
  const query = tool_input.query;
  
  try {
    const search_results = search_related_docs(bot, query);
    return search_results;
  } catch (e) {
    // エラー処理
    throw e;
  }
}
```

### 5.3 インターネット検索ツール

インターネットを検索するためのツールです。

```typescript
const internet_search_tool = new AgentTool(
  'internet_search',
  'Search the internet for information.',
  InternetSearchInput,
  _internet_search,
);

function _internet_search(
  tool_input: InternetSearchInput,
  bot: BotModel | null,
  model: type_model_name | null,
): ToolFunctionResult[] {
  const query = tool_input.query;
  const time_limit = tool_input.time_limit;
  const country = tool_input.country;
  
  // 検索エンジンの選択と検索の実行
  if (bot?.agent.tools.some(t => t instanceof InternetToolModel)) {
    // ボットの設定に基づいて検索エンジンを選択
    const internet_tool = bot.agent.tools.find(t => t instanceof InternetToolModel);
    if (internet_tool?.search_engine === 'firecrawl') {
      return _search_with_firecrawl(query, api_key, country, max_results);
    }
  }
  
  // デフォルトはDuckDuckGo
  return _search_with_duckduckgo(query, time_limit, country);
}
```

## 6. エージェント機能のUIとの連携

### 6.1 エージェントツールの表示

エージェントツールとその結果は、専用のUIコンポーネントで表示されます。

```tsx
<AgentToolList
  tools={tools}
  relatedDocuments={relatedDocuments}
  onClickRelatedDocument={handleClickRelatedDocument}
/>
```

### 6.2 エージェント状態の監視

エージェントの状態は、`useChat`フックを通じてコンポーネントに提供され、UIの表示を制御します。

```tsx
const {
  agentThinking,
  reasoningThinking,
  // ...
} = useChat();

// エージェントの状態に基づいてUIを表示
{agentThinking.matches('thinking') && (
  <div className="agent-thinking-indicator">
    エージェントが考えています...
  </div>
)}
```

## 7. エージェント機能の利点と課題

### 7.1 利点

1. **複雑なタスクの処理**: 外部ツールを使用して複雑なタスクを処理できる
2. **透明性の向上**: AIの思考プロセスを可視化できる
3. **正確性の向上**: 最新の情報や専門知識にアクセスできる
4. **拡張性**: 新しいツールを追加することで機能を拡張できる

### 7.2 課題

1. **複雑な状態管理**: 複数のツールと状態遷移を管理する必要がある
2. **エラーハンドリング**: ツールの実行エラーを適切に処理する必要がある
3. **UX設計**: 複雑なプロセスをユーザーに分かりやすく表示する必要がある
4. **パフォーマンス**: 複数のツールを使用する場合のレスポンス時間の管理

## 8. まとめ

Bedrock Chatのエージェント機能とXStateによる状態管理は、以下の特徴を持っています：

1. **宣言的な状態管理**: XStateを使用して複雑な状態遷移を宣言的に定義
2. **ツールの抽象化**: `AgentTool`クラスによるツールの抽象化と統一インターフェース
3. **リアルタイム表示**: WebSocketストリーミングとの連携によるリアルタイム表示
4. **拡張性**: 新しいツールを追加することで機能を拡張可能
5. **型安全性**: TypeScriptによる型安全な実装

これらの特徴により、Bedrock Chatは高度なエージェント機能を提供し、ユーザーはAIの思考プロセスを理解しながら、より正確で有用な回答を得ることができます。
