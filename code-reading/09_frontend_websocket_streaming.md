# Bedrock Chat WebSocketとストリーミング処理

## 1. WebSocketとストリーミングの概要

Bedrock Chatでは、AIモデルからのレスポンスをリアルタイムで表示するために、WebSocketを使用したストリーミング処理を実装しています。この機能により、ユーザーはAIの思考プロセスやレスポンスの生成をリアルタイムで確認できます。

### 1.1 ストリーミング処理の利点

- **即時フィードバック**: ユーザーは応答の生成をリアルタイムで確認できる
- **長い応答の処理**: 大きなレスポンスを小さなチャンクで受信できる
- **エージェント思考の可視化**: AIエージェントの思考プロセスを表示できる
- **推論プロセスの表示**: AIの推論ステップを表示できる

## 2. WebSocket接続の実装

### 2.1 `usePostMessageStreaming` フック

WebSocket接続とストリーミング処理の中核となるカスタムフックです。

```typescript
const usePostMessageStreaming = create<{
  post: (params: {
    input: PostMessageRequest;
    hasKnowledge?: boolean;
    dispatch: (completion: string) => void;
    thinkingDispatch: (event: AgentEvent) => void;
    reasoningDispatch: (event: ReasoningEvent) => void;
  }) => Promise<string>;
  errorDetail: string | null;
}>((set) => {
  return {
    errorDetail: null,
    post: async ({ input, dispatch, thinkingDispatch, reasoningDispatch }) => {
      // 実装...
    }
  };
});
```

### 2.2 WebSocket接続の確立

```typescript
const ws = new WebSocket(WS_ENDPOINT);

ws.onopen = () => {
  ws.send(
    JSON.stringify({
      step: PostStreamingStatus.START,
      token: token,
    })
  );
};
```

### 2.3 大きなメッセージのチャンク処理

API Gatewayの制限（32KB）を超えるメッセージを処理するために、大きなメッセージをチャンクに分割して送信します。

```typescript
// チャンク処理
const chunkedPayloads: string[] = [];
const chunkCount = Math.ceil(payloadString.length / CHUNK_SIZE);
for (let i = 0; i < chunkCount; i++) {
  const start = i * CHUNK_SIZE;
  const end = Math.min(start + CHUNK_SIZE, payloadString.length);
  chunkedPayloads.push(payloadString.substring(start, end));
}

// チャンクの送信
chunkedPayloads.forEach((chunk, index) => {
  ws.send(
    JSON.stringify({
      step: PostStreamingStatus.BODY,
      index,
      part: chunk,
    })
  );
});
```

### 2.4 メッセージ送信の完了通知

すべてのチャンクが送信されたことをサーバーに通知します。

```typescript
if (receivedCount === chunkedPayloads.length) {
  ws.send(
    JSON.stringify({
      step: PostStreamingStatus.END,
      token: token,
    })
  );
}
```

## 3. ストリーミングレスポンスの処理

### 3.1 メッセージタイプの処理

サーバーからのさまざまなタイプのメッセージを処理します。

```typescript
ws.onmessage = (message) => {
  try {
    // 特殊なメッセージの処理
    if (message.data === 'Session started.') {
      // セッション開始処理
      return;
    } else if (message.data === 'Message part received.') {
      // メッセージパート受信確認処理
      return;
    }

    const data = JSON.parse(message.data);

    if (data.status) {
      switch (data.status) {
        case PostStreamingStatus.AGENT_THINKING:
          // エージェント思考処理
          break;
        case PostStreamingStatus.AGENT_TOOL_RESULT:
          // ツール結果処理
          break;
        case PostStreamingStatus.AGENT_RELATED_DOCUMENT:
          // 関連ドキュメント処理
          break;
        case PostStreamingStatus.REASONING:
          // 推論処理
          break;
        case PostStreamingStatus.STREAMING:
          // 通常のストリーミングテキスト処理
          break;
        case PostStreamingStatus.STREAMING_END:
          // ストリーミング終了処理
          break;
        case PostStreamingStatus.ERROR:
          // エラー処理
          break;
      }
    }
  } catch (e) {
    // エラー処理
  }
};
```

### 3.2 エージェント思考の処理

エージェントの思考プロセスを処理し、UIに反映します。

```typescript
case PostStreamingStatus.AGENT_THINKING:
  if (completion.length > 0) {
    dispatch('');
    thinkingDispatch({
      type: 'thought',
      thought: completion,
    });
    completion = '';
  }
  Object.entries(data.log).forEach(([toolUseId, toolInfo]) => {
    const typedToolInfo = toolInfo as {
      name: string;
      input: { [key: string]: any };
    };
    thinkingDispatch({
      type: 'go-on',
      toolUseId: toolUseId,
      name: typedToolInfo.name,
      input: typedToolInfo.input,
    });
  });
  break;
```

### 3.3 ツール結果の処理

