# phone-privacy-audit

iPhone個人情報保護監査アプリ。

自分の端末とアカウント群を定期的に棚卸しするための、単一HTMLファイル（+ `support.js`）のセルフホスト型ツール。
想定しているのは、MullvadVPNを自ら選んで使うような、すでに知識と警戒心を持っている人。
啓蒙や説得のトーンではなく、業務監査ツールに近い無機質さを狙っている。

- `index.html` — アプリ本体（テンプレート＋ロジック＋内蔵データ）
- `support.js` — Claude Design ランタイム（生成物。編集しない）
- `settings-data.json` — 配信データ。**新着項目はこのファイルを書き換えて push するだけで反映される**

## 4本の柱

### 1. 設定状況確認

LINE・Google/Gemini・iOS 26・Instagram/Facebook・X・TikTok・Amazon/Alexa・ChatGPT・GitHub・Discord・WhatsApp・Yahoo! JAPAN・PayPay・VPN の
**14サービス・42項目**のプライバシー設定チェックリスト。

各項目に「設定パス」「なぜ重要か」「既定値はどうなっているか」を明記している。
X（Grok）へのデータ共有のように、**リロード後に設定が勝手にオンへ戻る既知の不具合**があるものは、項目ごとに注記フラグを立てている。

保護レベルで表示する重要度を絞り込める。

| レベル | 表示される重要度 |
| --- | --- |
| 基本 | 要対応（high）のみ |
| 標準 | 要対応 + 推奨（high / medium） |
| 徹底 | すべて（high / medium / low） |

### 2. 連携アカウント台帳

パスワードは一切扱わない。「どのサービスに、どのアカウントでログインしているか」という **SSOの関係性だけ** を記録する台帳。

「LINEとYahoo!の連携解除は済んでいるか」「GitHubアカウントをWordPressに使っているか」といった結びつきを可視化する。

### 3. ネットワーク診断

- IPv4 / IPv6 アドレスと接続情報（ISP・おおよその所在地）
- DNSサーバー
- 出口ノード（Cloudflare trace）
- WebRTC IPリーク検査
- タイムゾーン整合性
- トラッキング防止シグナル（DNT / GPC）
- 位置情報の許可（実際のブラウザ Permissions API）
- トラッキング許可（ATT相当・プレースホルダー）
- VPN接続状態（推定）

**診断は自動実行しない。** 明示的な「診断を開始する」操作を挟む。
WebRTC検査がOSの「ローカルネットワークへのアクセス」許可ダイアログを誘発しうるため、
ユーザーの能動的な操作の直後にしか出さない設計にしている。

### 4. 通知設定（プレビュー）

新着項目通知・定期リマインドのトグルと間隔設定（7 / 14 / 30日）。
ネイティブ版では `UNUserNotificationCenter` で実装する。Web版ではブラウザ通知APIがそのまま本番の代替手段になる。

ウィジェットプレビューはiOS版限定の将来構想で、未対応件数とVPN接続状態を表示するデザイン案。

## 技術方針

- **クライアントサイドのみで完結し、外部送信はしない。** 診断のためのAPI問い合わせ自体は発生するが（ipwho.is / ipify / Cloudflare trace 等）、結果はローカル処理のみ。チェック状況・台帳・設定はすべて `localStorage`（`pa_*`）に入る。
- **自動で設定を変更しない。** 気づかせるだけで、変更は本人の手で行わせる（Jumbo Privacyの教訓）。
- 配信元URL（`REMOTE_URL` 定数）を書き換えて git push するだけで、全ユーザーに新着項目が自動反映される。手動貼り付けは廃止した。

## ホスティング

静的ファイルを置くだけで動く。`main` への push で `.github/workflows/pages.yml` が走り、`settings-data.json` を検査したうえで GitHub Pages へ配信する。

**初回だけ手動の設定が要る。** Settings → Pages → Source を「GitHub Actions」にする。Actions のトークンには Pages サイトを新規作成する権限が無く、ワークフロー側からは有効化できない（`Resource not accessible by integration` になる）。なお **private リポジトリの Pages は GitHub Pro 以上が必要** で、無料プランならリポジトリを public にする。それまでは `deploy` ジョブだけが失敗し、`validate` ジョブは通常どおり通る。

他のホスティングに置く場合も、`index.html` / `support.js` / `settings-data.json` を同じディレクトリに並べるだけでよい（`.nojekyll` 同梱）。

`file://` で直接開くと `settings-data.json` の取得がブラウザにブロックされるが、その場合は内蔵データのまま静かに動作する。動作確認はHTTP経由で行うこと。

```
python3 -m http.server 8000
# → http://localhost:8000/
```

## 項目を追加・更新する

1. `settings-data.json` を編集する
2. `version` を必ず上げる（例：`2026-07-29c` → `2026-08-05a`）
3. commit して push

次回起動時、`version` の差分を検知して各端末のデータが差し替わり、増えた項目は「新着」タブに `NEW` として並ぶ。
既存項目の `id` は変えないこと。`id` がチェック状況（`pa_done` / `pa_seen`）の紐付けキーになっている。

項目のスキーマ:

```jsonc
{
  "id": "line-talkroom-info",        // 一意。後から変更しない
  "title": "トークルーム情報の提供をオフ",
  "path": "設定 → プライバシー管理 → 情報の提供 → トークルーム情報",
  "severity": "high",                // high | medium | low
  "why": "なぜ重要か・既定値はどうなっているか",
  "watch": "既知の不具合など（任意）",
  "verifiedAt": "2026-07-29"         // この内容を実機で確認した日
}
```

`index.html` 内の `APP_DATA` は、配信元に到達できない場合のフォールバックとして残している。大きな改訂のときは両方を揃えておくとよい。

## 既知の制約

- **iOSはサードパーティによるインストール済みアプリ一覧の自動取得を許可しない。** そのため初回に、使っているサービスを手動でチェックリストから選ばせる方式にしている。
- **`support.js`（Claude Design ランタイム）は `new Function()` と外部CDN（unpkg）への動的依存を持つ。** 厳格なCSP環境（Claudeのファイルプレビュー等）では正しく動作しない。実機・実際にホストした環境での動作を前提とする。
- ネットワーク診断のうち **「トラッキング許可」「VPN接続状態の確定判定」「ウィジェット」はiOSネイティブAPI（`ATTrackingManager` / `NWPathMonitor` / `WidgetKit`）が前提** で、Web版に代替手段はない。現状はプレースホルダーおよびデザイン案として、その旨をUI上に明示したまま置いている。
- DNSリーク検査は dnsleaktest.com のAPIに依存しており、CORSの都合で取得できないことがある。確実に見たい場合はブラウザで dnsleaktest.com を直接開く（チェックリスト側にも項目として入れてある）。
- ブラウザからは本来の traceroute を実行できないため、「出口ノード」は到達しているネットワーク結節点を代替指標として表示している。
