# Task Tracker

※日時はJSTで記録してください。新規タスクは着手前でも登録し、進捗があれば随時更新します。

| タスクNo | タスク名 | タスク詳細 | 着手日時 | 完了日時 | 主なコミットメッセージ | コミットID | 設計書ファイルパス | メモ |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| T-001 | 開発/本番設定の切り分け | Django・Next.js・docker-compose を本番用と開発用で分離し、`DEBUG` など開発専用設定が本番へ混入しない構成に刷新する | 2025-10-20 05:05 JST | 2025-10-20 05:25 JST | 未コミット | 未コミット | saleslist-docs/ | Django設定をbase/dev/prodで分割、Next.js/.env整備、docker-composeの環境変数を整理 |
| T-002 | 環境変数と設定値の一元管理 | API ベースURLやHMR設定などを `.env` / secrets に集約し、再デプロイ時に値がぶれないよう管理方法を定義する | 2025-10-20 05:30 JST | 2025-10-20 05:45 JST | 未コミット | 未コミット | saleslist-docs/ | `env/dev|prod/*.env` を新設し compose/env_file から参照、旧 env テンプレートを整理 |
| T-003 | 初期化処理の自動化 | DBシード・`createcachetable` などを起動フック化し、手動作業なしで安定起動できるようにする | 2025-10-20 05:30 JST | 2025-10-20 05:50 JST | 未コミット | 未コミット | saleslist-docs/ | backend entrypoint に `ENABLE_SAMPLE_DATA` を追加し、dev 用 `scripts/run-dev.sh` で migrate/seed/runserver を自動化 |
| T-004 | 余分なボリューム/コンテナのクリーンアップ整備 | 再デプロイ時の `down --volumes` 手順や orphan コンテナ除去のガイドラインを整備する | - | - | - | - | saleslist-docs/ | 運用手順書の更新も含む |
| B-001 | レビュー否認後に404へ遷移する不具合修正 | 本番レビュー画面で「否認」選択時に404画面へ飛ぶ挙動を修正し、正常に結果反映・画面遷移できるようにする | 2025-11-07 06:00 JST | 2025-11-07 06:37 JST | Fix review workflow guards and log processed ids / Guard review sample UI and surface job targets | fb0302cf / 11c09f63 | saleslist-front | 本番環境での再現確認とバックエンドAPIレスポンスの確認を含む |
| B-002 | 本番レビュー画面のサンプルデータ追加ボタン非表示対応 | 本番環境のレビュー画面に検証向け「サンプルデータ追加」ボタンが残っているため、運用環境では非表示にする | 2025-11-07 06:00 JST | 2025-11-07 06:37 JST | Guard review sample UI and surface job targets | 11c09f63 | saleslist-front | 環境判定ロジックの整備とUI確認が必要 |
| B-003 | バッチ処理の処理件数・対象企業ID可視化 | AI / バッチ系処理で処理した件数や企業IDをログ・画面から確認できるようにし、企業IDリンクで詳細へ遷移できる導線を追加する | 2025-11-07 06:00 JST | 2025-11-07 06:37 JST | Fix review workflow guards and log processed ids | fb0302cf | saleslist-backend | 既存ログ出力の拡張と管理UI/履歴画面の表示検討 |
| B-004 | 案件向け企業選択画面のソート/件数改善 | 企業追加ダイアログで 50 件しか表示されない問題に対応し、最大500件の表示と昇順/降順切替、件数サマリを提供する | 2025-11-07 09:00 JST | 2025-11-07 09:30 JST | Support ordering on client available companies / Add sort control and range summary to company selector | 432cb5e / 0f038f2 | saleslist-front | `page_size` パラメータ有効化と UI 表示改善 |
| B-005 | 企業管理→企業追加後にフィルタを維持 | 企業管理画面でフィルタ・選択した企業を案件に追加したあとも、同じフィルタ状態で企業管理画面に留まれるようにする | 2025-11-07 10:00 JST | 2025-11-13 00:45 JST | feat: keep company filters after project add | a15c8254 | saleslist-front | トースト経由で案件画面に遷移できる導線を残しつつ、自動遷移を廃止してフィルタ/選択状態を維持 |
| T-005 | 監視・ヘルスチェックの導入 | API ヘルスチェックと基本的な監視/通知の仕組みを整え、不具合検知を早期化する | - | - | - | - | saleslist-docs/ | 既存ログ設定の見直しも含める |
| T-006 | 補完候補レビュー機能の設計 | AI提案＋ルールベース取得した更新候補をレビューし、承認された項目のみ反映できるワークフローを設計・実装する | - | - | - | - | saleslist-docs/design | 新規レビュー画面、候補まとめロジック、承認履歴テーブル、操作ログを整備。実際のルールベース／AI連携は後続タスクで実装予定（現状はサンプル生成APIで代替）。本番運用時は国税庁API＋東京都/静岡県/大阪府オープンデータを採用し、否認済み再提案ロジックを組み込む計画。Power Plexy 連携は別途 T-010（AI候補生成）で実装予定。 |
| T-007 | 法人番号専用候補パイプライン | 公的データソースから法人番号のみを抽出し `company_update_candidates` に投入、レビュー導線を分離して優先処理できるようにする | - | - | - | - | saleslist-docs/design | `generate_corporate_number_candidates` の実装、重複/再提案ブロックの整理、バッチ投入と履歴保存までを実装 |
| T-008 | 法人番号レビューUI整備 | レビュー画面に法人番号専用のフィルタ／表示を追加し、レビュアーが最優先で処理できる導線を用意する | - | - | - | - | saleslist-front/app/companies/reviews | バッジ表示・タブ／フィルタ追加、承認後のトーストや進捗表示を法人番号向けに最適化 |
| T-009 | ユーザー向け利用ガイド整備 | 企業管理・レビュー・案件管理の連携と法人番号バッチ運用をまとめた利用手順書 `saleslist-docs/user-guide.md` を作成する | 2025-10-26 18:40 JST | 2025-10-26 19:10 JST | 未コミット | 未コミット | saleslist-docs/user-guide.md | レート制御/force実行の注意点、ダミー生成による検証方法を記載 |
| T-010 | 法人番号API利用モック画面 | API申請用スクリーンショット取得のため、法人番号/法人名検索のデモ画面を実装する | 2025-10-29 16:45 JST | 2025-10-29 16:55 JST | 未コミット | 未コミット | saleslist-front/app/corporate-number/mock/page.tsx | フロント単体のデモページ。バックエンド接続なし。 |
| T-011 | 法人番号・オープンデータCeleryタスク整備 | 既存バッチをCeleryタスク化し、フロントの実行ボタンと連動させる。1ヶ月クールダウン設定を適用 | 2025-10-30 02:40 JST | 2025-10-30 03:24 JST | 未コミット | 未コミット | saleslist-docs/ | docker/dev に redis/worker を追加、API は Celery 経由で実行するよう更新。 |
| T-012 | オープンデータソース拡充 | 埼玉/神奈川/千葉/愛知/兵庫/京都/福岡/広島の自治体データを `config/opendata_sources.yaml` に追加し、マッピングとテストを整備する | - | - | - | - | saleslist-docs/design | まずは各自治体データの公開形式調査と CSV/項目マッピングの確認が必要。 |
| INV-001 | 管理画面静的ファイル調査 | 本番 Django 管理画面で CSS/JS が 404 となる要因を切り分け、静的配信経路と設定の不整合を洗い出す | - | - | - | - | saleslist-docs/ | 調査タスク。リバースプロキシの `/static/` ルーティングと backend 側の collectstatic 結果を確認する。 |
| T-013 | Facebookページ活動状況の推定設計 | Graph API が利用できない Facebook ページについて、スクレイピング可能なメタ情報から活動状況をスコアリングするルールを定義する | 2025-10-31 11:20 JST | - | - | - | saleslist-docs/design/facebook-page-activity-assessment.md | メタタグ・OGタグ・HTTPレスポンスなど複数指標を組み合わせた「アクティブ/非アクティブ/判定不能」の三区分を策定する |
| T-013 | Celery バッチ本番スケジュール | 法人番号・オープンデータ取り込みを本番のみ定期実行し、開発環境は手動トリガー運用とする | - | - | - | - | saleslist-docs/ops | Celery Beat もしくは cron 設定を環境ごとに切り替える設計を検討。 |
| T-014 | AI補完候補値の正規化 | PowerPlexy などの結果をレビュー候補登録前に整形し、数値項目で承認エラーが起きないよう正規化を導入する | 2025-11-03 05:12 JST | 2025-11-03 16:45 JST | feat: derive PowerPlexy usage limits and prioritize never-enriched companies | backend:6a3e4f3 | saleslist-docs/design | established_year・capital などの正規化ルールを定義し、AI補完パイプラインへ組み込む |
| T-014 | AI候補自動取得連携 | Power Plexy 等の AI API を使った候補生成を実装し、レビューキューに連携する | - | - | - | - | saleslist-docs/design | API キー取得後にリクエスト仕様・プロンプト設計をまとめる。 |
| T-015 | ログイン2段階認証実装 | ログイン→メール通知→メール内パスワード入力のフローを追加し、トークンの有効期限を72時間（再操作時に延長）に設定する | 2025-10-30 21:40 JST | 2025-10-30 22:31 JST | 未コミット | 未コミット | saleslist-docs/design/two-factor-auth.md | バックエンドMFA API（pending/verify/resend）とフロントMFA入力導線を実装。Redisキャッシュ・SendGrid連携設定を追加し、APIテストとlintを実施。 |
| T-016 | データ収集履歴画面設計 | Cloneバッチ実行履歴と投入件数・スキップ件数・次回予定を可視化する管理画面の要件整理 | 2025-10-30 23:21 JST | 2025-10-31 00:24 JST | feat: surface AI usage limits and refine user guide delivery | front:0688b55 | saleslist-docs/design/data-collection-history.md | バックエンドAPI・Celery連携・フロント画面を実装し、履歴一覧/手動トリガー/スケジュール表示を統合。 |
| UI-001 | 役職フィルター設計 | 代表取締役・取締役・執行役員・その他で複数選択できる役職フィルター仕様を整理 | 2025-10-14 06:59 |  |  |  | saleslist-docs/design/role-filter.md | 要件整理とUI/バックエンドの設計案作成。 |
| ENV-001 | Docker環境整備 | 3010/8010/5442ポート固定、Celeryワーカー/Beat含むdocker構成の整備 | 2025-10-14 05:18 | 2025-10-14 05:58 | Introduce Celery-based Facebook activity sync / Expose latest Facebook activity in company list | backend:d32826d9, front:904b07b1 | saleslist-docs/dev-port-config.md | 既存composeのポート更新・worker/beat追加、E2E設定更新まで完了。 |
| DOC-001 | 企業最新更新列の仕様設計 | 企業管理画面に最新更新列を追加するため、Facebook友達数・投稿の追跡条件を整理 | 2025-10-14 05:11 | 2025-10-14 06:08 | ドキュメント更新 (active-company-timeline.md) | n/a | saleslist-docs/design/active-company-timeline.md | スケジュール・リスク・フォローアップを追記し仕様確定。 |
| UT-001 | ユースケーステストUT-001設計 | ユースケーステストUT-001の設計 | 2025-10-14 07:08 | 2025-10-14 07:15 | n/a | n/a | saleslist-docs/tests/ut-001-client-ng-project-flow.md |  |
| UT-002 | ユースケーステストUT-001構築 | UT-001設計に基づくテスト実装・環境構築 | 2025-10-14 07:20 | 2025-10-14 07:21 | n/a | n/a | saleslist-front/tests/e2e/ut-001-client-ng-project-flow.spec.ts | UT-001に対応する自動テスト実装。 |
| QA-001 | 企業管理 業界フィルター調査 | 業界フィルターが機能しない件の原因調査 | 2025-10-10 15:17 | 2025-10-10 15:45 | (legacy) Playwright scenario rewrite | 不明 | tests/e2e/companies.spec.ts | 既存タスク。管理外コミットにつき参照なし。 |
| OPS-001 | マスターデータ再投入とE2E再検証 | Industry初期化コマンド実行およびE2E全体の再実行 | 2025-10-10 16:01 | 2025-10-10 16:27 | (legacy) master data seed update | 不明 | masters/management/commands/load_master_data.py | 既存タスク。管理外コミットにつき参照なし。 |
| UI-002 | Service Coverage表示調整 | Service Coverageセクション削除とログ表示改善 | 2025-10-14 17:59 | 2025-10-14 18:14 | n/a | n/a | saleslist-front | プロジェクト履歴ダイアログのログ強化とdocker構成更新。 |
| UI-003 | 役職フィルター不具合調査 | 役職フィルターが効かない原因の特定とUI改善方針検討 | 2025-10-14 18:32 |  |  |  | saleslist-front |  |
| QA-002 | 業界フィルター不一致調査 | CSVインポート業界とフィルター未一致の原因調査、複数業界検索時の動作検証 | 2025-10-16 15:24 | 2025-10-16 16:26 | 業界フィルターを部分一致と検索型UIに刷新 | front:fa6de0c, backend:ccbdcda | saleslist-docs/design/industry-filter-enhancement.md | CSVデータ・API・UIの整合性を確認。 |
| FE-001 | クライアント企業リストエクスポート検討 | クライアント追加企業リストのエクスポート機能および7200件上限の要件整理 | 2025-10-16 15:24 | 2025-10-16 17:10 | クライアントCSVエクスポート実装と上限制御 | front:18f7ed1, backend:d5332fb | saleslist-backend/docs/client-company-export.md | 仕様案と影響範囲を整理。 |
| FE-002 | クライアント企業リストエクスポートUI実装 | クライアント詳細画面から企業リストCSVをエクスポートできるUI実装 | 2025-10-19 16:00 |  |  |  | saleslist-docs/design/client-export-ui.md | 2025-10-19 16:01 設計ドラフト作成。 |
| T-013 | 本番Celeryワーカー常駐構成整備 | `saleslist-infra/docker-compose/prd/docker-compose.yml` に `worker` / `beat` サービスを追加し、GitHub Actions（`vps-deploy.yml`）で backend と同時にビルド・起動するよう更新する | - | - | - | - | saleslist-infra/ | 本番VPS composeとSecrets、Redis設定の確認が必要 |
