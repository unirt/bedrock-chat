# Bedrock Chat フロントエンドのパフォーマンス最適化

## 1. パフォーマンス最適化の概要

Bedrock Chatのフロントエンドでは、大量のメッセージデータや複雑なUIを扱うため、様々なパフォーマンス最適化技術が適用されています。主な最適化手法は以下の通りです：

1. **メモ化**: 不要な再計算や再レンダリングを防止
2. **遅延ロード**: 必要なときに必要なコンポーネントをロード
3. **状態管理の最適化**: 効率的な状態更新と選択的レンダリング
4. **仮想化**: 大量のリストデータの効率的な表示
5. **コード分割**: バンドルサイズの最適化
6. **レンダリングの最適化**: 効率的なUIの更新

## 2. メモ化による最適化

### 2.1 `useMemo` によるデータの計算の最適化

計算コストの高い処理を必要なときだけ実行するために、`useMemo`を使用しています。

```tsx
// ChatPage.tsx
const messages = useMemo(() => {
  return getMessages(conversationId, currentMessageId);
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, [conversationId, chats, currentMessageId]);

const tools = useMemo(() => {
  if (!chatContent?.thinkingLog) {
    return undefined;
  }
  return convertThinkingLogToAgentToolProps(chatContent.thinkingLog);
}, [chatContent?.thinkingLog]);
```

### 2.2 `useCallback` による関数の最適化

関数の再生成を防ぐために、`useCallback`を使用しています。

```tsx
// ChatPage.tsx
const handleSubmit = useCallback(
  (content: string, attachments: AttachmentType[] = []) => {
    postChat({
      content,
      enableReasoning: reasoningEnabled,
      attachments,
      bot: bot
        ? {
            botId: bot.id,
            hasKnowledge: bot.hasKnowledge,
            hasAgent: bot.hasAgent,
          }
        : undefined,
    });
  },
  [postChat, reasoningEnabled, bot]
);
```

### 2.3 `React.memo` によるコンポーネントの最適化

不要な再レンダリングを防ぐために、`React.memo`を使用しています。

```tsx
// Button.tsx
const Button = React.memo(forwardRef<HTMLButtonElement, Props>((props, ref) => {
  // コンポーネントの実装
}));
```

## 3. 遅延ロードと遅延実行

### 3.1 コンポーネントの遅延ロード

必要なときだけコンポーネントをロードするために、React.lazyとSuspenseを使用しています。

```tsx
// routes.tsx
const LazyComponent = React.lazy(() => import('./components/HeavyComponent'));

// 使用時
<Suspense fallback={<div>Loading...</div>}>
  <LazyComponent />
</Suspense>
```

### 3.2 テキストの遅延表示

テキストを徐々に表示するための遅延表示コンポーネントを実装しています。

```tsx
// LazyOutputText.tsx
const LazyOutputText: React.FC<Props> = (props) => {
  const [displayText, setDisplayText] = useState('');

  useEffect(() => {
    const functions: NodeJS.Timeout[] = [];
    props.text.split('').forEach((_, idx) => {
      functions.push(
        setTimeout(() => {
          setDisplayText(() => {
            return props.text.substring(0, idx + 1);
          });
        }, idx * 200)
      );
    });

    return () => {
      // 多重読み込み対策でクリアする
      functions.forEach((f) => {
        clearTimeout(f);
      });
    };
  }, [props.text]);

  return (
    <div className={`${props.className ?? ''} flex items-center`}>
      {displayText}
      {props.text !== displayText && (
        <PiRectangleFill className="rotate-90 -scale-y-75 animate-fastPulse text-xl" />
      )}
    </div>
  );
};
```

## 4. 状態管理の最適化

### 4.1 Zustandの選択的更新

Zustandの選択的更新機能を使用して、必要な状態のみを更新しています。

```tsx
// useChat.ts
const {
  chats,
  conversationId,
  setConversationId,
  postingMessage,
  setPostingMessage,
  // 必要な状態のみを選択
} = useChatState();
```

### 4.2 Immerによる不変性の効率的な管理

Immerを使用して、状態の不変性を効率的に管理しています。

