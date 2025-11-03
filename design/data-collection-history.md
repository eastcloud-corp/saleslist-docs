# データ収集履歴（Cloneバッチ）画面設計

## 背景
- Clone 系 Celery タスクの実行状況を可視化する手段がなく、成功/失敗や投入件数を都度ログから確認する必要がある。
- 企業管理画面にサンプルデータ投入や Celery コマンド実行リンクが点在しており、運用オペレーションが散在している。
- バッチの安定運用に向け、実行履歴のトレーサビリティと「次回実行予定」の可視化が求められている。

## 目的
- Clone タスクの実行履歴を新設する「データ収集履歴」管理画面に集約し、最新順で一覧表示する。
- 実行結果（件数、スキップ、エラー有無など）とともに実行ID（UUID）で追跡できるようにする。
- 画面上部に次回実行予定時刻を表示し、運用者がバッチのスケジュールを即座に把握できる状態を整える。
- これまで企業管理画面に置いていた Clone 関連オペレーション導線を本画面へ移し、管理画面の整理を図る。

## 対象スコープ
- Clone 系 Celery タスク（法人番号・オープンデータ等をレビュー候補に投入するバッチ）。
- 実行履歴の保存と突き合わせを行うバックエンド API。
- Next.js 管理画面の新規ページ・ナビゲーション変更。

## 対象 Celery タスク
| job_name | 対象タスク | 概要 | 実行トリガー | 現行スケジュール |
| --- | --- | --- | --- | --- |
| clone.corporate_number | companies.tasks.run_corporate_number_import_task | 法人番号APIから候補を取得し、レビュー待ちへ投入 | 手動／定期（Celery Beat想定） | 未設定（要Beat設定確認） |
| clone.opendata | companies.tasks.run_opendata_ingestion_task | 各自治体オープンデータから候補を生成 | 手動／定期 | 未設定（要Beat設定確認） |
| clone.facebook_sync | companies.tasks.dispatch_facebook_sync | Facebookメトリクス同期（既存Beat設定あり） | Celery Beat | 毎日 02:00 JST |
| clone.ai_stub | companies.tasks.run_ai_ingestion_stub | AI連携スタブ | 手動 | なし |


## ログ・メトリクス要件
| 項目 | 必須 | 説明 |
| --- | --- | --- |
| `execution_uuid` | ✅ | タスクごとに発行する UUID。開始・終了ログ／DB 格納をこの ID で紐付ける。 |
| `job_name` | ✅ | タスク識別子（例: `clone.corporate_number`）。 |
| `data_source` | ✅ | 処理対象データソース（東京都CSV など）。複数の場合は配列も許容。 |
| `started_at` / `finished_at` | ✅ | 実行開始・終了時刻（JST）。終了がない場合は実行中扱い。 |
| `duration_seconds` | ✅ | `finished_at - started_at` の差分。終了時に算出。 |
| `input_count` | ✅ | データソースから取得した総件数。 |
| `inserted_count` | ✅ | レビュー候補テーブルへ追加した件数。 |
| `skipped_count` | ✅ | 重複・更新不要・期限切れなどでスキップされた件数。 |
| `error_count` | ✅ | 例外発生件数。0 の場合でも記録。 |
| `status` | ✅ | `SUCCESS` / `FAILURE` / `RUNNING`。 |
| `error_summary` | 任意 | エラー時の要約（例外メッセージ先頭 256 文字程度）。 |
| `next_scheduled_for` | ✅ | 次回実行予定時刻。終了時点で Celery Beat スケジュールまたは設定値から算出。 |
| `created_at` / `updated_at` | ✅ | レコード作成・更新時刻。 |

