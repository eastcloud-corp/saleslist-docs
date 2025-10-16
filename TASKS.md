# タスク管理表

下記の表にタスクを追記して管理してください。

| タスクNo | タスク名 | タスク詳細 | 着手日時 | 完了日時 | 主なコミットメッセージ | コミットID | 設計書ディレクトリ | メモ |
|----------|----------|------------|----------|----------|--------------------------|-----------|---------------------|------|
| UI-001 | 役職フィルター設計 | 代表取締役・取締役・執行役員・その他で複数選択できる役職フィルター仕様を整理 | 2025-10-14 06:59 |          |                          |           | docs/design/role-filter.md | 要件整理とUI/バックエンドの設計案作成。 |
| ENV-001 | Docker環境整備 | 3010/8010/5442ポート固定、Celeryワーカー/Beat含むdocker構成の整備 | 2025-10-14 05:18 | 2025-10-14 05:58 | Introduce Celery-based Facebook activity sync / Expose latest Facebook activity in company list | backend:d32826d9, front:904b07b1 | docs/dev-port-config.md | 既存composeのポート更新・worker/beat追加、E2E設定更新まで完了。
| DOC-001 | 企業最新更新列の仕様設計 | 企業管理画面に最新更新列を追加するため、Facebook友達数・投稿の追跡条件を整理 | 2025-10-14 05:11 | 2025-10-14 06:08 | ドキュメント更新 (active-company-timeline.md) | n/a | docs/design/active-company-timeline.md | スケジュール・リスク・フォローアップを追記し仕様確定。
| UT-001 | ユースケーステストUT-001設計 | ユースケーステストUT-001の設計 | 2025-10-14 07:08 | 2025-10-14 07:15 | n/a | n/a | docs/tests/ut-001-client-ng-project-flow.md | |
| UT-002 | ユースケーステストUT-001構築 | UT-001設計に基づくテスト実装・環境構築 | 2025-10-14 07:20 | 2025-10-14 07:21 | n/a | n/a | saleslist-front/tests/e2e/ut-001-client-ng-project-flow.spec.ts | UT-001に対応する自動テスト実装。 |
| QA-001 | 企業管理 業界フィルター調査 | 業界フィルターが機能しない件の原因調査 | 2025-10-10 15:17 | 2025-10-10 15:45 | (legacy) Playwright scenario rewrite | 不明 | tests/e2e/companies.spec.ts | 既存タスク。管理外コミットにつき参照なし。
| OPS-001 | マスターデータ再投入とE2E再検証 | Industry初期化コマンド実行およびE2E全体の再実行 | 2025-10-10 16:01 | 2025-10-10 16:27 | (legacy) master data seed update | 不明 | masters/management/commands/load_master_data.py | 既存タスク。管理外コミットにつき参照なし。
| UI-002 | Service Coverage表示調整 | Service Coverageセクション削除とログ表示改善 | 2025-10-14 17:59 | 2025-10-14 18:14 | n/a | n/a | saleslist-front | プロジェクト履歴ダイアログのログ強化とdocker構成更新。 |
| UI-003 | 役職フィルター不具合調査 | 役職フィルターが効かない原因の特定とUI改善方針検討 | 2025-10-14 18:32 |          |                          |           | saleslist-front | |
| QA-002 | 業界フィルター不一致調査 | CSVインポート業界とフィルター未一致の原因調査、複数業界検索時の動作検証 | 2025-10-16 15:24 | 2025-10-16 16:26 | 業界フィルターを部分一致と検索型UIに刷新 | front:fa6de0c, backend:ccbdcda | docs/design/industry-filter-enhancement.md | CSVデータ・API・UIの整合性を確認。 |
| FE-001 | クライアント企業リストエクスポート検討 | クライアント追加企業リストのエクスポート機能および7200件上限の要件整理 | 2025-10-16 15:24 | 2025-10-16 17:10 | クライアントCSVエクスポート実装と上限制御 | front:18f7ed1, backend:d5332fb | saleslist-backend/docs/client-company-export.md | 仕様案と影響範囲を整理。 |
必要に応じて行を追加し、最新の作業状況を更新してください。
