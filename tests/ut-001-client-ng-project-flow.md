# UT-001 クライアントNG・案件統合フロー ユースケーステスト設計

## 1. テスト目的
- クライアント別NGリスト管理と案件企業選定機能が業務フローどおりに動作することを検証する
- NG判定済み企業が案件追加フローで適切に除外されることを確認する
- 案件一覧ページのインライン編集ロックが複数ユーザー間で正しく制御されることを確認する

## 2. 対象範囲
- フロントエンド
  - `/clients/{id}` NGリストタブ
  - `/clients/{id}/select-companies`
  - `/projects` （インライン編集機能）
- バックエンド API
  - `POST /clients/{id}/ng-companies/import`
  - `POST /clients/{id}/ng-companies/search-add`
  - `DELETE /clients/{id}/ng-companies/{ng_id}`
  - `GET /clients/{client_id}/available-companies`
  - `POST /projects/{project_id}/add-companies/`
  - `POST /projects/page-lock/` / `DELETE /projects/page-lock/`

## 3. 関連仕様・参照資料
- `saleslist-backend/docs/client_ng_management_spec.md`
- `saleslist-backend/docs/correct_business_flow.md`
- `saleslist-front/tests/e2e/client-ng-project-workflows.spec.ts`
- `saleslist-front/tests/e2e/ut-001-client-ng-project-flow.spec.ts`
- `saleslist-front/app/clients/[id]/select-companies/page.tsx`
- `saleslist-front/app/projects/page.tsx`

## 4. 前提条件・環境
- Docker コンテナ（`docker-compose.hotreload.yml`）でフロント・バックエンド・Redis を起動済み
- `.env` 設定によりフロントエンドから API へアクセス可能
- 管理者ユーザー（例: `salesnav_admin@budget-sales.com` / `salesnav20250901`）でログインできること
- 補助ユーザーを追加作成できる権限があること
- Playwright 等の自動テストがロック取得に干渉しない状態（必要なら停止）

## 5. テストデータ準備
テストごとに汚染を避けるため、以下のような一時データを事前に作成する。API での投入例を示す。

### 5.1 クライアント作成
```bash
curl -X POST "$API_BASE/clients/" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "UT001 クライアント",
    "industry": "IT・ソフトウェア",
    "contact_person": "テスト担当",
    "contact_email": "ut001-client@example.com"
  }'
```

### 5.2 企業マスタ作成（少なくとも4社）
```bash
for name in "UT001-マッチ" "UT001-NG検索" "UT001-候補A" "UT001-候補B"; do
  curl -X POST "$API_BASE/companies/" \
    -H "Authorization: Bearer $ADMIN_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{
      \"name\": \"${name}\",
      \"industry\": \"IT・ソフトウェア\",
      \"prefecture\": \"東京都\",
      \"city\": \"千代田区\",
      \"employee_count\": 120,
      \"revenue\": 500000000,
      \"established_year\": 2018,
      \"website_url\": \"https://example.com\"
    }"
done
```

### 5.3 案件作成（クライアント紐付け）
```bash
curl -X POST "$API_BASE/projects/" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "UT001 既存案件",
    "client_id": <client_id>,
    "description": "UT-001用の検証案件"
  }'
```

### 5.4 補助ユーザー作成（排他検証用）
```bash
curl -X POST "$API_BASE/users/" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "ut001-secondary@example.com",
    "name": "UT001 副ユーザー",
    "password": "Passw0rd!001",
    "role": "user"
  }'
```

> **メモ:** テスト完了後は `DELETE /clients/{id}/`、`DELETE /projects/{id}/`、`DELETE /companies/{id}/` でデータをクリーンアップする。

## 6. シナリオ

### 6.1 シナリオA: NGリスト管理
| Step | 操作 | 期待結果 |
|------|------|----------|
| A-1 | 管理者でログインし `/clients/{client_id}` を開く | クライアント詳細が表示される |
| A-2 | `NGリスト` タブを選択 | `登録済みNGリスト (0件)` が表示される |
| A-3 | `CSVテンプレート` をダウンロードし、`UT001-マッチ` と `UT001-CSV未登録` を記載したCSVを用意する | CSVが作成できる |
| A-4 | `CSVインポート` から CSV をアップロード | トースト「インポート完了」「2件のNGリストを登録しました（マッチ: 1件）」が表示され、一覧に2行追加される |
| A-5 | `企業検索` ボタンで `企業検索からNG企業追加` ダイアログを開き、`UT001-NG検索` で検索 → 選択 → NG理由「競合企業」入力 → `NGリストに追加` | トースト「追加完了」「UT001-NG検索をNGリストに追加しました」が表示され、一覧に3行目が追加される。マッチ/未マッチのカウンターが更新される |
| A-6 | `UT001-CSV未登録をNGリストから削除` ボタンを押し確認ダイアログで承諾 | トースト「削除完了」が表示され、対象行が削除され、未マッチ数が減少する |

