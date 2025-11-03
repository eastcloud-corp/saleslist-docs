# ログイン二段階認証（メールトークン）設計

## 目的
- 既存の ID/PW 認証にワンタイムメールトークンを追加し、なりすましリスクを低減する。
- 既存フロー（ログイン → ダッシュボード）に「メールトークン入力」画面を挟むのみで最小限の UI 変更に留める。
- SendGrid を利用したメール通知と Redis を用いたトークン管理で、追加ミドルウェアの導入を避ける。

## 全体フロー
1. 利用者が `/login` でメールアドレス＋パスワードを送信。
2. 認証情報が正しい場合、初回 JWT は発行せず「仮認証セッション ID (pending_auth_id)」を生成。
3. サーバは 6 桁の数字トークンを生成し、Redis に `pending_auth:{uuid}` として保存（TTL 30 分）。
   - 保存情報: `user_id`, `hashed_token`, `email`, `created_at`, `resend_count`, `last_sent_at`。
4. SendGrid 経由でトークンをメール送信。
5. フロントは `/login/mfa` へ遷移し、`pending_auth_id` を保持したままトークン入力を待つ。
6. 利用者がトークンを送信すると、バックエンドは Redis で検証し、成功したら JWT（有効期限 72 時間）を発行。
7. 成功後、該当 Redis データを削除し、ダッシュボードへ遷移。

## バックエンド設計
### 主要コンポーネント
- **Redis**: `pending_auth:{uuid}` キーで保管。TTL 30 分。
- **トークン生成**: 6 桁の数字（`random.randint(100000, 999999)` を想定）。保存時は SHA-256 などでハッシュ化。
- **メール送信**: SendGrid v3 API (`/mail/send`) を利用。`SENDGRID_API_KEY` を環境変数で管理。
- **レート制御**:
  - 再送: `last_sent_at` から 30 秒経過していない場合は拒否。
  - 30 分以内の再送最大 5 回 (`resend_count >= 5` でエラー)。
- **失敗処理**: トークン不一致は 400、期限切れ/未発行は 410 を返し、ログに記録。

### API 仕様
| メソッド | パス | 概要 |
| --- | --- | --- |
| `POST /api/v1/auth/login` | 第 1 段階認証。成功時 `{"pending_auth_id", "mfa_required": true}` を返す。
| `POST /api/v1/auth/mfa/resend` | トークン再送要求。レート制御と再送回数を検証。
| `POST /api/v1/auth/mfa/verify` | トークン検証。成功時に従来通り `access_token`, `refresh_token`, `user` を返す。

レスポンス例:
```json
// login 成功時
{
  "mfa_required": true,
  "pending_auth_id": "8f6b6c95-..."
}

// verify 成功時
{
  "access_token": "...",
  "refresh_token": "...",
  "user": {"id": 2, "email": "user@example.com", "name": "山田太郎", ...}
}
```

### Redis 構造
```json
{
  "user_id": 2,
  "hashed_token": "...sha256...",
  "email": "user@example.com",
  "created_at": "2025-10-30T04:00:00+09:00",
  "resend_count": 1,
  "last_sent_at": "2025-10-30T04:00:10+09:00"
}
```

### 設定項目（`.env`）
| 変数 | デフォルト | 説明 |
| --- | --- | --- |
| `SENDGRID_API_KEY` | (必須) | SendGrid の API キー |
| `MFA_TOKEN_TTL_SECONDS` | `1800` | メールトークンの有効期間（30 分）|
| `MFA_RESEND_INTERVAL_SECONDS` | `30` | 再送可能になるまでの待機時間 |
| `MFA_MAX_RESEND_COUNT` | `5` | 再送最大回数（期間内）|
| `MFA_TOKEN_LENGTH` | `6` | トークン桁数（必要なら拡張）|
| `MFA_DEBUG_EMAIL_RECIPIENT` | `""` | ローカル検証用の宛先を強制する場合に設定（未設定時はユーザーのメールアドレスへ送信）|

## フロントエンド設計
1. `/login` で `mfa_required: true` を受け取った場合、`/login/mfa?pending_auth_id=...&email=...` に遷移。
2. トークン入力フォームを表示（6 桁入力専用）。
3. 「再送する」ボタンを設置し、API から 429/400 を受け取ったらトーストで通知。
4. 成功時は既存の auth context にトークンを保存し、ダッシュボードへリダイレクト。
5. 失敗時はエラーメッセージを表示。3 回連続失敗などの UI 制限は任意（現時点ではサーバ側のみ）。

UI 構成案:
- タイトル: `メールで送信した確認コードを入力してください`
- フィールド: 6 桁の数字。自動フォーカス・自動進行は後続検討。
- ボタン: `確認する`（primary）、`再送する`（outline）。

## メールテンプレート（例）
```
件名: 【SalesList】ログイン確認コードのお知らせ
本文:
{{name}} 様
SalesList へのログインを確認するため、以下の確認コードを入力してください。

確認コード: {{token}}
有効期限: 発行から30分

このメールに心当たりがない場合は破棄してください。
```
SendGrid 側でテンプレート化する場合は Template ID を環境変数で管理。

SendGrid API キーが未設定の場合は Django のメールバックエンド（SMTP 等）を用いて送信を試みる。ローカル検証では `EMAIL_HOST`, `EMAIL_HOST_USER` 等と併せて `MFA_DEBUG_EMAIL_RECIPIENT` を設定することで任意アドレス（例: Gmail）に確認コードを転送できる。

## ログ・監査
- 成功/失敗時は `AUTH_MFA_SUCCESS`, `AUTH_MFA_FAILURE` として構造化ログを出力。
- 失敗ログには `user_id`, `reason (expired|mismatch|resend_limit)` を含める。
- SendGrid API 失敗時は警告ログ＋ Sentry 通知を推奨。

## 将来拡張案
- IP アドレスごとの送信回数制限（DoS 耐性強化）。
- 管理者への強制 MFA ポリシー（ロール別設定）。
- FIDO2 / Authenticator アプリ連携の検討。メール MFA は第一段階として導入。
