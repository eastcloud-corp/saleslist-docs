# 企業データ補完レビュー機能 設計書

## 背景
- 保有する企業データに不足項目・古い情報が散見される
- Facebook 等から自動取得するのは規約上困難なため、許諾不要・公開可能なソースを組み合わせて補完する
- AI (Power Plexy 等) とルールベースで集めた候補情報を人がレビューし、正しい項目のみ反映させる仕組みを整備する

## 機能概要
1. **候補取得レイヤー**
   - *ルールベース*:
     - 公式サイトの RSS / ニュース更新日、PR Times API、求人 API など、明示的に許可されたソースから候補を抽出
     - フォーマットが一定で決定的に取得できる項目を対象（所在地、URL、メール、最新更新日など）
   - *AIベース* (Power Plexy 等):
     - サイト本文や資料から「事業概要」「提供サービス分類」「従業員数」など曖昧な情報を抽出
     - AI 候補には信頼度スコアを付与（例: 0〜100）
   - 取得結果は `company_update_candidates` に保存
     - 例: `company_id`, `field`, `candidate_value`, `source_type` (RULE / AI), `confidence`, `collected_at`, `hash`
     - 同一候補の重複防止のため値ハッシュを保持

2. **レビューキュー生成ロジック**
   - 同一企業で複数項目の候補が揃った場合は同じチケットに束ねる
   - 状態: `pending_review` → `approved` / `rejected` / `partial`
   - `company_review_batches` テーブルでチケット単位、`company_review_items` で項目単位に管理
   - 既に却下された値が再度提案された場合は過去の却下理由を表示しつつ再レビュー判断可

3. **レビューページ (UI)**
   - **一覧**:
     - 企業名、候補件数、最新候補取得日時、ソース (RULE/AI)、信頼度平均
     - フィルタ: ソース種別、信頼度閾値、取得日、担当者未割当
   - **詳細**:
     - 現行データと候補の比較表示、差分ハイライト
     - 項目ごとに「承認」「否認」「修正して承認」を選択
     - 一括承認 / 一括否認、AI 候補は信頼度表示と警告 (低スコア)
     - 承認・否認理由の入力欄 (任意)
     - 同時レビュー対象の項目が複数ある場合はタブ or アコーディオンで表示
   - **承認結果の反映**:
     - `company_update_history` に履歴保存 (`approved_by`, `approved_at`, `field`, `old_value`, `new_value`)
     - 本体 `companies` テーブルの対象項目を更新
     - UI 上で「最終更新者」「更新日時」「更新ソース」を表示

4. **ワークフロー**
   - (1) バッチ/AI が候補収集 → `company_update_candidates` に反映
   - (2) 企業単位でレビューキューを生成 → `company_review_batches` のステータスを `pending`
   - (3) レビュアーが UI で承認・否認
   - (4) 承認時に `companies` テーブル更新、否認時は候補に `rejected` を記録
   - (5) 反映後、アクティビティログ/通知に出力（Slack 通知等）
   - (6) バッチは一定期間経過した否認項目は再提案検討（例: 30日後に再収集）

5. **注意点・補足**
   - 法人番号などの確実な識別子が無い企業は AI だけで判定が難しい
     - レビューで人が最終確認することを前提とする。
     - 評価履歴を蓄積して今後のモデル改善に活用。
   - 企業情報の収集は公開データ・許諾済みデータに限る。Facebook のスクレイピングは禁止。
   - バックアップ・ロールバックを容易にするため、更新履歴テーブルに過去値を保存。
   - 既存の CRM/SFA にも反映が必要な場合は、API 連携や Webhook 等で通知設計。

6. **今後のタスク**
   1. 補完対象項目の定義とデータソースの棚卸
   2. ルールベース取得スクリプトの実装 (RSS, PR Times, 求人 API 等)
   3. Power Plexy など AI API のプロトタイプ連携
   4. `company_update_candidates` / `company_review_batches` / `company_review_items` / `company_update_history` テーブルスキーマ設計
   5. レビューページの UI ワイヤーフレーム作成
   6. API 実装 (候補取得 → レビュー → 承認反映)
   7. ログ/通知設計 (Slack, メール)
   8. 運用フロー整備 (担当者、SLA、再提案ルール)

## 8. 開発環境向け補完候補トリガー
- API: `POST /api/v1/companies/reviews/generate-sample/`
  - 任意で `company_id` を指定可能。未指定時は最新更新企業から選択。
  - ルールベース候補をダミー生成し、レビューキューに投入する。
