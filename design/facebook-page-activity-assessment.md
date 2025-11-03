# Facebookページ活動状況 推定ロジック設計

## 概要

Graph API で他社 Facebook ページの詳細を取得できないケース（権限不足・地域制限・取得対象がビジネスアカウントでない等）向けに、ブラウザで取得可能な公開 HTML 情報からページの活動状況を推定する。スクレイピングはあくまで補助的な判定手段とし、結果は「アクティブ推定」「非アクティブ疑い」「判定不能」の 3 区分で返却する。

## 目的

- Facebook 公式 API が利用できない場合でも、営業リストの鮮度を保つための指標を得る。
- 再調査・手動判断を減らすため、機械的に抽出できるシグナルを整理しスコアリング基準を明文化する。
- 判定の信頼度とリスク（規約違反・誤判定）を把握し、運用時の注意事項を明確にする。

## スコープ

- 入力: Facebook ページ URL（公開ページを想定。個人プロフィールは対象外）
- 取得手段: ヘッドレスブラウザによる HTML レンダリング結果（ログインなし／匿名アクセス）
- 出力: 下記三値と補助情報
  - `status`: `ACTIVE`, `INACTIVE_SUSPECTED`, `UNKNOWN`
  - `score`: 0〜100 の信頼度スコア
  - `signals`: 判定に利用したシグナルの一覧（後述）
  - `observed_at`: 判定実施日時

## 取得するシグナル

| カテゴリ | シグナル | 取得方法 | メモ |
| --- | --- | --- | --- |
| HTTP 応答 | ステータスコード | `response.status()` | 200 以外（特に 404/410/451/5xx）は非アクティブ寄り。429/503 は判定不能扱い |
| HTTP 応答 | 最終リダイレクト URL | `response.url()` | `login` や `checkpoint` が含まれる場合はログイン壁の可能性 |
| DOM / メタタグ | `<title>` | `document.title` | 「Facebook」だけ等テンプレ化されたタイトルを検出 |
| DOM / メタタグ | `meta[name=description]` `meta[property=og:description]` | `document.querySelector` | 「Page Not Available」「このコンテンツは利用できません」など固定文言であれば非アクティブ疑い |
| DOM / メタタグ | `meta[property=og:image]` | `document.querySelector` | 画像 URL が正常（`safe_image.php` 以外）ならアクティブ寄り |
| DOM / メタタグ | `meta[property=og:url]` | `document.querySelector` | 正常な URL （ページID/ユーザー名）を含むか |
| DOM 本文 | CTA ボタン (`data-testid="page-call-to-action"`) | `document.querySelectorAll` | ボタンが存在すれば公開中の可能性が高い |
| DOM 本文 | 投稿タイムスタンプ (`data-utime` 属性) | `document.querySelectorAll('[data-utime]')` | 最も新しい時刻が 90 日以内ならアクティブ寄り。取得できなければスコアに影響させない |
| DOM 本文 | エラーメッセージ (`.uiInterstitialContent`, `div[data-testid="error_detail_code"]`) | `document.textContent` | 「Page Not Found」「アカウントが凍結されました」等を検出 |

## 判定アルゴリズム

1. **ブラウザ起動**: Playwright (Chromium) で `stealth` モードを適用し、Facebook へアクセス。タイムアウトは 15 秒を目安。
2. **HTTP レベル判定**  
   - 404/410/451 → `INACTIVE_SUSPECTED`, score=10, signalsに `http_status` を追加して終了。  
   - 429/5xx → `UNKNOWN`, score=0（再実行推奨）。  
   - 200 → DOM 判定へ進む。
3. **DOM 判定**  
   - 事前に `document.documentElement.outerHTML` を保存（監査用）。  
   - 上表のシグナルを抽出。  
   - 以下のスコアリングを適用（合計 100 点満点）:

| 条件 | 重み |
| --- | --- |
| 正常な `og:description` が存在し、固定エラー文言に該当しない | +25 |
| `og:image` が空でなく `safe_image.php` 以外 | +15 |
| CTA ボタン / メッセンジャーリンクが存在 | +15 |
| 投稿タイムスタンプ最新が 90 日以内 | +25（期限経過で 0） |
| `og:url` が対象ページ URL と一致 | +10 |
| HTML に明示的なエラーメッセージが含まれる | -40 |
| `<title>` が固定エラータイトル | -20 |
| リダイレクトURLに `login` 等を含みコンテンツ取得失敗 | -30 |

4. **閾値判定**  
   - score ≥ 60 → `ACTIVE`  
   - score ≤ 20 → `INACTIVE_SUSPECTED`  
   - それ以外 → `UNKNOWN`

5. **監査用ログ**  
   - 取得したシグナル、スコア、レスポンスコード、処理時間を構造化ログに記録。  
   - HTML 全文はコンプライアンスの観点から保存しないが、デバッグ時のみ暗号化ストレージに一定期間保持する（要運用判断）。

## エラーハンドリング

- タイムアウト / ネットワークエラー → `UNKNOWN`, 再試行上限 2 回。  
- reCAPTCHA / ログイン画面表示 → `UNKNOWN`, signals に `requires_login` を含める。  
- HTML 解析で例外が出た場合も `UNKNOWN` とし、ログでスタックトレースを残す。

## セキュリティ・法的考慮

- 自動スクレイピングは Facebook 利用規約違反となる可能性がある。運用前に法務チェックを実施し、アクセス頻度を制限（例: 1 ドメイン/日 1 回・連続アクセス間隔最小 30 秒）。  
- 認証が必要な情報にはアクセスしない。ログインセッションやユーザー Cookie を利用する実装は禁止。  
- 取得したデータは営業施策の参考ステータスとしてのみ利用し、ページコンテンツを二次利用しない。

## インターフェース設計

```json
{
  "status": "ACTIVE",
  "score": 75,
  "signals": {
    "http_status": 200,
    "title": "Company Name - Home | Facebook",
    "meta_description": "Company Name. Contact us...",
    "og_image_present": true,
    "cta_detected": true,
    "latest_post_unixtime": 1761800000,
    "error_banner_detected": false,
    "requires_login": false
  },
  "observed_at": "2025-10-31T02:30:00Z"
}
```

## 開発タスク（T-013）

1. **スクレイピング基盤準備**  
   - Playwright ランナーの共通ユーティリティ作成  
   - タイムアウト・リトライ制御の実装
2. **シグナル抽出モジュール**  
   - DOM 解析スクリプトの実装（TypeScript 想定）  
   - 固定エラーメッセージ辞書の整備（多言語対応）
3. **スコアリング実装 & 単体テスト**  
   - スコア計算関数  
   - パターン別の期待値テスト
4. **API / バッチ統合**  
   - 企業データ収集パイプラインから呼び出し  
   - 結果を `facebook_page_status` などのフィールドへ保存
5. **監査ログ & アラート**  
   - スコア低下時の通知条件整理  
   - 再試行上限の計測ダッシュボード
6. **運用ガイド作成**  
   - スクレイピング頻度・法務チェックリスト  
   - 再調査が必要なケースのハンドリング

## 今後の課題・オープンクエスチョン

- Facebook の DOM 構造は頻繁に変化するため、定期的なメンテナンスコストが発生する。監視方法（例: 主要シグナルの取得率モニタリング）を別途検討する。  
- ページの「休止」状態（投稿は無いがプロフィール自体は有効）をどう扱うか要議論。必要であればステータスを 4 区分化する。  
- 取得した HTML に個人情報が含まれる可能性があるため、ログのマスキングポリシーを策定する。  
- 規約面で問題があると判断された場合の代替策（外部サービス連携・人的チェックフロー）を検討する。
