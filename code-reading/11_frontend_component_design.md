# Bedrock Chat フロントエンドのコンポーネント設計パターン

## 1. コンポーネント設計の概要

Bedrock Chatのフロントエンドでは、再利用可能で保守性の高いコンポーネント設計が採用されています。主に以下の設計パターンが使用されています：

1. **アトミックデザイン**: 小さな単位のコンポーネントから複雑なUIを構築
2. **コンポジションパターン**: コンポーネントの組み合わせによる機能拡張
3. **プレゼンテーショナル/コンテナパターン**: 表示と状態管理の分離
4. **カスタムフックによる関心の分離**: ロジックとUIの分離
5. **Storybookによるコンポーネント開発**: コンポーネントの独立した開発と文書化

## 2. アトミックデザインの適用

アトミックデザインは、UIコンポーネントを原子（Atoms）、分子（Molecules）、有機体（Organisms）、テンプレート（Templates）、ページ（Pages）の5つのレベルに分類する設計手法です。

### 2.1 原子（Atoms）

最小単位のUIコンポーネントです。ボタン、入力フィールド、アイコンなどが該当します。

```tsx
// Button.tsx - 原子コンポーネントの例
const Button = forwardRef<HTMLButtonElement, Props>((props, ref) => {
  return (
    <button
      ref={ref}
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
      onClick={(e) => {
        e.stopPropagation();
        e.preventDefault();
        props.onClick();
      }}
      disabled={props.disabled || props.loading}>
      {props.icon && !props.loading && (
        <div className="-ml-1 mr-2">{props.icon}</div>
      )}
      {props.loading && <PiSpinnerGap className="-ml-1 mr-2 animate-spin" />}
      {props.children}
      {props.rightIcon && <div className="ml-2">{props.rightIcon}</div>}
    </button>
  );
});
```

他の原子コンポーネントの例：
- `InputText.tsx`: テキスト入力フィールド
- `Skeleton.tsx`: ローディング表示用のスケルトン
- `Toggle.tsx`: トグルスイッチ
- `ButtonIcon.tsx`: アイコンボタン

### 2.2 分子（Molecules）

原子を組み合わせた、より複雑なUIコンポーネントです。

```tsx
// SearchTextBox.tsx - 分子コンポーネントの例
const SearchTextBox: React.FC<Props> = (props) => {
  return (
    <div className={twMerge('relative', props.className)}>
      <div className="pointer-events-none absolute inset-y-0 left-0 flex items-center pl-3">
        <PiMagnifyingGlass className="h-5 w-5 text-gray-400" />
      </div>
      <input
        type="text"
        className="block w-full rounded-lg border border-gray-300 bg-gray-50 p-2 pl-10 text-sm text-gray-900 focus:border-blue-500 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-700 dark:text-white dark:placeholder-gray-400 dark:focus:border-blue-500 dark:focus:ring-blue-500"
        placeholder={props.placeholder}
        value={props.value}
        onChange={(e) => props.onChange(e.target.value)}
      />
      {props.value && (
        <button
          type="button"
          className="absolute inset-y-0 right-0 flex items-center pr-3"
          onClick={() => props.onChange('')}>
          <PiX className="h-5 w-5 text-gray-400" />
        </button>
      )}
    </div>
  );
};
```

他の分子コンポーネントの例：
- `ButtonReasoning.tsx`: 推論機能を制御するボタン
- `ButtonCopy.tsx`: コピー機能付きボタン
- `UploadedAttachedFile.tsx`: アップロードファイル表示

### 2.3 有機体（Organisms）

分子や原子を組み合わせた、より複雑なUIコンポーネントです。

```tsx
// InputChatContent.tsx - 有機体コンポーネントの例
const InputChatContent: React.FC<Props> = (props) => {
  // 状態と関数
  // ...

  return (
    <div className={twMerge('flex flex-col', props.className)}>
      {/* ファイルアップロード領域 */}
      {/* テキスト入力エリア */}
      <div className="relative">
        <Textarea
          className="pr-24"
          placeholder={t('chat.input.placeholder')}
          value={content}
          onChange={setContent}
          onKeyDown={handleKeyDown}
          disabled={props.disabled}
          autoFocus
        />
        <div className="absolute bottom-2 right-2 flex items-center space-x-2">
          {/* モデル選択 */}
          {/* 送信ボタン */}
        </div>
      </div>
    </div>
  );
};
```

他の有機体コンポーネントの例：
- `ChatMessage.tsx`: チャットメッセージ表示
- `Drawer.tsx`: サイドナビゲーション
- `ModalDialog.tsx`: モーダルダイアログ
- `ListItemBot.tsx`: ボットリスト項目

