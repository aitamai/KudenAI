# Components Overview

主要ページを支える UI/サービスコンポーネントを、フロントエンド（Next.js/React）、バックエンド（Django/Celery）、サードパーティ連携（LangChain/Stripe/Pinecone/Langfuse）ごとに整理する。

## 1. フロントエンド UI
- **AuthFormGroup**
  - フロー: メール/パス入力 → `POST /api/auth/login`
  - 機能: バリデーション、レスポンシブ対応、エラー表示、remember-me でセッション延長
- **OTPVerificationPanel**
  - Redis TTL に合わせたカウントダウン、再送ボタン（30秒クールダウン）、Django-OTP 連携
  - 2FA 成功時にセッションストアを更新 (`session:{user_id}:{uuid}`)
- **PasswordResetModal**
  - リンク起動→メール送信→トークン入力→新パスワード設定
  - 再設定成功で全端末ログアウト通知
- **DashboardCards**
  - 日記/チャット/アバター/トークン/プランのエントリポイント
  - Langfuse/usage_counters から利用量を取得し、残量警告や購入導線を表示
- **DiaryChatComposer**
  - 会話タイムライン（AI質問/ユーザー回答/要約）
  - Whisper 音声入力、IndexedDB バッファ、既存日記検索結果のハイライト
- **DiarySummaryPanel**
  - LangChain 要約/追記、感情/タグ抽出の結果をカード表示
  - 保存ボタンで Redis `diary:buffer` と PostgreSQL `diary_entries` を同期
- **AvatarStudio**
  - プロフィール入力率ゲージ、機密情報チェックボックス
  - 性格診断/文体チェックの実行ボタン、trait 結果カード、AI 公開スイッチ
  - プロンプトエディタ（システム/ロール/安全/評価/テストのタブ）
- **ChatRoom**
  - アバターとの対話UI。RAG ソース選択、引用表示、トークンカウンタ、Langfuse trace ID 表示
  - 固定 URL 共有、SNS 操作（フォロー/いいね/コメント/シェア/通報）
- **ProfileSecuritySection**
  - 2FA デバイス管理、セッション一覧/リモートログアウト、パスワード変更
  - 性格診断サマリ JSON 表示と再生成ボタン
- **BillingUsageView**
  - token_ledger/usage_counters/Stripe サブスク状態、インボイス/JPN 領収対応
  - 追加トークン購入ボタン（Stripe Checkout）、Langfuseトークン可視化グラフ
- **AdminConsole**
  - ユーザー管理、通報キュー、監査ログ、プラン調整
  - MAU/DAU/ARPU/LTV/トークン収支などのメトリクスボード

## 2. バックエンドサービス
- **AuthService (Django)**
  - `authenticate`、Django-OTP、Redis セッション、RBAC、パスワードリセットトークン
- **ProfileService**
  - プロフィール更新、機密項目の AES 暗号化、AI 公開フラグ、trait JSON 管理
- **DiaryService**
  - 日記セッション管理、Redis バッファ、Whisper STT、LangChain 要約/追記
  - Celery 03:00 バッチで PostgreSQL 永続化、Pinecone upsert、Langfuse trace
- **ChatService**
  - アバター対話、RAG ソース制御、LangChain 呼び出し、Langfuse trace 発行
- **AvatarService**
  - 性格診断/文体チェック/trait 推定、プロンプトバージョン管理、公開URL発行、固定チャットルーム作成
- **BillingService**
  - token_ledger 二重仕訳、usage_counters 日次更新、Stripe Webhook、追加トークン購入
- **EmbeddingService**
  - LangChain で埋め込み生成、`hash(user_id+text)` ID 変換、Pinecone upsert、Langfuse 記録
- **ModerationService**
  - SNS 通報処理、違反判定、アカウント/アバターブロック、監査ログ出力
- **NotificationService**
  - OTP/パスワードリセットメール、Slack/メール/Push 通知、セグメント配信

## 3. 非同期処理 / バッチ
- **Celery Beat スケジューラ**
  - 03:00 Redis→PostgreSQL フラッシュ、BigQuery/Redshift ストリーム、S3 バックアップ
  - Embedding 再生成、trait 判定、Langfuse 集計
- **Celery Worker**
  - Whisper 音声 STT、Stripe Webhook 冪等処理、トークン記帳再試行、RAG インデックス更新

## 4. 外部連携コンポーネント
- **LangChain Pipelines**
  - LLM (OpenAI/Gemini/Claude) 抽象化、プロンプトテンプレート、RAG ソース選択、trait 推定、要約/追記
- **Pinecone Vector Store**
  - 日記・診断・会話ログ embedding を namespace（user/org/avatar）別に管理
  - upsert 時に重複防止 ID を付与、Langfuse trace で可視化
- **Langfuse Trace**
  - LLM/RAG/Embedding/Token すべてのフェーズで trace を発行し、token_ledger ID と紐付けて監査・コスト計測
- **Stripe Integration**
  - サブスク/アドオン課金、Webhook 冪等処理、usage_counters 反映、請求/領収データ生成
- **Whisper STT Gateway**
  - WebRTC 録音→音声アップロード→テキスト化→DiaryService へ送信

## 5. 監視/セキュリティ
- **CloudWatch Dashboards**
  - API レイテンシ、OTP 失敗、Celery 失敗、LLM レスポンスタイムを監視。閾値超過で Slack/SNS 通知
- **Sentry**
  - フロント/バック双方の例外トラッキング
- **AuditLogger**
  - 2FA 設定/公開設定/トークン調整/モデレーションを `admin_events` / `audit_logs` に記録
- **WAF/RateLimiter**
  - Redis ベースで per-IP/per-user レート制限、Abuse 検知で WAF 更新

## 6. データパイプライン
- **TokenLedger Pipeline**
  - LLM 応答完了→actual_tokens 算出→ユーザー→運営/運営→アバター仕訳→usage_counters→Langfuse
- **DiaryEmbedding Pipeline**
  - 日記保存→Celery ジョブ→LangChain embedding→Pinecone upsert→Langfuse trace
- **RAG Retrieval Pipeline**
  - ユーザー発話→ソース選択→Pinecone 検索→引用整形→LLM 回答→UI へ返却

各コンポーネントの詳細仕様は `docs/pages.md` と `input_document/00〜08` の章を参照。