```tsx
// useChatState.ts
pushMessage: (id, parentMessageId, currentMessageId, content) => {
  set(() => ({
    chats: produce(get().chats, (draft) => {
      // 状態の更新処理...
      if (draft[id] && parentMessageId && parentMessageId !== 'system') {
        draft[id][parentMessageId] = {
          ...draft[id][parentMessageId],
          children: [
            ...draft[id][parentMessageId].children,
            currentMessageId,
          ],
        };
        draft[id][currentMessageId] = {
          ...content,
          parent: parentMessageId,
          children: [],
        };
      } else {
        draft[id] = {
          [currentMessageId]: {
            ...content,
            children: [],
            parent: null,
          },
        };
      }
    }),
  }));
},
```

### 4.3 状態の正規化

会話データは、以下のように正規化されて保存されています：

```typescript
type ChatStateType = {
  [id: string]: MessageMap;
};

type MessageMap = {
  [messageId: string]: MessageModel;
};
```

これにより、特定のメッセージの更新や削除が効率的に行えます。

## 5. 効率的なレンダリング

### 5.1 条件付きレンダリング

必要なときだけコンポーネントをレンダリングすることで、パフォーマンスを向上させています。

```tsx
// ChatPage.tsx
{agentThinking.matches('thinking') && (
  <div className="agent-thinking-indicator">
    エージェントが考えています...
  </div>
)}
```

### 5.2 スケルトンローディング

データの読み込み中に、スケルトンUIを表示することで、ユーザー体験を向上させています。

```tsx
// Skeleton.tsx
const Skeleton: React.FC<Props> = (props) => {
  return (
    <div
      className={twMerge(
        `h-4 w-2/3 animate-pulse rounded bg-aws-font-color-light/20 dark:bg-aws-font-color-dark/20`,
        props.className
      )}></div>
  );
};

// 使用例
{loadingConversation ? (
  <Skeleton className="h-6 w-1/2" />
) : (
  <h1>{conversation.title}</h1>
)}
```

## 6. メッセージ処理の最適化

### 6.1 メッセージの効率的な変換

会話履歴を表示用の配列に変換する処理を最適化しています。

```tsx
// MessageUtils.ts
export const convertMessageMapToArray = (
  messageMap: MessageMap,
  currentMessageId: string
): DisplayMessageContent[] => {
  if (Object.keys(messageMap).length === 0) {
    return [];
  }

  const messageArray: DisplayMessageContent[] = [];
  let key: string | null = currentMessageId;
  let messageContent: MessageMap[string] = messageMap[key];

  // 効率的なメッセージ変換アルゴリズム
  // ...

  return messageArray;
};
```

### 6.2 マークダウンレンダリングの最適化

マークダウンのレンダリングを最適化するために、必要なプラグインのみを使用しています。

```tsx
// ChatMessageMarkdown.tsx
const ChatMessageMarkdown: React.FC<Props> = (props) => {
  // 関連ドキュメントの状態管理
  const { relatedDocuments, setRelatedDocument, resetRelatedDocument } =
    useRelatedDocumentsState();

  // 引用表示の処理
  const processChildren = useCallback(
    (children: ReactNode): ReactNode => {
      // 引用処理の最適化
      // ...
    },
    [props.relatedDocuments, relatedDocuments, props.messageId]
  );

  return (
    <ReactMarkdown
      className={twMerge('prose dark:prose-invert', props.className)}
      remarkPlugins={[remarkGfm, remarkBreaks, remarkMath]}
      rehypePlugins={[
        [rehypeExternalLinks, { target: '_blank' } as Options],
        rehypeKatex,
      ]}
      components={{
        // 最適化されたコンポーネント
        // ...
      }}>
      {props.children}
    </ReactMarkdown>
  );
};
```

## 7. スクロール処理の最適化

チャットメッセージのスクロール処理を最適化しています。

```tsx
// useScroll.ts
const useScroll = () => {
  const [disabled, setDisabled] = useState(false);

  useEffect(() => {
    const elem = document.getElementById('messages');
    if (!elem) {
      return;
    }
    const listener = () => {
      // 最下部までスクロールしている場合は、自動スクロールする
      if (elem.scrollTop + elem.clientHeight === elem.scrollHeight) {
        setDisabled(false);
      } else {
        setDisabled(true);
      }
    };
    elem.addEventListener('scroll', listener);

    return () => {
      elem.removeEventListener('scroll', listener);
    };
  }, []);

  return {
    scrollToTop: () => {
      document.getElementById('messages')?.scrollTo({
        top: 0,
        behavior: 'smooth',
      });
    },
    scrollToBottom: () => {
      if (!disabled) {
        document.getElementById('messages')?.scrollTo({
          top: document.getElementById('messages')?.scrollHeight,
          behavior: 'instant',
        });
      }
    },
  };
};
```