### 2.4 テンプレート（Templates）

ページのレイアウトを定義するコンポーネントです。

```tsx
// AppContent.tsx - テンプレートコンポーネントの例
const AppContent: React.FC = () => {
  return (
    <div className="flex h-screen flex-col bg-white dark:bg-aws-ui-color-dark">
      <Header />
      <div className="flex flex-1 overflow-hidden">
        <Drawer />
        <main className="flex-1 overflow-hidden">
          <Outlet />
        </main>
      </div>
      <Snackbar />
    </div>
  );
};
```

### 2.5 ページ（Pages）

最終的なユーザーインターフェースを構成するコンポーネントです。

```tsx
// ChatPage.tsx - ページコンポーネントの例
const ChatPage: React.FC = () => {
  // 状態とフック
  // ...

  return (
    <div className="flex h-full flex-col">
      {/* ヘッダー部分 */}
      <div className="flex items-center justify-between border-b p-4">
        {/* ... */}
      </div>

      {/* メッセージ表示エリア */}
      <div
        id="messages"
        className="flex-1 overflow-y-auto p-4">
        {/* メッセージリスト */}
      </div>

      {/* 入力エリア */}
      <div className="border-t p-4">
        <InputChatContent
          disabled={postingMessage}
          onSubmit={handleSubmit}
          modelId={modelId}
          reasoningEnabled={reasoningEnabled}
          onChangeReasoningEnabled={setReasoningEnabled}
          supportReasoning={supportReasoning}
        />
      </div>
    </div>
  );
};
```

## 3. コンポジションパターン

Reactのコンポジションパターンを活用して、コンポーネントの再利用性と柔軟性を高めています。

### 3.1 子要素の受け渡し（Children Props）

```tsx
// Button.tsx
const Button = forwardRef<HTMLButtonElement, Props>((props, ref) => {
  return (
    <button /* ... */>
      {props.icon && !props.loading && (
        <div className="-ml-1 mr-2">{props.icon}</div>
      )}
      {props.children}
      {props.rightIcon && <div className="ml-2">{props.rightIcon}</div>}
    </button>
  );
});
```

### 3.2 特殊化されたコンポーネント

基本コンポーネントを拡張して、特殊化されたコンポーネントを作成しています。

```tsx
// ButtonIcon.tsx - Buttonを拡張したコンポーネント
const ButtonIcon: React.FC<Props> = (props) => {
  return (
    <Button
      className={twMerge('p-1', props.className)}
      icon={props.icon}
      onClick={props.onClick}
      text={props.text}
      outlined={props.outlined}
      disabled={props.disabled}>
      {props.children}
    </Button>
  );
};
```

## 4. プレゼンテーショナル/コンテナパターン

UIの表示（プレゼンテーショナル）とロジック（コンテナ）を分離するパターンを採用しています。

### 4.1 プレゼンテーショナルコンポーネント

見た目のみを担当するコンポーネントです。

```tsx
// ChatMessageMarkdown.tsx - プレゼンテーショナルコンポーネントの例
const ChatMessageMarkdown: React.FC<Props> = (props) => {
  return (
    <ReactMarkdown
      className={twMerge('prose dark:prose-invert', props.className)}
      remarkPlugins={[remarkGfm]}
      components={{
        code({ node, inline, className, children, ...props }) {
          // マークダウンのコード表示処理
        },
        // その他のマークダウン要素の処理
      }}>
      {props.children}
    </ReactMarkdown>
  );
};
```

### 4.2 コンテナコンポーネント

状態管理とロジックを担当するコンポーネントです。多くの場合、ページコンポーネントやカスタムフックがこの役割を果たします。

```tsx
// ChatPage.tsx - コンテナコンポーネントの例
const ChatPage: React.FC = () => {
  // 状態管理とロジック
  const {
    agentThinking,
    reasoningThinking,
    conversationError,
    postingMessage,
    newChat,
    postChat,
    messages,
    // ...
  } = useChat();

  // イベントハンドラ
  const handleSubmit = useCallback(
    (content: string, attachments: AttachmentType[] = []) => {
      // 送信処理
    },
    [/* 依存配列 */]
  );

  // UIレンダリング
  return (
    <div className="flex h-full flex-col">
      {/* コンポーネントの組み合わせ */}
    </div>
  );
};
```

## 5. カスタムフックによる関心の分離

ロジックをカスタムフックに抽出することで、UIコンポーネントとビジネスロジックを分離しています。

### 5.1 UIロジックのカスタムフック