## データ保存方針
| フィールド | 型 | 説明 | 備考 |
| --- | --- | --- | --- |
| id | AutoField | 主キー | |
| execution_uuid | UUIDField | 実行識別子 | unique=True |
| job_name | CharField(64) | タスク名 | index |
| data_source | JSONField | データソース情報 | 配列/オブジェクト格納 |
| started_at | DateTimeField | 開始時刻 | timezone aware (JST 表示) |
| finished_at | DateTimeField(null=True) | 終了時刻 | 実行中は null |
| duration_seconds | IntegerField | 所要時間(秒) | finished_at 更新時に算出 |
| input_count | IntegerField | 取得件数 | default=0 |
| inserted_count | IntegerField | 追加件数 | default=0 |
| skipped_count | IntegerField | スキップ件数 | default=0 |
| error_count | IntegerField | エラー件数 | default=0 |
| skip_breakdown | JSONField(null=True) | スキップ内訳 | 理由別件数（オプション） |
| status | CharField(16) | 実行状態 | RUNNING/SUCCESS/FAILURE |
| error_summary | TextField(null=True) | エラー概要 | 256〜512文字程度 |
| metadata | JSONField(null=True) | 任意メタ情報 | 実行パラメータ保存 |
| next_scheduled_for | DateTimeField(null=True) | 次回予定 | Celery Beat算出結果 |
| created_at | DateTimeField(auto_now_add=True) | 作成日時 | |
| updated_at | DateTimeField(auto_now=True) | 更新日時 | |

- 新規 Django モデル `DataCollectionRun` を `Clone` 専用で作成し、上記フィールドを保持。
  - 保存期間は原則無制限（年次運用で手動アーカイブ／削除）。
  - `execution_uuid` をユニーク制約。
  - `job_name` + `started_at` で降順インデックスを付与し、最新順ソートを高速化。
- Celery タスクハンドラ内で以下を実装：
  1. 実行開始時に `execution_uuid` を生成し、`DataCollectionRun` レコードを `RUNNING` で作成。
  2. 処理中に `input_count` や `skipped_count` を随時更新できるよう、レコードを部分更新。
  3. 正常終了時に `status=SUCCESS`、`finished_at`、`duration_seconds`、`inserted_count` 等を保存。
  4. 例外発生時は `status=FAILURE`、`error_summary` を保存し、`finished_at` を記録。
- Celery シグナル（`task_prerun` / `task_postrun` / `task_failure`）を活用して共通処理化。
- 既存構造化ログ (`logger.info`) にも `execution_uuid` を含め、ログ集計との突き合わせを可能にする。

## API 設計
### エンドポイント案
| メソッド | パス | 説明 |
| --- | --- | --- |
| `GET /api/v1/data-collection/runs` | 実行履歴一覧（デフォルト新しい順、ページネーション対応）。 |
| `GET /api/v1/data-collection/runs/{execution_uuid}` | 個別詳細取得。 |
| `POST /api/v1/data-collection/runs/search` | フィルタ検索（期間、ジョブ、ステータスなど）※将来的に追加。 |
| `POST /api/v1/data-collection/trigger` | 手動実行トリガー（ジョブ毎に実行、オプションで企業指定） |

### 手動トリガーAPI 仕様
- リクエスト例
```json
{
  "job_name": "clone.corporate_number",
  "options": {
    "company_ids": [123, 456],
    "source_keys": ["tokyo"],
    "allow_missing_token": false
  }
}
```
- `job_name` は対象タスク（clone.corporate_number / clone.opendata / clone.facebook_sync / clone.ai_stub）に限定。
- `options.company_ids` を指定した場合、その企業 ID のみを対象に処理。未指定時はタスク既定の対象を全件処理。
- `options.source_keys` でオープンデータのソースキーを絞り込み。その他オプションは既存コマンドの引数に準拠。
- 同一 `job_name` で `status=RUNNING` のレコードが存在する場合は 409 を返却し、二重実行を防止。
- レスポンスは起動された `execution_uuid` と現在の `next_scheduled_for` を返す。