## 8. WebSocketストリーミングの最適化

### 8.1 チャンク処理

大きなメッセージを効率的に処理するために、チャンク処理を実装しています。

```tsx
// usePostMessageStreaming.ts
const chunkedPayloads: string[] = [];
const chunkCount = Math.ceil(payloadString.length / CHUNK_SIZE);
for (let i = 0; i < chunkCount; i++) {
  const start = i * CHUNK_SIZE;
  const end = Math.min(start + CHUNK_SIZE, payloadString.length);
  chunkedPayloads.push(payloadString.substring(start, end));
}
```

### 8.2 効率的なメッセージ処理

WebSocketから受信したメッセージを効率的に処理しています。

```tsx
// usePostMessageStreaming.ts
ws.onmessage = (message) => {
  try {
    // 特殊なメッセージの早期処理
    if (
      message.data === '' ||
      message.data === 'Message sent.' ||
      message.data.startsWith('{"message": "Endpoint request timed out",')
    ) {
      return;
    }
    
    // その他のメッセージ処理
    // ...
  } catch (e) {
    // エラー処理
  }
};
```

## 9. ビルドとバンドルの最適化

### 9.1 Viteによる高速ビルド

Viteを使用して、開発時の高速なHMRとプロダクションビルドの最適化を実現しています。

```javascript
// vite.config.ts
export default defineConfig({
  plugins: [react()],
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          // チャンク分割の設定
        },
      },
    },
  },
  // その他の設定
});
```

### 9.2 コード分割

ルートベースのコード分割を実装して、初期ロード時間を短縮しています。

```tsx
// routes.tsx
const ChatPage = React.lazy(() => import('./pages/ChatPage'));
const BotExplorePage = React.lazy(() => import('./pages/BotExplorePage'));
// その他のページコンポーネント

const routes = [
  {
    path: '/',
    element: <App />,
    children: [
      { path: '/', element: <Suspense fallback={<div>Loading...</div>}><ChatPage /></Suspense> },
      { path: '/bot/my', element: <Suspense fallback={<div>Loading...</div>}><BotExplorePage /></Suspense> },
      // その他のルート
    ],
  },
];
```

## 10. Tailwind CSSの最適化

### 10.1 JITモードの活用

Tailwind CSSのJIT（Just-In-Time）モードを活用して、必要なCSSのみを生成しています。

```javascript
// tailwind.config.js
module.exports = {
  mode: 'jit',
  // その他の設定
};
```

### 10.2 クラス名の最適化

`tailwind-merge`を使用して、クラス名の衝突を解決し、最適化しています。

```tsx
// Button.tsx
<button
  className={twMerge(
    'flex items-center justify-center whitespace-nowrap rounded-lg border p-1 px-3',
    props.text && 'border-0 dark:text-aws-font-color-dark',
    props.outlined && 'border-aws-squid-ink-light/50 dark:border-aws-font-color-gray/50 hover:bg-white dark:hover:bg-aws-ui-color-dark dark:text-aws-font-color-dark',
    !props.text &&
      !props.outlined &&
      'bg-aws-sea-blue-light dark:bg-aws-ui-color-dark dark:border-aws-ui-color-dark text-aws-font-color-white-light dark:text-aws-font-color-white-dark',
    props.disabled || props.loading ? 'opacity-30' : 'hover:brightness-75',
    props.className
  )}
  // その他のプロパティ
>
  {/* ボタンコンテンツ */}
</button>
```

## 11. まとめ

Bedrock Chatのフロントエンドパフォーマンス最適化は、以下の特徴を持っています：

1. **メモ化の活用**: `useMemo`、`useCallback`、`React.memo`による不要な再計算や再レンダリングの防止
2. **効率的な状態管理**: Zustandの選択的更新とImmerによる不変性の効率的な管理
3. **レンダリングの最適化**: 条件付きレンダリングとスケルトンローディングによるユーザー体験の向上
4. **データ処理の最適化**: メッセージの効率的な変換とマークダウンレンダリングの最適化
5. **WebSocketストリーミングの最適化**: チャンク処理と効率的なメッセージ処理
6. **ビルドとバンドルの最適化**: Viteによる高速ビルドとコード分割
7. **CSSの最適化**: Tailwind CSSのJITモードと`tailwind-merge`の活用

これらの最適化技術により、Bedrock Chatは大量のメッセージデータや複雑なUIを扱いながらも、高いパフォーマンスを維持しています。