### 6.2 シナリオB: 企業選択画面でのNG判定
| Step | 操作 | 期待結果 |
|------|------|----------|
| B-1 | クライアント詳細のヘッダーから `営業対象企業を選択` をクリック | `/clients/{client_id}/select-companies` に遷移する |
| B-2 | テーブルのロード完了を待ち、検索欄に `UT001-NG検索` を入力 | NG対象行のみが表示され、チェックボックスは disabled、`NG` バッジと警告アイコンが表示される |
| B-3 | NG対象行のチェックボックスをクリックしようとする | 選択されずトースト「選択できません」「この企業はNGリストに登録されています」が表示される |
| B-4 | 検索欄をクリアし、`UT001-候補A` と `UT001-候補B` 行が表示されていることを確認 | 2社とも「選択可能」バッジで表示される |

### 6.3 シナリオC: 既存案件への企業追加
| Step | 操作 | 期待結果 |
|------|------|----------|
| C-1 | `UT001-候補A` と `UT001-候補B` のチェックボックスをオン | 右上バッジが「2社選択中」になり、選択状態が保持される |
| C-2 | `既存案件を選択` プルダウンから「UT001 既存案件」を選ぶ | プルダウンに案件名が表示される |
| C-3 | `既存案件に追加` をクリック | トースト「成功」「2社を案件に追加しました」が表示され、`/projects/{project_id}` に遷移する |
| C-4 | 案件詳細テーブルに `UT001-候補A/B` が出現することを確認 | 追加された企業が一覧に表示される |

### 6.4 シナリオD: 案件ページのインライン編集と排他制御
| Step | 操作 | 期待結果 |
|------|------|----------|
| D-1 | 管理者で `/projects` を開き、`UT001 既存案件` を含むリストを表示 | 案件行が読み込まれる |
| D-2 | `編集モード` ボタンをクリック | トースト「編集モードを開始しました」が表示され、ボタンが「編集完了 (OFF)」表示に変わり、対象行が編集状態になる |
| D-3 | 別ブラウザ/シークレットで補助ユーザーがログインし `/projects` を開く | 同じ案件リストが表示される |
| D-4 | 補助ユーザー側で `編集モード` をクリック | トースト「編集モードを開始できません」「このページはUT001 副ユーザーが編集中です」が表示され、ボタンは `編集モード (ON)` に戻る |
| D-5 | 管理者側で `編集完了` を押してロックを解放 | トースト「編集モードを終了しました」が表示され、補助ユーザー側で再度 `編集モード` が有効になることを確認 |

## 7. バリエーション・異常系観点
- CSVに存在しない企業名のみを含めた場合、未マッチカウントが増加し一覧に `matched=false` で表示されること
- NG登録済み企業を再度 CSV で取り込んだ際、重複エラーが UI で表示されること
- 企業選択画面で `NG企業を除外` スイッチをオンにすると NG 行がリストから除外されること
- `既存案件に追加` 実行時にネットワークエラーを発生させた場合、トーストが `エラー / 企業追加に失敗しました` になること
- インライン編集モードで 30 分以上経過した際、ロックがタイムアウトし再取得が必要になること（ログ `perf.projects.page_lock.acquire` で確認）
- 補助ユーザーが編集モードを開始した状態で管理者が同様に試みた場合も 409 が返り、UI が同様のエラートーストになること

## 8. 検証ログ・モニタリング
- バックエンド `saleslist-backend/logs/app.log` に以下イベントが出力されること
  - `perf.clients.ng_import`（CSV import）
  - `perf.clients.ng_delete`（NG削除）
  - `perf.projects.page_lock.acquire` / `.release`（ロック取得・解放）
- DB 確認（任意）: `client_ng_companies` に `matched` および `company_id` が正しく紐付くこと、`project_companies` に追加企業が作成されること
- フロントエンド Console にエラーが発生しないこと

## 9. フォローアップ・未決事項
- 新規案件作成フロー（`案件作成して追加`）は今回対象外。別テスト ID での検証が必要
- `NG企業を除外` スイッチと複合フィルタの組み合わせテストは範囲拡大時に追加
- 既存自動テスト（Playwright `client-ng-project-workflows.spec.ts`）との重複範囲を定期的に見直し、手動テストでのみ担保する観点を明確化すること
