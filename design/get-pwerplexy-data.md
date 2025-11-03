1. 概要

本機能は、営業リスト内の欠損データ（代表者名・設立年・資本金・所在地など）を自動補完するバッチ処理である。
AIによる情報補完には Powerplexy API を使用し、ルールベース＋AIベースのハイブリッド補完 を実施。
Celery によりスケジュール制御し、App Run 環境（さくらのクラウド）上で日次実行を行う。

2. 処理フロー
1. Celery Beat が毎日AM3:00にタスクを起動
2. Redisキャッシュから当月のAPI使用量を取得
3. 当月使用額 < $20 かつ リクエスト数 < 5000件 の場合のみ実行
4. Django ORM経由で営業リストを取得
5. 各レコードの欠損項目を検出
   ├─ ルールベースで取得（metaタグ／正規表現）
   └─ 不足分を Powerplexy API に問い合わせ
6. JSON出力を解析してDBを更新
7. Redisのカウンタを更新
8. Slack / 管理者メールへ完了通知

3. システム構成図
+------------------------------------------+
| Sakura Cloud App Run                    |
|------------------------------------------|
| Django REST Framework (API)              |
| Celery Worker                            |
| Celery Beat (Scheduler)                  |
| Redis (Broker + Cache)                   |
+------------------------------------------+
          │
          ▼
   Powerplexy API (https://api.perplexity.ai)

4. Djangoアプリ構成（ディレクトリ）
/app/
 ├── manage.py
 ├── project/
 │   ├── settings.py
 │   ├── celery.py
 │   └── urls.py
 ├── core/
 │   ├── config.py              # APIキー・閾値設定
 │   ├── powerplexy_client.py   # Powerplexy API呼出し
 │   ├── redis_usage.py         # 月間使用量管理
 │   ├── enrich_rules.py        # 欠損検出＋プロンプト生成
 │   ├── notify.py              # Slack/メール通知
 │   └── utils.py
 ├── leads/
 │   ├── models.py              # 営業リストモデル
 │   ├── tasks.py               # Celeryバッチタスク
 │   ├── serializers.py
 │   ├── views.py
 │   └── admin.py
 └── requirements.txt

5. データ構造
モデル：Lead（営業リスト）
カラム名	型	補足
name	CharField	企業名
corporate_number	CharField	法人番号
industry	CharField	業種
contact_person_name	CharField	担当者名
contact_person_position	CharField	役職
facebook_url	URLField	SNS URL
tob_toc_type	CharField	toB / toC
business_description	TextField	事業概要
prefecture / city / location	CharField	所在地情報
employee_count	IntegerField	従業員数
revenue	DecimalField	売上高
capital	DecimalField	資本金
established_year	IntegerField	設立年
website_url	URLField	公式サイト
contact_email	EmailField	連絡先
phone	CharField	電話番号
notes	TextField	備考
ai_last_updated	DateTimeField	最終AI補完日時
ai_source	CharField	“rule” / “ai” / “manual”
6. API構成（Powerplexy）
項目	値
URL	https://api.perplexity.ai/query
Method	POST
Header	Authorization: Bearer ${API_KEY}
Model	sonar-medium
Content-Type	application/json
Rate Limit	60 req/min（Celery内で制御）
7. プロンプト仕様（JSON限定出力）
次の企業情報をJSONで返してください。
企業名: 株式会社Lis
URL: https://lis-sales-japan.jp/about/
欠損項目: 設立年, 資本金, 従業員数, 所在地
出力形式:
{
  "設立年": "",
  "資本金": "",
  "従業員数": "",
  "所在地": ""
}


→ 出力はJSON限定とし、文章生成を抑制（トークン節約）。

8. Redisによるコスト・件数管理
Redisキー構造
キー	例	値	説明
usage:2025-10:calls		int	当月のAPI呼び出し数
usage:2025-10:cost		float	当月の推定課金額（USD換算）
ルール

1リクエストあたり $0.05（推定値）

呼び出し前に両方をチェック
→ cost < 150.0 and calls < 3000 の場合のみ実行

呼び出し後に +1件、+0.05USD をインクリメント

9. Celery設定
celery.py

Broker：Redis (redis://localhost:6379/0)

Queue名：ai_enrich

Rate Limit：10/m

Beatスケジュール：毎日 AM3:00

tasks.py

タスク名：leads.tasks.run_ai_enrich

ログ：logs/ai_enrich_YYYYMMDD.log

成功時・エラー時通知：Slack / メール（notify.py）

10. タスク処理ロジック（擬似フロー）
def run_ai_enrich():
    usage = redis_usage.get_current_month_usage()
    if usage.cost >= 150 or usage.calls >= 3000:
        notify.warn("⚠️ Powerplexy API上限到達")
        return

    leads = Lead.objects.filter(needs_ai_update=True)
    # 実装では ai_last_enriched_at が null の企業を優先する order_by を付与している
    for lead in leads:
        missing_fields = detect_missing_fields(lead)
        if not missing_fields:
            continue

        # ルールベース抽出
        enriched = try_rule_based_scrape(lead)
        if not is_complete(enriched):
            # Powerplexy補完
            prompt = build_prompt(lead, missing_fields)
            result = query_powerplexy(prompt)
            lead.update_from_ai(result)
            redis_usage.increment(cost=0.05)

    notify.success(f"AI補完完了: {len(leads)}件")

11. 通知設計
通知種別	タイミング	内容
成功通知	バッチ完了時	実行件数、残リクエスト数、推定残コスト
警告通知	上限到達時	「月額150USDまたは3,000件超過」
エラー通知	APIエラー時	スタックトレース、対象企業名

Slack Webhook または SMTP（SendGrid等）で送信。

12. 運用ルール
項目	内容
実行頻度	1回／日（03:00）
最大処理件数	`POWERPLEXY_DAILY_RECORD_LIMIT` 件／日（未設定時は月間上限を当月日数で除算）
上限設定	月額150USDまたは3,000件（`POWERPLEXY_*` 環境変数で調整可能）
バックオフ	API429時は 10分スリープ再試行
リトライ	最大3回（Celery retry機構使用）
13. 成果物
ファイル名	内容
core/powerplexy_client.py	Powerplexy呼び出し処理
core/redis_usage.py	月間使用量管理モジュール
core/enrich_rules.py	欠損項目検出とプロンプト生成
leads/tasks.py	Celeryバッチ本体
core/notify.py	Slack通知機能
celery.py	Celery設定
scheduler.py	Beatスケジュール設定
14. 今後の拡張
項目	内容
✳ DBキャッシュ化	AI出力をRedisキャッシュ＋PostgreSQLに永続化
✳ 手動再取得API	/api/leads/{id}/refresh/ で個別再取得可能に
✳ 出力精度向上	AI回答信頼度スコアの導入
✳ トークンコスト自動計測	Powerplexyのレスポンスサイズから動的計算