エージェントが使用したツールの結果を処理します。

```typescript
case PostStreamingStatus.AGENT_TOOL_RESULT:
  thinkingDispatch({
    type: 'tool-result',
    toolUseId: data.result.toolUseId,
    status: data.result.status,
  });
  break;
```

### 3.4 関連ドキュメントの処理

検索結果などの関連ドキュメントを処理します。

```typescript
case PostStreamingStatus.AGENT_RELATED_DOCUMENT:
  thinkingDispatch({
    type: 'related-document',
    toolUseId: data.result.toolUseId,
    relatedDocument: data.result.relatedDocument,
  });
  break;
```

### 3.5 推論処理

AIの推論プロセスを処理します。

```typescript
case PostStreamingStatus.REASONING:
  reasoningDispatch({
    type: 'write',
    content: data.completion,
  });
  break;
```

### 3.6 通常のストリーミングテキスト処理

AIからの通常のテキストレスポンスを処理します。

```typescript
case PostStreamingStatus.STREAMING:
  if (data.completion || data.completion === '') {
    completion += data.completion;
    dispatch(completion);
  }
  break;
```

## 4. エラーハンドリングと接続管理

### 4.1 エラー処理

WebSocketの通信エラーやサーバーからのエラーメッセージを処理します。

```typescript
case PostStreamingStatus.ERROR:
  ws.close();
  console.error(data);
  set({
    errorDetail:
      data.reason || i18next.t('error.predict.invalidResponse'),
  });
  throw new Error(
    data.reason || i18next.t('error.predict.invalidResponse')
  );
```

### 4.2 WebSocketエラーハンドリング

WebSocket接続自体のエラーを処理します。

```typescript
ws.onerror = (e) => {
  ws.close();
  console.error(e);
  reject(i18next.t('error.predict.general'));
};
```

### 4.3 接続終了処理

WebSocket接続が閉じられたときの処理を行います。

```typescript
ws.onclose = () => {
  resolve(completion);
};
```

## 5. ストリーミング処理とUIの連携

### 5.1 `useChat` フックとの統合

ストリーミング処理は`useChat`フックと統合され、UIコンポーネントに提供されます。

```typescript
const postChat = (params: {
  content: string;
  enableReasoning: boolean;
  base64EncodedImages?: string[];
  attachments?: AttachmentType[];
  bot?: BotInputType;
}) => {
  // ...

  const postPromise: Promise<string> = new Promise((resolve, reject) => {
    if (USE_STREAMING) {
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
        .then((message) => {
          resolve(message);
        })
        .catch((e) => {
          reject(e);
        });
    } else {
      // 非ストリーミング処理
    }
  });

  // ...
};
```

### 5.2 状態マシンとの連携

ストリーミングイベントは、XStateの状態マシンと連携して処理されます。

```typescript
thinkingDispatch({
  type: 'go-on',
  toolUseId: toolUseId,
  name: typedToolInfo.name,
  input: typedToolInfo.input,
});
```

## 6. パフォーマンス最適化

### 6.1 チャンク処理

大きなメッセージを32KBのチャンクに分割して送信することで、API Gatewayの制限を回避しています。

```typescript
const CHUNK_SIZE = 32 * 1024; //32KB
```

### 6.2 効率的なメッセージ処理

特殊なメッセージタイプを早期に処理することで、パフォーマンスを最適化しています。

```typescript
if (
  message.data === '' ||
  message.data === 'Message sent.' ||
  message.data.startsWith('{"message": "Endpoint request timed out",')
) {
  return;
}
```

## 7. セキュリティ対策

### 7.1 認証トークンの使用

WebSocket接続時に認証トークンを送信して、セキュリティを確保しています。

```typescript
const token = (await fetchAuthSession()).tokens?.idToken?.toString();
```

### 7.2 エラーメッセージの安全な処理

エラーメッセージは適切に処理され、センシティブな情報が漏洩しないようにしています。

```typescript
set({
  errorDetail:
    data.reason || i18next.t('error.predict.invalidResponse'),
});
```

## 8. WebSocketとストリーミング処理のまとめ

Bedrock ChatのフロントエンドにおけるWebSocketとストリーミング処理は、以下の特徴を持っています：

1. **リアルタイム応答**: ユーザーはAIの応答をリアルタイムで確認できる
2. **複数のメッセージタイプ**: 通常のテキスト、エージェント思考、ツール結果、推論など、さまざまなタイプのメッセージを処理
3. **大きなメッセージの処理**: チャンク処理により、API Gatewayの制限を回避
4. **状態マシンとの連携**: XStateの状態マシンと連携して、複雑な状態遷移を管理
5. **エラーハンドリング**: 接続エラーやサーバーエラーを適切に処理
6. **セキュリティ対策**: 認証トークンの使用とエラーメッセージの安全な処理

これらの機能により、Bedrock Chatは高度なリアルタイムチャット体験を提供しています。
