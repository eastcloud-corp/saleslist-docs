# 開発環境ポート設定メモ

開発スタックを `docker compose -f saleslist-infra/docker-compose/dev/docker-compose.yml up -d db redis backend worker beat frontend` で起動した際のデフォルト公開ポートは以下の通りです。

| サービス | コンテナ内ポート | ホスト側デフォルト | 備考 |
| --- | --- | --- | --- |
| Next.js Frontend (`saleslist_frontend`) | 3000 | `${FRONTEND_PORT:-3010}` | `FRONTEND_PORT` を指定しない場合は **3010** で待ち受け。競合する場合は `FRONTEND_PORT=3002` などを指定して上書き可能。 |
| Django Backend (`saleslist_backend`) | 8000 | 8010 | API エンドポイントは `http://localhost:8010` でアクセス。 |
| PostgreSQL (`saleslist_db`) | 5432 | 5442 | アプリ側の接続情報は `saleslist_backend/settings.py` に準拠。 |
| Redis (`saleslist_redis`) | 6379 | 6380 | 開発用途のキャッシュ。 |
| Celery Worker / Beat | - | - | ポート公開は行わず、バックエンドと Redis 経由で連携します。 |

> **メモ:** Playwright や各種 E2E 仕様書では `http://localhost:3010` を前提としているため、上記デフォルトと整合しています。ローカルで別プロジェクトとポートが競合する場合は、起動前に `FRONTEND_PORT`（および必要に応じて `BACKEND_PORT` を導入）を設定した上で `docker compose` を実行してください。

環境変数の例:

```bash
# フロントエンドを 3002 で起動する例
FRONTEND_PORT=3002 docker compose -f saleslist-infra/docker-compose/dev/docker-compose.yml up -d db redis backend worker beat frontend
```

将来的にポートを固定したい場合は、`saleslist-infra/docker-compose/dev/docker-compose.yml` の `ports` セクションを直接書き換える運用でも問題ありません。
