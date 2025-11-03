# 法人番号取得パイプライン 設計書

## 1. 背景
- レビュー機能(T-006)で補完候補を扱えるようになったが、実データを投入して検証するためのパイプラインが未整備。
- 法人番号は公的に公開されており確定値として扱えるため、最初の実装対象として適している。
- ルールベースでの取得とレビュー投入を分離し、その他項目（住所・従業員数など）向けのAI候補とは別フローで扱う。

## 2. 目的
1. 国税庁法人番号APIなどの信頼できるソースから法人番号を取得する。
2. 取得結果を `company_update_candidates` に投入し、レビューワークフローに流す。
3. 承認時には自動で `companies.corporate_number` に反映し、履歴を残す。

## 3. 全体構成
```
┌──────────────┐   ┌────────────────────────┐   ┌─────────────────────┐
│ 法人番号API   │ ⇒ │ 収集ジョブ (cron / celery) │ ⇒ │ ingest_corporate... │
└──────────────┘   └────────────────────────┘   └────────┬────────────┘
                                                        │
                                                        ▼
                                           company_update_candidates
                                                        │
                                                        ▼
                                           レビュー画面(`/companies/reviews`)
                                                        │（承認）
                                                        ▼
                                               companies.corporate_number
```

## 4. データソース
- **国税庁 法人番号公表サイト API**
  - エンドポイント: `https://api.houjin-bangou.nta.go.jp/...`
  - パラメータ: 商号・所在地・法人番号など。
  - 利用方式: 拠点の商号リストをキーにして検索し、法人番号を抽出。
  - 仕様: 月間リクエスト制限等があるためキャッシュ・増分取得戦略が必要。
- 将来拡張
  - 官報／各自治体オープンデータ等を追加ソースとして扱う場合は、取得モジュールを差し替え可能にしておく。

## 5. 処理フロー詳細
1. `collect_corporate_numbers.py`（仮称）を cron / Celery Beat から実行。
2. 企業マスタから対象企業を抽出（例: 法人番号未設定のもの）。
3. 国税庁APIを呼び出し、レスポンスの法人番号・商号を整形。
4. `ingest_corporate_number_candidates(entries)` に渡して候補を投入。
5. レビュー画面では「対象フィールド=法人番号」で絞り込み、優先的に承認。
6. 承認時に `companies.corporate_number` を更新し、`company_update_history` に記録。

## 6. バリデーション・重複制御
- `_normalize_corporate_number` でハイフン等を除去して正規化。
- 既に同じ法人番号が設定されている場合はスキップ。
- `create_candidate_entry` 内でハッシュ＋再提案ブロックを利用し、過去に否認された値の再投入を防止。
- 二重投入防止: `value_hash` と `STATUS_PENDING` の組み合わせで重複候補を弾く。

## 7. エラー処理・ロギング
- API呼び出し失敗時はリトライ（指数バックオフ）＋Slack通知（将来的に導入）。
- 候補投入時に何件作成されたか、スキップ理由（既存値・不明など）をINFOログで残す。
- APIレスポンスに法人番号が存在しない場合は WARN ログに残す。

## 8. API / CLI インターフェース
- REST: `POST /api/v1/companies/reviews/import-corporate-numbers/`
  ```json
  {
    "entries": [
      {
        "company_id": 123,
        "corporate_number": "1234-5678-9012",
        "source_detail": "nta-api"
      }
    ]
  }
  ```
  - 戻り値: `created_count`, `batch_ids`
- CLI (Celeryタスク想定): `python manage.py import_corporate_numbers --source nta --target missing`
  - `--source`: 利用する収集モジュール
  - `--target`: 未設定企業のみ、または特定IDのみに限定

## 9. セキュリティ・レート制限
- APIキーが必要な場合は `env/dev/backend.env` に `NTA_CORPORATE_API_KEY` を追加し、Django設定から参照。
- 大量リクエスト時は1秒あたりの呼び出し回数制限を設ける。
- Django設定側では `CORPORATE_NUMBER_API_TOKEN` / `CORPORATE_NUMBER_API_BASE_URL` / `CORPORATE_NUMBER_API_TIMEOUT` / `CORPORATE_NUMBER_API_MAX_RESULTS` を参照する。 `.env` に定義が無い場合は呼び出しをスキップする。

## 10. スケジュール・タスク分割
1. **T-007**: 国税庁APIクライアント実装、法人番号取得ジョブ、管理コマンド整備。
2. **T-008**: フロントの法人番号フィルタ・バッジ表示（完了）。
3. **T-009**: Power Plexy 等を使った非確定項目（住所・従業員数など）の候補生成。

## 11. 今後の改善アイデア
- 取得結果に住所・商号なども含め、レビュー画面で比較表示できるようにする。
- 承認後に他システムへ通知するWebhooksを整備（CRMなど）。
- 取得ソースを複数に拡張し、確信度に応じてレビュー優先度を調整。
- 監視: 候補投入件数や失敗件数をメトリクス化し、ダッシュボードに表示。
