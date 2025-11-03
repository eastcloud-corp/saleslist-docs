# セキュリティ強化設計メモ（レート制御／クロスサイト対策／監視基盤）

## 1. APIレートリミット設計（DDoS・辞書攻撃耐性強化）

- **目的**: API への過剰リクエストによるリソース逼迫を抑止し、認証系エンドポイントへの総当たり攻撃を早期遮断する。
- **方針**:
  - Django REST Framework の `DEFAULT_THROTTLE_CLASSES` に `AnonRateThrottle` と `UserRateThrottle` を設定し、匿名／認証済みで異なるしきい値を適用する。
  - ヘルスチェック系は `ScopedRateThrottle` を使い、内部監視用途のアクセストークン（特定 IP または VPC）からのみ広いレートを許可する。
  - Nginx もしくは API Gateway 層で `limit_req` 相当のバースト制御を掛け、アプリ層の前段で粗い防御を実施する。
  - レート超過時のレスポンスに `Retry-After` を付与し、クライアント側のリトライ制御指針を明示する。
- **実装ステップ**:
  1. `saleslist_backend/settings.py` にレートリミット設定を追加（例: 匿名: 60/min、ユーザー: 600/min）。
  2. `accounts` ログイン API にはより厳しいしきい値（例: 10/min）を `ThrottleScope` で設定。
  3. Nginx/ALB 設定で `/health` / `/api/health/*` へのアクセスを社内 CIDR もしくは API キーで制限。
  4. レート超過ログを `security.rate_limit` チャンネルに集約し、SIEM からのアラート条件を定義。

## 2. クロスサイト対策設計（XSS / CSRF / Clickjacking）

- **目的**: フロントエンド（Next.js）でのユーザー入力処理とバックエンドの HTTP 応答ヘッダーを整備し、クロスサイト脅威を軽減する。
- **方針**:
  - Next.js 側で外部入力を HTML として描画する箇所を棚卸しし、`DOMPurify` 等でサニタイズする。`dangerouslySetInnerHTML` 依存箇所は限定的にし、ユニットテストでペイロードを検証。
  - SPA 全体の `Content-Security-Policy` を厳格化し、`script-src 'self'` `frame-ancestors 'none'` を基本とする（必要箇所のみ allowlist）。
  - Django の `X_FRAME_OPTIONS` と `CSRF_TRUSTED_ORIGINS` を見直し、管理画面／API のクリックジャッキングを防止。
  - 認証 API（Bearer トークン）には `Origin` / `Referer` チェックを追加し、クロスオリジン要求を拒否。
  - Playwright シナリオに XSS ペイロード投入テストを追加し、DOM 上のスクリプト挿入が無効化されていることを自動確認。
- **実装ステップ**:
  1. `saleslist-front/app` 配下でユーザー入力のレンダリング箇所をリスト化し、サニタイズユーティリティを共通化。
  2. Next.js の `next.config.js` もしくは `middleware.ts` でグローバルヘッダーとして CSP / XFO / Referrer-Policy 等を設定。
  3. Django 側で `SECURE_BROWSER_XSS_FILTER`、`X_FRAME_OPTIONS` を有効化し、管理サイトのヘッダーも統一。
  4. Playwright E2E に XSS 挙動チェック（例: フォームに `<img src=x onerror=alert(1)>` を入力しても実行されない）を追加。

## 3. 監査ログ・監視基盤設計（ゼロデイ／F5アタック検知）

- **目的**: アプリケーションログを一元的に収集し、異常トラフィックやゼロデイ兆候を自動検知できる体制を整える。
- **方針**:
  - `saleslist.health` を含む既存ロガーを `structlog` 形式で標準化し、ログに `request_id`、`user_id`、`client_ip`、`user_agent` を付与。
  - 収集先として ELK / OpenSearch / Datadog / CloudWatch など、既存インフラに合わせたログ基盤へ転送。
  - レートリミット超過、認証失敗連続発生、ヘルスチェック異常などを検知するアラートルールを設計。
  - F5 アタック等を識別するため、特定エンドポイントの秒間アクセス数をダッシュボード化（Prometheus + Grafana 等）。
- **実装ステップ**:
  1. `saleslist_backend/settings.py` の LOGGING で JSON ログ出力をデフォルト化し、Cloud Logging もしくは Filebeat から SIEM へ転送。
  2. ASGI ミドルウェア層でリクエスト ID 付与／レスポンスヘッダー反映を実装し、APM とのトレース連携を図る。
  3. アラート定義（例: 5 分間に認証失敗 50 回以上、ヘルスチェック 3 回連続失敗など）を監視ツールへ登録。
  4. 監視結果を `~/.saleslist_logs/` で既存運用しているログと同様に保持し、必要に応じて過去 90 日分を保全。

---

※ メール二段階認証（MFA）は本設計完了後に導入予定。ログ／監視基盤に MFA の成功・失敗ログも統合できるよう、API 設計時にイベントスキーマを揃えておくことを推奨します。
