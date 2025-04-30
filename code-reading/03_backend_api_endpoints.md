# Bedrock Chat バックエンドAPIエンドポイント一覧

## 1. 会話管理 API (`/conversation`)

会話の作成、取得、検索、削除などを行うためのエンドポイント群です。

| メソッド | エンドポイント | 説明 |
|---------|--------------|------|
| GET | `/health` | ヘルスチェック用エンドポイント |
| POST | `/conversation` | 新しいメッセージを送信し、AIからの応答を取得 |
| GET | `/conversation/{conversation_id}` | 特定の会話履歴を取得 |
| DELETE | `/conversation/{conversation_id}` | 特定の会話を削除 |
| GET | `/conversations` | すべての会話メタデータを取得 |
| DELETE | `/conversations` | すべての会話を削除 |
| GET | `/conversations/search` | キーワードで会話を検索 |
| PATCH | `/conversation/{conversation_id}/title` | 会話のタイトルを更新 |
| GET | `/conversation/{conversation_id}/proposed-title` | 会話のタイトル候補を取得 |
| PUT | `/conversation/{conversation_id}/{message_id}/feedback` | メッセージにフィードバックを送信 |
| GET | `/conversation/{conversation_id}/related-documents` | 会話に関連するドキュメントを取得 |
| GET | `/conversation/{conversation_id}/related-documents/{source_id}` | 特定の関連ドキュメントを取得 |

## 2. ボット管理 API (`/bot`)

カスタムボットの作成、取得、更新、削除などを行うためのエンドポイント群です。

| メソッド | エンドポイント | 説明 |
|---------|--------------|------|
| POST | `/bot` | 新しいプライベートボットを作成 |
| PATCH | `/bot/{bot_id}` | 所有ボットのタイトル、指示、説明を更新 |
| PATCH | `/bot/{bot_id}/starred` | ボットのスター状態を更新 |
| PATCH | `/bot/{bot_id}/visibility` | ボットの可視性を切り替え |
| GET | `/bot` | すべてのボットを取得（プライベート/混合、スター付き、制限付き） |
| GET | `/bot/pinned` | すべてのピン留めされたボットを取得 |
| GET | `/bot/private/{bot_id}` | 特定のプライベートボットを取得 |
| GET | `/bot/summary/{bot_id}` | ボットの概要を取得 |
| DELETE | `/bot/{bot_id}` | ボットを削除（所有または共有） |
| GET | `/bot/{bot_id}/presigned-url` | ボット用の署名付きURLを取得 |
| DELETE | `/bot/{bot_id}/uploaded-file` | アップロードされたファイルを削除 |
| DELETE | `/bot/{bot_id}/recently-used` | 最近使用したボットの履歴から削除 |
| GET | `/bot/{bot_id}/agent/available-tools` | ボットで利用可能なツールを取得 |

## 3. 管理者 API (`/admin`)

管理者向けの機能を提供するエンドポイント群です。

| メソッド | エンドポイント | 説明 |
|---------|--------------|------|
| GET | `/admin/published-bots` | 公開されているすべてのボットを取得 |
| GET | `/admin/public-bots` | すべての公開ボットを取得（使用状況を含む） |
| GET | `/admin/users` | すべてのユーザーを取得（使用状況を含む） |
| GET | `/admin/bot/public/{bot_id}` | 特定の公開ボットを取得 |
| PATCH | `/admin/bot/{bot_id}/pushed` | ボットのピン留め状態を更新 |

## 4. API公開 API (`/bot/{bot_id}/publication`)

ボットをAPIとして公開するための機能を提供するエンドポイント群です。

| メソッド | エンドポイント | 説明 |
|---------|--------------|------|
| POST | `/bot/{bot_id}/publication` | ボットを公開 |
| GET | `/bot/{bot_id}/publication` | ボットの公開情報を取得 |
| DELETE | `/bot/{bot_id}/publication` | ボットの公開を削除 |
| GET | `/bot/{bot_id}/publication/api-key/{api_key_id}` | ボット公開用APIキーを取得 |
| POST | `/bot/{bot_id}/publication/api-key` | ボット公開用APIキーを作成 |
| DELETE | `/bot/{bot_id}/publication/api-key/{api_key_id}` | ボット公開用APIキーを削除 |

## 5. ボットストア API (`/store`)

ボットストア機能を提供するエンドポイント群です。

| メソッド | エンドポイント | 説明 |
|---------|--------------|------|
| GET | `/store/search` | クエリ文字列でボットを検索 |
| GET | `/store/popular` | 人気のボットを取得 |
| GET | `/store/pickup` | ピックアップボットを取得 |

## 6. ユーザー管理 API (`/user`)

ユーザー情報の取得や検索を行うためのエンドポイント群です。

| メソッド | エンドポイント | 説明 |
|---------|--------------|------|
| GET | `/user/search` | ユーザーを検索 |
| GET | `/user/group/search` | ユーザーグループを検索 |
| GET | `/user/{user_id}` | 特定のユーザー情報を取得 |

## 7. 公開API (`/published_api`)

公開されたAPIのエンドポイント群です。

| メソッド | エンドポイント | 説明 |
|---------|--------------|------|
| GET | `/health` | ヘルスチェック用エンドポイント |
| POST | `/conversation` | 新しいメッセージを送信（ボットID不要） |
| GET | `/conversation/{conversation_id}` | 特定の会話履歴を取得 |
| GET | `/conversation/{conversation_id}/{message_id}` | 特定のメッセージを取得 |

## 8. 認証と認可

APIエンドポイントへのアクセスは、以下の方法で制御されています：

1. **認証**: Amazon Cognitoを使用したJWTトークンベースの認証
2. **認可**: 以下の権限チェック関数を使用
   - `check_admin`: 管理者権限のチェック
   - `check_creating_bot_allowed`: ボット作成権限のチェック
   - `check_publish_allowed`: API公開権限のチェック

## 9. エラーハンドリング

以下のエラーに対して適切なHTTPステータスコードが返されます：

- `RecordNotFoundError`: 404 Not Found
- `FileNotFoundError`: 404 Not Found
- `RecordAccessNotAllowedError`: 403 Forbidden
- `ValueError`, `TypeError`, `AssertionError`: 400 Bad Request
- `PermissionError`: 403 Forbidden
- `ValidationError`: 422 Unprocessable Entity
- `ResourceConflictError`: 409 Conflict
- その他の例外: 500 Internal Server Error

## 10. ミドルウェア

アプリケーションには以下のミドルウェアが実装されています：

1. **CORS**: クロスオリジンリクエストを許可するためのミドルウェア
2. **現在のユーザー追加**: リクエストに現在のユーザー情報を追加するミドルウェア
3. **リクエストログ**: リクエスト情報をログに記録するミドルウェア
