# 実装プラン: シンプルAIチャット（Claudeライク）

## Context

[要件定義書_シンプルAIチャット.md](./要件定義書_シンプルAIチャット.md) に基づき、Claude APIを使ったシンプルなWebチャットアプリをゼロから実装する。リポジトリは現在ドキュメントのみ（コードなし）。Must機能（MVP）→ Should機能（v1.0）→ Could機能の順に、マイルストーン単位で動作確認しながら進める。

## 技術スタック（要件定義書 §4.2 準拠）

- Next.js (App Router) + React + TypeScript + Tailwind CSS
- `@anthropic-ai/sdk` によるストリーミング（Vercel AI SDKは使わない — カスタムSSEイベント（保存済みメッセージID・タイトル・エラー種別）が必要で、依存も減らせるため）
- SQLite + Prisma（`conversations` / `messages` / `settings`）
- `react-markdown` + `remark-gfm` + `rehype-highlight`（生HTML非描画がそのままXSS対策。`rehype-raw`は入れない）
- 状態管理は plain React state（`useState` + カスタムフック2つ）。Redux/Zustand/SWRなし

## データモデル（prisma/schema.prisma）

- `Conversation`: id, title（default "新しい会話"）, model（default "claude-sonnet-5"）, createdAt, updatedAt
- `Message`: id, conversationId（Cascade delete）, role, content, model?, stopped（中断された応答フラグ）, createdAt
- `Setting`: key-value（systemPrompt / defaultModel / maxTokens。初回読み込み時にデフォルトをseed）

## API設計

| Route | Method | 内容 |
|---|---|---|
| `/api/conversations` | GET / POST | 一覧（updatedAt降順）/ 新規作成 |
| `/api/conversations/[id]` | GET / PATCH / DELETE | 履歴取得 / title・model更新 / 削除 |
| `/api/settings` | GET / PUT | 設定の取得・更新 |
| `/api/chat` | POST | 本体。SSEストリーム返却 |

### `/api/chat` のフロー（コア）

1. 検証（content ≤ 50,000字、会話存在確認）
2. `regenerate: true` なら最後のassistantメッセージを削除、通常時はuserメッセージを**API呼び出し前に**保存（失敗時も入力が残り再試行できる）
3. 履歴ロード → `lib/truncate.ts` で古いメッセージから切り捨て（文字数ベース予算 ~350,000字。ISS-002は「単純切り捨て」で解決）
4. `anthropic.messages.stream()` 呼び出し。`thinking: {type:"disabled"}` を明示（Sonnet 5は省略時adaptive thinkingになりコスト・レイテンシが不安定になるため）。`temperature`等は送らない（Sonnet 5は非デフォルト値を拒否）。`maxRetries: 1`（要件: 自動リトライ1回まで）。`req.signal` をSDKに渡す（停止ボタン・タブ閉じで上流もキャンセル）
5. SSEイベント: `delta`（逐次テキスト）/ `done`（保存済みmessageId）/ `title`（初回のみHaikuで生成、失敗は無視）/ `error`（retryable判定つき。コンテキスト超過は「新しい会話を開始してください」）
6. 中断時は途中までのテキストを `stopped: true` で保存（リロード後も残る）

## フロントエンド構成

```
app/page.tsx          # シェル。conversations/activeId/messages/isStreaming をここに集約
components/
  Sidebar.tsx         # 会話一覧・新規・削除・切替 (F-003)
  ChatView.tsx / MessageList.tsx / MessageItem.tsx
  Markdown.tsx        # react-markdown + gfm + highlight (F-004)
  MessageInput.tsx    # Enter送信/Shift+Enter改行、50k字ガード、生成中はStopボタン
  ModelSelector.tsx / SettingsDialog.tsx / Disclaimer.tsx
hooks/
  useConversations.ts # 会話CRUD
  useChat.ts          # 送信/停止/再生成。AbortController保持、fetch body readerでSSEパース
lib/
  db.ts               # PrismaClientシングルトン（dev hot-reload対策のglobalThisガード）
  anthropic.ts / models.ts / truncate.ts
```

モデル: 既定 `claude-sonnet-5`、選択肢 `claude-haiku-4-5-20251001`、タイトル生成はHaiku固定。

## マイルストーン（各段階で動作確認してから次へ）

| # | 内容 | 完了条件 |
|---|---|---|
| M0 | scaffold: create-next-app、依存導入、Prismaスキーマ+migrate、env設定 | dev起動、Prisma Studioに3テーブル、`.env*`/`dev.db`がgitignore済み |
| M1 | チャット送受信+永続化 (F-001/F-002): `/api/chat`、プレーンテキストのストリーミングUI | 送信→2秒以内に逐次表示、リロードで履歴復元 |
| M2 | 会話管理 (F-003): サイドバー+conversations API | 3会話で切替混線なし、削除でメッセージもcascade削除 |
| M3 | **MVP完成** (F-004+ガードレール): Markdown/ハイライト、50k字制限、エラー+再試行ボタン、フッター免責、履歴切り捨て | コード・表が正しく表示、`<script>`が文字列表示（XSS無効）、通信断でエラー+再試行 |
| M4 | 停止・再生成 (F-005/F-006) | 生成中Stopで途中テキストがリロード後も残る、再生成は最後の応答のみ置換 |
| M5 | **v1.0完成** (F-007/F-008): モデル選択、設定画面（システムプロンプト） | Haiku切替が保存メッセージのmodel列で確認できる、設定が再起動後も有効 |
| M6 | Could (F-009/F-010): Haikuタイトル自動生成、コピーボタン | 初回応答後にサイドバーへ日本語タイトル、コピーで元Markdown取得 |

## テスト方針

- 自動テストは `lib/truncate.ts` のみVitestで（唯一アルゴリズム的にリスクがある純関数）: 最新優先・予算遵守・単発超過メッセージのエラー
- それ以外は各マイルストーンの手動確認（要件定義書 §5 の方針どおり）。M3完了時に評価セット20問を `docs/eval/` に作成して目視確認

## 落とし穴（実装時に注意）

- StreamingルートはNode runtime必須（Prisma/SQLite）。`Cache-Control: no-transform` でバッファリング防止
- assistantメッセージの保存はサーバー側のみ（single source of truth）。クライアントは `done` イベントのmessageIdで整合
- 再生成は「最後のassistant削除+再生成」を同一トランザクションで（クラッシュ時の重複防止）
- APIキーは `.env.local`、`DATABASE_URL` は `.env`（Prismaが読むのは`.env`）。`NEXT_PUBLIC_` 禁止

## 検証（最終）

1. `npm run dev` で起動し、M1〜M6の完了条件を順に手動確認
2. `npx vitest run` でtruncateテスト通過
3. 要件定義書 §5.1 の観点（表示崩れ0件、中断時のエラー表示、履歴の保存/再読込一致）を評価セットで確認
