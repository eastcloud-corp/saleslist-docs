# Codex運用ルール

## 基本方針
- 会話は常に丁寧な日本語で行う。
- 記録が必要と判断した場合のみ、本ファイルへ簡潔に追記する。
- タスクや進捗ログは `saleslist-docs/task_tracker.md` で管理する。着手時にタスクを起票し、完了時に完了日時と関連コミットIDを記録する。
- コードをコミット／プッシュする前に、可能な限り以下を実行してエラーがないことを確認する。
  - `npm run lint` / `flake8` などの lint ツール
  - `npm run test` / `pytest` などの自動テスト
  - `npm exec tsc`
  - 必要に応じて `npm run build` 等のビルド確認

## コマンド実行ポリシー
### 許可コマンド
- 明示的に禁止されていないすべてのコマンド
- `npm` / `yarn` / `pip` / `python` / `node`
- `curl` / `wget` などのHTTPリクエスト系
- `docker` / `docker-compose`
- `psql` / `mysql` 等のデータベース操作
- 一般的なファイル操作 (`cp`, `mv`, `cat` など)
- テキスト処理 (`rg`, `grep`, `sed`, `awk` など)

### 禁止コマンド
- `wsl` 関連コマンド

### 開発用途での許可コマンド
- `npm install` / `npm run` / `npm test`
- `python3` / `pip3 install`
- APIテスト目的の `curl` / `wget`
- `docker` / `docker-compose`
- `psql` / `sqlite3`
- ツール経由のファイル読写

## 作業環境設定
### 自動承認対象
- ファイル読取・編集
- APIテスト
- データベースのSELECTクエリ
- コンテナ操作
- npm / pip 等の依存操作

### 個別承認が必要な操作
- ファイル削除を伴うコマンド
- システム全体に影響する変更
- すべてのGit操作
- WSL関連操作

### 環境固有メモ
- フロントエンド開発サーバーは `docker compose ... up -d frontend` で起動し、常にホスト側 `http://localhost:3010` に割り当てる。`docker compose run` で別ポート（3000など）を開けないこと。
- フロントの環境変数を変更したりコードを更新したら、`docker compose -f saleslist-infra/docker-compose/dev/docker-compose.yml build frontend` → `... up -d frontend` を実行してイメージを再ビルドする。ビルドが難しい場合は代わりに `docker compose -f saleslist-infra/docker-compose/dev/docker-compose.yml run --rm --service-ports frontend npm run dev -- --hostname 0.0.0.0 --port 3010` を使う。
- バックエンドは `http://localhost:8010`、PostgreSQL は `localhost:5442`、Redis は `localhost:6380` を固定ポートとする。テスト実行時もこのポート設定を前提にしているため、compose 起動前に別用途で占有していないか確認すること。

---
本ファイルはCodexエージェントの運用ルールのみを記載する。その他の仕様は以下を参照すること。
- `saleslist-docs/task_tracker.md`: タスク起票・設計書のリンク・完了コミットIDの記録。
- `saleslist-infra/README.md`: 
  - 「ローカル開発環境の起動」にセットアップ・起動・停止手順。
  - 「デプロイメント」「ヘルスチェック」に本番／ステージングのデプロイおよび稼働確認手順。必要に応じて `saleslist-infra/scripts/` 配下のスクリプトも参照。
- `saleslist-docs/design`: 設計書格納ディレクトリ 
- `saleslist-docs/bugfix`: バグ修正方針格納ディレクトリ
