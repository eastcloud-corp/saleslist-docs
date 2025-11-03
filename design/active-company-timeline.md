# 企業管理: 最新更新列の仕様設計

## 背景
企業管理画面で「アクティブな企業」を判別する指標として、Facebookから取得している以下の情報の変化を可視化したいという要望がある。

- 友だち数の増加
- Post（記事）の最新投稿時刻

既存画面では「最新の更新」を示す列が無いため、更新頻度が高い企業を一覧から素早く把握できない課題がある。

## 要件概要
企業管理テーブルに「最新の更新」という新列を追加し、以下の条件のいずれかを満たした時刻で更新する。

1. 直前に記録した友だち数より現在の Facebook 友だち数が増加した場合
2. 直前に記録した最新 Facebook 投稿より新しい投稿が見つかった場合（投稿情報は Graph API の `/posts` 等で取得）

これら条件を評価するために、バックエンドで Facebook 友だち数と最新投稿時刻、およびそれらの取得時刻を保持する必要がある。

## 対象範囲
- フロントエンド：企業一覧テーブルへの列追加と表示ロジック
- バックエンド：データモデル拡張、最新更新時刻の計算ロジック、Facebook同期処理
- バッチ/同期処理：Facebook API からの取得・更新トリガ
- テスト：Jest/Playwright/E2E、バックエンド単体テスト

## 詳細仕様案

### データモデル
- Company モデルに `facebook_page_id` を追加し、Graph API 呼び出しに利用する（URL から解決できないケースも想定）
- `CompanyFacebookSnapshot`（新規テーブル）
  - `company_id`（FK）
  - `friend_count`（int）
  - `friend_count_fetched_at`（datetime）
  - `latest_posted_at`（datetime）
  - `latest_post_fetched_at`（datetime）
  - `created_at` / `updated_at`
  - `source`（enum: cron, manual, webhook 等）

- `Company` モデルへの追加フィールド
  - `latest_activity_at`（datetime）: 表示用の最新更新時刻
  - 必要に応じて `facebook_friend_count`, `facebook_latest_post_at` などのキャッシュも追加検討

### 最新更新ロジック
1. Facebook API から現在の友だち数 (`current_friend_count`) と最新投稿時刻 (`current_latest_post_at`) を取得する。
2. 直前のスナップショット (`previous_snapshot`) と比較する。
   - 友だち数比較: `current_friend_count > previous_snapshot.friend_count`
   - 投稿時刻比較: `current_latest_post_at > previous_snapshot.latest_posted_at`
3. いずれか真の場合、`latest_activity_at` を `now()` で更新し、新しいスナップショットを保存する。
4. 変化が無い場合でもスナップショットは更新し、取得時刻を記録する（監査目的）。

### バッチ/同期
- 既存の Facebook 同期ジョブへロジックを組み込む。
- 取得頻度（例: 1日1回）や APIトークン管理は既存仕様に準拠。
- エラー時はリトライと失敗通知を検討。

### フロントエンド表示
- 企業一覧テーブルに「最新の更新」列を追加。
- `latest_activity_at` を日本時刻で表示。存在しない場合は `-` 表示。
- Tooltip または詳細ビューで、友だち数増加・投稿更新の履歴をハイライトする案も検討。

### API 変更
- `GET /companies/` のレスポンスへ `latest_activity_at` を追加。
- 必要に応じて `facebook_friend_count`, `facebook_latest_post_at` を含める。

### テスト観点
- バックエンド
  - 友だち数増加・投稿更新時に `latest_activity_at` が更新されること。
  - 変化が無い場合に時刻が維持されること。
  - 部分的なデータ欠落（友だち数未取得等）の扱い。
- フロントエンド
  - 新列が正しい時刻を表示する。
  - 並べ替えやフィルタ（要件次第）の挙動を確認。
- E2E（Chromium）
  - モック API で友だち数増加などを再現し、表示が切り替わることを確認。

## タスク分解（初期案）
1. データモデル整理と ERD 更新
2. バックエンドのモデル・マイグレーション実装
3. Facebook 同期処理に比較／更新ロジックを組み込み
4. API レスポンス拡張
5. フロントエンドのテーブル列追加実装
6. バックエンド単体テスト、フロントユニットテスト、Chromium E2E テスト
7. 運用面（ジョブ監視・ロギング）整理

---