```tsx
// useScroll.ts - UIロジックのカスタムフック
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

### 5.2 ビジネスロジックのカスタムフック

```tsx
// useChat.ts - ビジネスロジックのカスタムフック
const useChat = () => {
  // 状態管理
  const {
    chats,
    conversationId,
    setConversationId,
    // ...
  } = useChatState();

  // API呼び出し
  const conversationApi = useConversationApi();
  
  // メッセージ送信処理
  const postChat = (params: {
    content: string;
    enableReasoning: boolean;
    base64EncodedImages?: string[];
    attachments?: AttachmentType[];
    bot?: BotInputType;
  }) => {
    // 実装...
  };

  // その他の関数...

  return {
    agentThinking,
    reasoningThinking,
    conversationError,
    postingMessage,
    newChat,
    postChat,
    // その他の状態と関数...
  };
};
```

## 6. コンポーネントのテスト可能性

Storybookを使用して、コンポーネントを独立してテストおよび文書化しています。

### 6.1 Storybookの活用

```tsx
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import Button from './Button';
import { PiPlus } from 'react-icons/pi';

const meta = {
  title: 'Components/Button',
  component: Button,
  parameters: {
    layout: 'centered',
  },
  tags: ['autodocs'],
} satisfies Meta<typeof Button>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Default: Story = {
  args: {
    children: 'Button',
    onClick: () => console.log('clicked'),
  },
};

export const WithIcon: Story = {
  args: {
    children: 'Button',
    icon: <PiPlus />,
    onClick: () => console.log('clicked'),
  },
};
```

## 7. コンポーネント間の通信パターン

### 7.1 Props によるデータの受け渡し

最も基本的な通信方法として、親から子へのpropsの受け渡しを使用しています。

```tsx
// ChatPage.tsx
<ChatMessage
  chatContent={message}
  isStreaming={postingMessage && idx === messages.length - 1}
  relatedDocuments={relatedDocuments}
  onSubmitFeedback={handleSubmitFeedback}
/>
```

### 7.2 コールバック関数による子から親への通信

子コンポーネントから親コンポーネントへのデータの受け渡しには、コールバック関数を使用しています。

```tsx
// ChatMessage.tsx
const handleSubmitFeedback = (feedback: PutFeedbackRequest) => {
  if (props.onSubmitFeedback && props.chatContent) {
    props.onSubmitFeedback(props.chatContent.id, feedback);
    setIsFeedbackOpen(false);
  }
};
```

### 7.3 コンテキストによるグローバルな状態共有

アプリケーション全体で共有する必要がある状態には、Reactコンテキストを使用しています。

```tsx
// SnackbarProvider.tsx
export const SnackbarContext = createContext<{
  open: (message: string, severity?: AlertColor) => void;
}>({
  open: () => {},
});

export const SnackbarProvider: React.FC<Props> = ({ children }) => {
  // 実装...
  
  return (
    <SnackbarContext.Provider value={{ open }}>
      {children}
      {/* スナックバーコンポーネント */}
    </SnackbarContext.Provider>
  );
};
```

## 8. レスポンシブデザインの実装

Tailwind CSSを使用して、レスポンシブデザインを実装しています。

```tsx
// Drawer.tsx
<div
  className={twMerge(
    'fixed inset-y-0 left-0 z-40 flex h-screen w-64 flex-col overflow-y-auto border-r border-gray-200 bg-white pb-4 transition-transform dark:border-gray-700 dark:bg-gray-800 md:translate-x-0',
    isOpen ? 'translate-x-0' : '-translate-x-full'
  )}
>
  {/* ドロワーコンテンツ */}
</div>
```

## 9. アクセシビリティへの配慮

アクセシビリティを考慮したコンポーネント設計を行っています。

```tsx
// Button.tsx
<button
  ref={ref}
  className={/* ... */}
  onClick={(e) => {
    e.stopPropagation();
    e.preventDefault();
    props.onClick();
  }}
  disabled={props.disabled || props.loading}>
  {/* ボタンコンテンツ */}
</button>
```

## 10. まとめ

Bedrock Chatのフロントエンドコンポーネント設計は、以下の特徴を持っています：

1. **階層的な設計**: アトミックデザインに基づく階層的なコンポーネント設計
2. **再利用性**: コンポジションパターンによる高い再利用性
3. **関心の分離**: プレゼンテーショナル/コンテナパターンとカスタムフックによる関心の分離
4. **テスト可能性**: Storybookを活用した独立したコンポーネントのテストと文書化
5. **レスポンシブ対応**: Tailwind CSSを使用したレスポンシブデザイン
6. **アクセシビリティ**: アクセシビリティを考慮したコンポーネント設計

これらの設計パターンにより、Bedrock Chatは保守性が高く、拡張性のあるフロントエンドアーキテクチャを実現しています。
