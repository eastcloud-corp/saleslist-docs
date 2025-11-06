# デプロイ用タグリリース手順

本番デプロイは GitHub Actions が `deploy-*` 形式のタグに反応して実行されます。  
タグは GitHub Releases から発行し、リリースノートに変更点を記録してください。

## 1. 事前チェックリスト

1. `saleslist-backend` / `saleslist-front` / `saleslist-infra` / `saleslist-docs` の `main` が最新であること。
2. CI（lint / test）が通過していること。
3. デプロイ対象コミットがステージングで確認済み、または承認済みであること。
4. Secrets / 環境変数の更新が必要な場合は、リリース前に設定を完了しておくこと。

## 2. リリースタグの作成手順（Web UI）

1. `saleslist-infra` リポジトリを開く。
2. 右側メニューの **“Releases”** → **“Draft a new release”** をクリック。
3. **“Choose a tag”** で新しいタグ名を入力し、`Create new tag` を選択。  
   - フォーマット例: `deploy-20251107`, `deploy-20251107-1`.  
   - 対象ブランチは `main` を指定。
4. **Release title / Description** にデプロイ内容・要約・確認者などを記載。
5. 必要に応じてチェックボックス（Pre-release 等）を設定。
6. **“Publish release”** を押すとタグが作成され、GitHub Actions が起動する。

> **Note**: CLI からタグを push する場合も `deploy-*` フォーマットに揃えること。  
> 例: `git tag deploy-20251107 && git push origin deploy-20251107`

## 3. デプロイ後の確認

1. GitHub Actions → `Deploy to Sakura VPS` ワークフローで実行ログを確認。
2. VPS 側でコンテナが最新化されているか、必要に応じて `docker compose ps` などで状態確認。
3. アプリケーションのヘルスチェック (`./scripts/health-check.sh production`) を実行。
4. 重要変更の場合は関係者に周知（Slack / Teams 等）。

## 4. トラブルシューティング

| 症状 | 対処 |
| --- | --- |
| ワークフローが動かない | タグ名が `deploy-` で始まっているか確認。手動実行 (`workflow_dispatch`) も利用可。 |
| デプロイ失敗（GA エラー） | 実行ログを確認し、該当ステップでのエラー内容に応じて再実行または修正を行う。Secrets の権限エラーにも注意。 |
| ロールバックしたい | 直前の安定タグを再リリースするか、当該タグを削除し、安定版タグを `deploy-*` で発行し直す。 |

## 5. 運用メモ

- タグ命名は日付ベースを基本とし、同日複数回のデプロイは `deploy-20251107-2` のように末尾番号を振る。
- リリースノートには以下を最低限記載:
  - 対象コミット（バックエンド / フロント / ドキュメント）
  - 主要変更点
  - 確認者 / 実施者
- 緊急デプロイ時は `workflow_dispatch`（手動実行）も利用できるが、ログを残すため可能な限りタグを発行する。
