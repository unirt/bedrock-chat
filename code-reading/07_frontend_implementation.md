# Bedrock Chat フロントエンド実装

## 1. 技術スタック

Bedrock Chatのフロントエンドは、以下の技術スタックで構築されています：

- **フレームワーク**: React
- **言語**: TypeScript
- **ルーティング**: React Router
- **スタイリング**: Tailwind CSS
- **状態管理**: Zustand
- **国際化**: i18next
- **認証**: AWS Amplify
- **ビルドツール**: Vite

## 2. アプリケーション構造

### 2.1 ディレクトリ構造

```
frontend/
├── src/
│   ├── @types/         # 型定義
│   ├── assets/         # 静的アセット
│   ├── components/     # 再利用可能なコンポーネント
│   ├── constants/      # 定数
│   ├── features/       # 機能別モジュール
│   ├── hooks/          # カスタムフック
│   ├── i18n/           # 国際化リソース
│   ├── layouts/        # レイアウトコンポーネント
│   ├── pages/          # ページコンポーネント
│   ├── providers/      # コンテキストプロバイダー
│   ├── utils/          # ユーティリティ関数
│   ├── App.tsx         # アプリケーションのルートコンポーネント
│   ├── main.tsx        # エントリーポイント
│   └── routes.tsx      # ルート定義
```

### 2.2 主要なファイル

- **App.tsx**: アプリケーションのルートコンポーネント。認証とレイアウトを設定
- **routes.tsx**: アプリケーションのルーティング設定
- **main.tsx**: アプリケーションのエントリーポイント

## 3. ルーティング

アプリケーションは、React Routerを使用して以下のルートを定義しています：

```typescript
const rootChildren = [
  { path: '/', element: <ChatPage /> },
  { path: '/bot/my', element: <BotExplorePage /> },
  { path: '/bot/recently-used', element: <BotRecentlyUsedPage /> },
  { path: '/bot/starred', element: <BotStarredPage /> },
  { path: '/bot/discover', element: <BotDiscoverPage /> },
  { path: '/bot/new', element: <BotKbEditPage /> },
  { path: '/bot/edit/:botId', element: <BotKbEditPage /> },
  { path: '/bot/api-settings/:botId', element: <BotApiSettingsPage /> },
  { path: '/bot/:botId', element: <ChatPage /> },
  { path: '/conversations', element: <ConversationHistoryPage /> },
  { path: '/admin/shared-bot-analytics', element: <AdminSharedBotAnalyticsPage /> },
  { path: '/admin/api-management', element: <AdminApiManagementPage /> },
  { path: '/admin/bot/:botId', element: <AdminBotManagementPage /> },
  { path: '/:conversationId', element: <ChatPage /> },
  { path: '*', element: <NotFound /> },
];
```

## 4. 認証

### 4.1 認証プロバイダー

アプリケーションは、AWS Amplifyを使用して認証を実装しています。カスタム認証プロバイダーのオプションもサポートしています。

```typescript
// App.tsx
const customProviderEnabled =
  import.meta.env.VITE_APP_CUSTOM_PROVIDER_ENABLED === 'true';
const socialProviderFromEnv = import.meta.env.VITE_APP_SOCIAL_PROVIDERS?.split(
  ','
).filter(validateSocialProvider);

// ...

return (
  <ErrorBoundary fallback={<ErrorFallback />}>
    {customProviderEnabled ? (
      <AuthCustom>
        <AppContent />
      </AuthCustom>
    ) : (
      <Authenticator.Provider>
        <AuthAmplify socialProviders={socialProviderFromEnv}>
          <AppContent />
        </AuthAmplify>
      </Authenticator.Provider>
    )}
  </ErrorBoundary>
);
```

### 4.2 認証コンポーネント

- **AuthAmplify**: AWS Amplify認証を使用するコンポーネント
- **AuthCustom**: カスタム認証プロバイダーを使用するコンポーネント

## 5. 主要なページコンポーネント

### 5.1 ChatPage