- 応答例（一覧）:
```json
{
  "next_scheduled_for": "2025-10-31T00:30:00+09:00",
  "count": 2,
  "results": [
    {
      "execution_uuid": "a1b2c3d4-...",
      "job_name": "clone.corporate_number",
      "data_source": ["houjin_bangou_api"],
      "started_at": "2025-10-30T22:05:12+09:00",
      "finished_at": "2025-10-30T22:06:45+09:00",
      "duration_seconds": 93,
      "input_count": 512,
      "inserted_count": 428,
      "skipped_count": 84,
      "error_count": 0,
      "status": "SUCCESS"
    },
    { "...": "..." }
  ]
}
```
- クエリパラメータ: `?page=1&job_name=clone.corporate_number&status=SUCCESS&started_after=...`。
- 認可: 管理者ロール（`role=admin`）のみアクセス可。

## Celery スケジュールとの連携
- `next_scheduled_for` は以下の優先順位で決定：
  1. Celery Beat の `schedule` 定義から計算 (`crontab` → 次回時刻)。
  2. 手動実行後に設定されたクールダウンがある場合はその終了時刻。
- バックエンドは `Clone` タスク設定から次回時刻を算出するヘルパーを提供し、API レスポンスにも含める。
- 画面上部に `次回実行予定: 2025-10-31 00:30 JST` を表示し、ステータスが `RUNNING` の場合は「実行中」と表示。

## フロントエンド設計
- 新規ページ: `/data-collection/history`。
- 画面構成:
  1. **ヘッダー領域**: 次回実行予定時刻、直近ステータス、手動トリガーボタン。
  2. **フィルタバー**: 期間（当日/7日/30日）、ジョブ種別、ステータス、データソース。
  3. **履歴テーブル**（最新順）：1ページ 100 件固定、ページングで過去分を閲覧。
     - 列: 開始日時、終了日時、ジョブ名、データソース、処理件数（取得/投入/スキップ）、ステータス、所要時間、実行ID（コピーアイコン付）、エラー概要。
     - 行クリックでエラー概要の全文と JSON レスポンスを展開表示（専用モーダルは不要）。
- 企業管理画面にあった「サンプルデータ投入」「Celery コマンド実行」導線は削除し、本画面に集約。
- UI コンポーネントは既存の `Table` / `Badge` / `Alert` を再利用。処理件数は `Tooltip` で補足説明を表示。

## 権限とナビゲーション
- Sidebar の「管理」セクションに「データ収集履歴」を追加（Admin ロールのみ表示）。
- 企業管理画面から Clone 関連のカード／ボタンを撤去し、必要ならリダイレクトバナーを一定期間表示。

## 監査・アラート拡張（将来）
- `DataCollectionRun` の結果に基づき、件数急減・失敗時に Slack/Sentry 通知を送る仕組みを追補可能。
- `execution_uuid` を使い Kibana / Cloud Logging で詳細ログへ遷移する導線を検討。

## 実装ステップ案
1. Django モデル・マイグレーション追加、シリアライザー／ViewSet 実装。
2. Celery タスク共通ラッパーで `execution_uuid` 発行とログ更新を実装。
3. API エンドポイント・テスト追加（一覧、新規記録）。
4. Next.js 側で専用 hooks (`useDataCollectionRuns`) とページ実装。
5. ナビゲーション更新・既存オペレーション導線移設。
6. ドキュメント整備（運用手順、通知連携など）。

## 依存関係・検討事項
- Celery Beat 設定ファイル（`saleslist-backend/saleslist_backend/settings/*`）との整合を保つ。スケジュールが環境で異なる場合は API レイヤーで環境情報を返す。
- Redis / DB への書き込み負荷は軽微だが、履歴の保持期間（例: 90 日）を設定し、古いレコードを削除するバッチを用意するとよい。
- 次回実行予定はタスク種別ごとに差がある場合があるため、`job_name` 毎に設定値を持つ Config テーブルを検討。
- 手動トリガーにロック（同時実行防止）が必要な場合は別途整理する。