- フロントエンド `/companies/reviews` に「サンプル候補生成」ボタンを実装済み。
  - 開発環境でボタンを押す -> API をコール -> 新しい候補が一覧に反映される。
  - 実運用に切り替える際は、この API を実際の取得バッチ/ETL に置換。

## 9. 否認済み項目の再提案ポリシー
- 現状は否認された候補も新しい候補として再登録される（過去レコードは残す）ため、レビュアーは前回コメントを参照しつつ再判断可能。
- 実運用では以下の選択肢を検討:
  - **自動再提案**: 一定期間（例: 30日）経過、または別ソースで再取得した場合のみ再レビュー。
  - **手動再提案**: レビュアーが「再提案許可」フラグを設定した項目のみリストに戻す。
  - **AI活用**: 否認理由を学習データとして蓄積し、今後の候補生成条件や信頼度を改善。
- `CompanyUpdateCandidate` の `status` / `rejected_at` を使えば、否認済みかいつ否認されたか判定可能。再提案フローに組み込む。

## 10. 実装フォローアップ (T-006 後続)
- 実際のルールベース／AI 連携は T-006 の後続タスクで実装予定であり、現状は開発検証用のサンプル生成 API で代替している。
- 本番運用ではダミー API を実バッチに置換し、否認済み項目の再提案ロジック（再提案タイミングや別ソース確認など）を組み込む。
- 否認時に `CompanyUpdateCandidate.block_reproposal` と否認理由コード (`rejection_reason_code`) を蓄積し、同じ値が再提案されないようにフィルタする。レビューダイアログでは「同じ値を再提案しない」チェック・理由選択・補足メモを入力できる。

## 11. 法人番号専用パイプライン（T-007）
- 目的: 公的データソース等から法人番号のみを抽出し、確度 100% の候補としてレビューに載せる。その他項目とは候補投入経路を分離し、誤提案時の影響を最小化する。
- 候補投入:
  - 新規サービス `generate_corporate_number_candidates(company_ids?: list)` を実装。
  - 取得値は `field='corporate_number'`, `source_type='RULE'`, `confidence=100`, `source_detail` に取得元 URL やAPI名を格納。
  - 既存の `create_candidate_entry` を利用し、重複検知・再提案ブロックと連携。
- レビュー動線:
  - フロントエンドに「法人番号のみ」フィルタ／タブを追加し、レビュアーが優先的に処理できる導線を用意。
  - 異なる企業で同じ法人番号が提案された場合は、否認理由コード `mismatch_company` のテンプレートを標準提示。
- 運用:
  - バッチ化（例: 夜間ジョブ）で定期的に法人番号候補を取得。
- 承認後は `companies.corporate_number` を更新し、履歴 (`company_update_history`) へ記録。
- 付随して法人番号をキーに外部システム連携を検討（将来的な拡張）。

## 12. ルールベース候補の採用ソース（確定）
- **国税庁 法人番号 Web-API (拡張版)**  
  - JSON を利用し、法人番号・所在地・商号・資本金等の確定値を取得。  
  - Celery バッチで日次〜週次実行、レート制限（1 日 5,000 件／ 2 秒インターバル）とクールダウンを遵守。  
  - 候補には `source_type=RULE`, `source_detail=nta_corporate_api` を付与し、`value_hash` は正規化後の値で算出する。  
- **認証不要の自治体オープンデータ**  
  - 第 1 弾として以下 3 件を採用。ダウンロード URL は `config/opendata_sources.yaml` に登録し、CSV→正規化→候補投入のフローを構築する。  
    1. 東京都オープンデータ「法人会員企業一覧」 (`法人番号`,`商号`,`所在地`,`資本金`,`業種`,`URL` など)  
    2. 静岡県オープンデータ「企業情報データベース」 (`法人番号`,`所在地`,`業種`,`従業員数`,`URL`,`電話番号`)  
    3. 大阪府オープンデータ「中小企業サポート企業情報」 (`法人番号`,`所在地`,`資本金`,`従業員数`,`業種コード`,`URL`)  
  - これらの CSV は認証不要で取得でき、公式サイト URL を含むため `companies.website_url` の補完にも利用する。  
  - 住所は DB 設計どおり（都道府県＋市区町村＋番地等）に正規化し、電話番号はハイフンを除去したうえでハッシュ生成・候補提示を行う。  
- **名寄せとレビュー方針**  
  - 法人番号が一致する場合は直接候補化。法人番号が無いレコードは企業名ベースでマッチングし、レビュー（人による確認）を前提に候補提示する。  
  - 全候補で `source_detail` に元データセット名／配布元 URL を保持し、レビュアーが出典を確認できるようにする。