チャットインターフェースを提供する中心的なページコンポーネントです。

```typescript
const ChatPage: React.FC = () => {
  const { t } = useTranslation();
  const navigate = useNavigate();
  const { open: openSnackbar } = useSnackbar();
  const { errorDetail } = usePostMessageStreaming();
  const { isAdmin } = useLoginUser();
  const { pinBot, unpinBot } = useBotPinning();

  const {
    agentThinking,
    reasoningThinking,
    conversationError,
    postingMessage,
    newChat,
    postChat,
    messages,
    conversationId,
    setConversationId,
    hasError,
    retryPostChat,
    setCurrentMessageId,
    regenerate,
    continueGenerate,
    getPostedModel,
    loadingConversation,
    getShouldContinue,
    relatedDocuments,
    giveFeedback,
    reasoningEnabled,
    setReasoningEnabled,
    supportReasoning,
  } = useChat();

  // ...
};
```

### 5.2 BotExplorePage

ユーザーが作成したボットを表示・管理するページです。

### 5.3 BotDiscoverPage

ボットストアからボットを探索するページです。

### 5.4 BotKbEditPage

ボットの作成・編集を行うページです。

### 5.5 AdminPages

管理者向けの各種ページコンポーネントです。

## 6. 主要なコンポーネント

### 6.1 ChatMessage

チャットメッセージを表示するコンポーネントです。テキスト、画像、添付ファイル、ツール使用結果などの様々なコンテンツタイプをサポートしています。

```typescript
const ChatMessage: React.FC<Props> = (props) => {
  const { t } = useTranslation();
  const [isEdit, setIsEdit] = useState(false);
  const [changedContent, setChangedContent] = useState('');
  const [isFeedbackOpen, setIsFeedbackOpen] = useState(false);

  // ...

  return (
    <div className={twMerge('flex flex-col', props.className)}>
      {/* メッセージコンテンツの表示 */}
      {/* フィードバックボタン */}
      {/* 関連ドキュメント */}
      {/* エージェントツール */}
      {/* 推論表示 */}
    </div>
  );
};
```

### 6.2 InputChatContent

ユーザーがメッセージを入力するためのコンポーネントです。テキスト入力、ファイルアップロード、モデル選択などの機能を提供します。

### 6.3 Drawer

サイドバーナビゲーションを提供するコンポーネントです。会話履歴、ボットリスト、設定などにアクセスできます。

### 6.4 SwitchBedrockModel

使用するBedrockモデルを選択するためのコンポーネントです。

## 7. カスタムフック

### 7.1 useChat

チャット機能の中核となるカスタムフックです。メッセージの送信、受信、会話の管理などを担当します。

```typescript
const useChat = () => {
  const { conversationId, setConversationId, ... } = useChatState();
  const { postMessage, ... } = useConversationApi();
  const { postMessageStreaming, ... } = usePostMessageStreaming();
  
  // ...

  const postChat = useCallback(
    async (content: string, attachments: AttachmentType[] = []) => {
      // メッセージ送信処理
    },
    [...]
  );

  // ...

  return {
    conversationId,
    setConversationId,
    postChat,
    // ...その他の状態と関数
  };
};
```

### 7.2 useConversation

会話データの取得と管理を行うカスタムフックです。

### 7.3 useBot

ボットデータの取得と管理を行うカスタムフックです。

### 7.4 usePostMessageStreaming

ストリーミングレスポンスを処理するカスタムフックです。WebSocketを使用してリアルタイムでメッセージを受信します。

## 8. 状態管理

アプリケーションは、Zustandを使用して状態を管理しています。主な状態ストアは以下の通りです：

### 8.1 useChatState

チャット関連の状態を管理するストアです。

```typescript
const useChatState = create<{
  conversationId: string;
  setConversationId: (s: string) => void;
  postingMessage: boolean;
  setPostingMessage: (b: boolean) => void;
  chats: ChatStateType;
  setMessages: (id: string, messageMap: MessageMap) => void;
  // ...その他の状態と関数
}>((set, get) => {
  return {
    conversationId: '',
    setConversationId: (s) => {
      set(() => {
        return {
          conversationId: s,
        };
      });
    },
    // ...その他の実装
  };
});
```