上記は初期草案のため、実装前に API チーム／フロントチームとのすり合わせが必要。

## 運用方式

### スケジューリング
- Celery Beat を導入し、タイムゾーンを `Asia/Tokyo` に設定
- 毎日 AM 2:00 (JST) にタスク `dispatch_facebook_sync` をキューへ投入
- Worker / Beat / Redis をdocker-composeに追加し、本番でも同構成

### タスク分割
1. **ディスパッチタスク** (`dispatch_facebook_sync`)
   - Facebook連携対象企業IDを取得
   - 500〜1000件ごとにチャンク分割
   - `sync_facebook_chunk.delay(chunk_ids)` を並列でキューへ投入

2. **チャンクタスク** (`sync_facebook_chunk`)
   - 各企業について Facebook API から友だち数・最新投稿を取得
   - 過去スナップショットと比較し、条件を満たす場合 `latest_activity_at` を更新
   - スナップショットテーブルへ記録
   - レート制限に備え、API呼び出し間に間隔を挿入／再試行ポリシーを設定

3. **フォローアップ**
   - 失敗した企業IDは専用キューに積んで後続タスクで再試行
   - 処理結果はSlack通知や監査ログ (別ファイル) で可視化

### Facebook API
- Graph API (例: `/PAGE_ID?fields=fan_count` / `/posts?limit=1&fields=created_time`) を利用
- 認証トークン・アプリ設定は別途管理 (環境変数 / Vault)
- 今後、API利用方法・トークン取得手順を技術ノートにまとめる

### データフロー概要
1. Celery Beat が AM2:00 に `dispatch_facebook_sync` を投入
2. チャンクタスクが対象企業を分割し、Facebook API へリクエスト
3. 取得したメトリクスを `process_company_metrics` で比較し、`Company` と `CompanyFacebookSnapshot` を更新
4. API レスポンス経由で `latest_activity_at` 等がフロントへ传播され、一覧の「最新の更新」列に反映

### Feature Toggle
- `NEXT_PUBLIC_SHOW_FACEBOOK_ACTIVITY` を `false` にしておけばフロントの "最新の更新" 列は非表示のまま運用できる。
- 同期処理を一時停止したい場合は、`FACEBOOK_ACCESS_TOKEN` を空にしたり `FEATURE_TOGGLE_FACEBOOK_SYNC`（今後追加予定）で Beat 登録を制御する。
- 本番リリース前は列を隠した状態で、Facebook審査やトークン発行が完了次第フラグを `true` に切り替える。

### 制約・リスクと対応方針
- **Graph API のレート制限**: チャンクサイズとタスク並列数を調整。将来的にはアプリ審査やトークン更新手順をドキュメント化する。
- **FacebookページID の入力漏れ**: フォームで `facebook_page_id` を任意入力としたが、同期対象企業はこの項目を設定する運用を案内。未設定の場合はスキップし警告ログを出す。
- **トークン失効**: Celery タスクで `FacebookClientConfigurationError` を検知した際に警告ログを発報し、Slack/監視へ通知する仕組みを検討する。
- **API レスポンスの遅延**: タイムアウト (`FACEBOOK_GRAPH_API_TIMEOUT`) を設定し、失敗時は自動リトライ＋フォールバックキューで再処理。

### 次のステップ
- Celery構成: `celery.py` 追加、`docker-compose` に worker / beat サービス追加
- 認証トークン (`FACEBOOK_ACCESS_TOKEN`) や API バージョン (`FACEBOOK_GRAPH_API_VERSION`) は環境変数で管理し、Celery ワーカー/Beat から参照する。
- モデル・マイグレーション実装
- Facebook API アクセスラッパを作成し、テスト用モックを用意
- Playwright ではFacebookアクセスを直接行わず、モックレスポンスで同期後の表示を確認

## 今後のフォローアップタスク
| タスク候補 | 内容 | 備考 |
| --- | --- | --- |
| CELERY-OPS-001 | Celery worker/beat の本番デプロイ手順を Ops Runbook に追記 | 監視・再起動手順を含む |
| DOC-002 | Facebook トークン管理ノート作成 | 取得手順、期限、ローテーション |
| QA-002 | API レート制限に対する負荷テスト | チャンクサイズ調整のため |
| WEB-UI-001 | 企業詳細画面に Facebook 同期履歴タブを追加 | ユーザーが履歴を確認できるようにする |