## 7. 詳細設計（ルールベース先行）

### 7.1 対象項目とルール例
| 項目 | 既存データ | ルールベース補完 | 備考 |
| --- | --- | --- | --- |
| 住所 | `companies.address` | 企業公式サイトの「会社概要」ページの構造化情報／公開 PDF の表抽出（厳密にスクレイピング禁止を遵守し、許可済みデータのみ） | 曖昧な場合は候補提示のみ |
| 電話番号 | `companies.phone` | 同上。正規表現で電話番号フォーマットを抽出 | 正規表現検証で不正記号除外 |
| メールアドレス | `companies.email` | 公式サイト上の `mailto:` や問い合わせ先のテキスト | 迷惑メール対策のため公開頻度が低いので候補扱い |
| 公式ウェブサイトURL | `companies.website` | 企業リストに登録済みの URL を基準に最新 URL を確認／HTTP リダイレクトで更新 | HTTP ステータス 200 のみ候補 |
| ニュース更新日 | `company_external_updates` | RSS / API から取得した最新投稿日時 | リンク付きで表示 |
| 従業員数等 | `companies.employee_count` | API 提供されている公開データ（商工会議所等）から取得 | データソースに注意 |
| 求人状況 | `company_external_updates` | API が公開されている求人サイトから新着有無を取得 | ロールごとのフィルタ

### 7.2 データモデル案
```
company_update_candidates
- id
- company_id
- field          (住所, 電話, employee_count, etc)
- candidate_value
- value_hash
- source_type    (RULE, AI)
- source_detail  (rss_url, api_name, etc)
- confidence     (0-100, RULEはデフォルト100)
- collected_at
- status         (pending, merged, rejected, expired)

company_review_batches
- id
- company_id
- status (pending, in_review, approved, rejected, partial)
- created_at
- updated_at
- assigned_to (optional)

company_review_items
- id
- batch_id
- candidate_id
- field
- current_value
- candidate_value
- confidence
- decision (pending, approved, rejected, updated)
- comment
- decided_by
- decided_at

company_update_history
- id
- company_id
- field
- old_value
- new_value
- source_type (RULE, AI, manual)
- approved_by
- approved_at
- comment
```

### 7.3 UI ワイヤーフレーム概要
1. **レビュー一覧ページ**
   - カラム例: 企業名 / 候補件数 / 最終取得日時 / ソース種別統計 / ステータス / 担当者
   - 行アクション: 詳細へ / 一括承認 / 担当者割当
   - フィルタ: ソース (RULE/AI), 信頼度 (RULE=100, AI=閾値設定), 更新日, ステータス

2. **レビュー詳細モーダル（またはページ）**
   - 左ペイン: レビュー対象企業の基本情報（社名、現在値、最終更新日）
   - 右ペイン: 項目ごとの候補カード
     - 既存値 vs 候補値をテーブル表示
     - 候補が複数ある場合: タブ or アコーディオン
     - 「承認」「否認」「編集して承認」ボタン
     - ソース表示（アイコン: ルール / AI）、証跡リンク（RSS、APIレスポンス）
     - AI 候補には信頼度＋注意メッセージ
   - 下部に「承認した項目だけ反映」の確認セクション

3. **レビュー結果の反映後**
   - 成功トースト + `company_update_history` への反映
   - 企業詳細ページの「最新更新履歴」に今回の変更を追加表示

### 7.4 ワークフロー詳細（ルールベースのみ先行）
1. 夜間バッチでルールベース収集 → `company_update_candidates` へ挿入（status=pending）
2. 同一company_idで`pending`の候補が一定数ある、または一つでもあれば `company_review_batches` にエントリ作成
3. レビュアーが UI で承認
4. バックエンドAPI `/companies/{id}/review/approve` に承認結果を POST
5. サービス側で `companies` テーブル更新、`company_update_history` に記録、`company_update_candidates` の status を `merged` に更新
6. 否認された候補は `rejected` とし、理由コメントを保存（再提出ロジックに活用）
7. Slack/メール通知（承認時、要フォロー時など）

### 7.5 今後の AI 連携 (メモ)
- Power Plexy などからの提案は `source_type=AI` として、confidence スコアを参照。
- レビューログを AI モデル改善にフィードバックできるよう設計。
- 法人番号がない場合の誤認防止として「候補値の類似度チェック」などを検討。