### 8.2 その他のストア

- **useBotStore**: ボット関連の状態を管理
- **useSnackbar**: 通知メッセージの表示を管理
- **useLoginUser**: ログインユーザー情報を管理

## 9. 国際化

アプリケーションは、i18nextを使用して国際化をサポートしています。

```typescript
// i18n/index.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';

// 言語リソース
import en from './locales/en.json';
import ja from './locales/ja.json';
// ...その他の言語

i18n
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    resources: {
      en: { translation: en },
      ja: { translation: ja },
      // ...その他の言語
    },
    fallbackLng: 'en',
    interpolation: {
      escapeValue: false,
    },
  });

export default i18n;
```

## 10. API通信

### 10.1 APIフック

バックエンドAPIとの通信を行うカスタムフックが実装されています：

- **useConversationApi**: 会話関連のAPI
- **useBotApi**: ボット関連のAPI
- **useAdminApi**: 管理者機能関連のAPI

### 10.2 WebSocket通信

ストリーミングレスポンスを受信するためのWebSocket通信が実装されています。

```typescript
// hooks/usePostMessageStreaming.ts
const usePostMessageStreaming = () => {
  // WebSocketの接続と管理
  // ストリーミングメッセージの処理
  // ...
};
```

## 11. 特殊機能の実装

### 11.1 エージェント機能

エージェント機能は、XStateを使用して状態遷移を管理しています。

```typescript
// features/agent/xstates/agentThink.ts
export const agentThinkingState = createMachine({
  id: 'agentThinking',
  initial: 'idle',
  states: {
    idle: {
      on: {
        START: 'thinking',
      },
    },
    thinking: {
      on: {
        TOOL_USE: 'toolUse',
        DONE: 'done',
      },
    },
    toolUse: {
      on: {
        TOOL_RESULT: 'thinking',
      },
    },
    done: {
      type: 'final',
    },
  },
});
```

### 11.2 推論機能

推論機能も同様にXStateを使用して実装されています。

```typescript
// features/reasoning/xstates/reasoningState.ts
export const reasoningState = createMachine({
  id: 'reasoning',
  initial: 'idle',
  states: {
    idle: {
      on: {
        START: 'thinking',
      },
    },
    thinking: {
      on: {
        UPDATE: 'thinking',
        DONE: 'done',
      },
    },
    done: {
      type: 'final',
    },
  },
});
```

### 11.3 知識ベース機能

知識ベース機能は、ファイルアップロードとメタデータ管理を含む複雑な機能です。

```typescript
// features/knowledgeBase/pages/BotKbEditPage.tsx
const BotKbEditPage: React.FC = () => {
  // ボット情報の取得と管理
  // ファイルアップロード処理
  // 知識ベース設定
  // ...
};
```

## 12. エラーハンドリング

アプリケーションは、React Error Boundaryを使用してエラーをキャッチし、適切なフォールバックUIを表示します。

```typescript
// App.tsx
return (
  <ErrorBoundary fallback={<ErrorFallback />}>
    {/* アプリケーションコンテンツ */}
  </ErrorBoundary>
);
```

また、APIエラーは各APIフック内で処理され、スナックバーで通知されます。

## 13. レスポンシブデザイン

アプリケーションは、Tailwind CSSを使用してレスポンシブデザインを実装しています。モバイル、タブレット、デスクトップの各画面サイズに対応しています。

```typescript
// components/Drawer.tsx
<div
  className={twMerge(
    'fixed inset-y-0 left-0 z-40 flex h-screen w-64 flex-col overflow-y-auto border-r border-gray-200 bg-white pb-4 transition-transform dark:border-gray-700 dark:bg-gray-800 md:translate-x-0',
    isOpen ? 'translate-x-0' : '-translate-x-full'
  )}
>
  {/* ドロワーコンテンツ */}
</div>
```
