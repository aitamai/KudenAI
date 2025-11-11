# 08_API仕様（現行システム）

## 認証系
| メソッド/パス | 概要 | 認可 |
| --- | --- | --- |
| `POST /api/auth/register` | メール・パスワード・プロフィール初期値を登録、OTP を発行 | Public |
| `POST /api/auth/login` | メール/パスワード検証、成功時に OTP フェーズへ | Public |
| `POST /api/auth/otp/verify` | Redis と比較し 2FA 成功でセッション発行 | Public |
| `POST /api/auth/password/reset/request` | uuid4/token_urlsafe トークン生成、メール送信 | Public |
| `POST /api/auth/password/reset/confirm` | 新パスワード設定、全端末ログアウト | Public |
| `DELETE /api/auth/session/{id}` | 特定セッションを失効 | Auth |

## プロフィール/設定
| `GET/PUT /api/profile` | 名前/アイコン/職業/年齢/居住地/AI公開フラグ/機密チェックを操作 |
| `POST /api/profile/totp` | Django-OTP TOTP デバイスの登録/削除 |
| `GET /api/profile/usage` | トークン利用量、日次 usage_counters を返却 |

## 日記・会話
| `POST /api/diary/sessions` | 日記チャットを開始、AI 質問を返す |
| `POST /api/diary/messages` | ユーザー回答（テキスト/音声URL）。Whisper STT 結果を返却 |
| `POST /api/diary/summary` | LangChain で要約/追記提案を生成 |
| `GET /api/diary/{date}` | 保存済み日記と会話履歴を取得 |
| `GET /api/diary/search` | 既存日記を全文/Embedding で検索（RAG） |

## AI アバター
| `POST /api/avatars` | 性格診断/文体/設定を登録、trait を推定 |
| `POST /api/avatars/{id}/prompt` | プロンプト生成・バージョン管理 |
| `POST /api/avatars/{id}/publish` | 公開、固定 URL 発行、チャット起動 |
| `GET /api/avatars/{id}` | 詳細・公開設定・trait・進捗% |

## チャット/RAG
| `POST /api/chat/{avatar_id}/messages` | ユーザー発話 → RAG 検索 → LLM 応答。Langfuse trace ID を返却 |
| `GET /api/chat/{avatar_id}/history` | 会話履歴を返却 |
| `POST /api/rag/query` | Drive/Notion/S3 ソースを指定して検索（権限チェック） |

## Embedding/ナレッジ
| `POST /api/embeddings/reindex` | 対象（日記/会話/診断）を再埋め込み |
| `GET /api/embeddings/status` | Pinecone upsert の状態、Langfuse trace を参照 |

## 課金/トークン
| `GET /api/billing/usage` | token_ledger / usage_counters を集計して返却 |
| `POST /api/billing/purchase` | 追加トークン購入を Stripe Checkout で開始 |
| `POST /api/stripe/webhook` | サブスク更新/支払い成功を受信し、usage を更新 |

## 管理/監査
| `GET /api/admin/audit` | audit_logs をフィルタリングして表示（admin 限定） |
| `POST /api/admin/token/adjust` | token_ledger に補正仕訳を追加 |
| `GET /api/admin/moderation` | 通報キュー、アバターブロック/解除 |

## スマホアプリ連携
- `GET /api/mobile/bootstrap`: PWA/ネイティブ双方で利用。利用状況、ジョブ状態、RAG 状態のダッシュボードデータを返す
- ディープリンク (`app.kuden.ai/chat/{avatar}`) からチャットを即起動

## 共通事項
- 認証: `Authorization: Bearer <session_token>`。2FA 未完了のセッションは `403 OTP_REQUIRED`
- レート制限: Redis ベース (per-IP/per-user)。Abuse 検知時は WAF を更新
- レスポンス共通項目: `trace_id` (Langfuse)、`token_usage`、`ttl_hint`（Redis バッファの残時間）
- 監査: 重要 API は `audit_logs` + CloudWatch + Sentry

