# CLAUDE.md

## このプロジェクトについて
news-pigeon: 複数ソース(RSS/API/HTML/SPA)からトレンド記事をスクレイピングし、トピックごとにDiscordへ整形済みEmbedとして通知するWebスクレイピング基盤。

## 技術スタック
- 言語: TypeScript(ESM, target ES2022)、実行はtsx経由のNode.js
- 主要依存: cheerio(静的HTML解析)、fast-xml-parser(RSS/Atom解析)、playwright(SPA/Cloudflare対策スクレイピング、stealthモード)、dotenv(環境変数)
- テスト: Vitest(vitest.config.tsでglobals有効、environment: node)
- パスエイリアス: `@/*` → `src/*`(tsconfig.json / vitest.config.tsで設定)

## コマンド
- テスト実行: `npm test`(vitest run)— 確認済み
- テスト監視: `npm run test:watch`— 確認済み
- 特定トピックのスクレイプ: `npm run scrape -- --topic <id>`(tsx src/commands/scrape.ts)— 確認済み
- 通知送信: `npm run notify -- --topic <id>`(tsx src/commands/notify.ts)— 確認済み
- クリーンアップ: `npm run cleanup`(tsx src/commands/cleanup.ts)— 確認済み
- build: package.jsonにbuildスクリプトなし — 未確認
- README記載の`npm run start`は package.json に存在せず — 未確認(古い記述の可能性、下記「規約・注意」参照)

## 構造
- `src/commands/`: scrape/notify/cleanupの各エントリポイントと引数処理(args.ts)
- `src/scrapers/`: rss/api/html/playwright各スクレイパー実装
- `src/pipeline/`, `src/discord/`, `src/output/`, `src/types/`, `src/utils/`: 実行制御・Discord送信・出力読み書き・型定義・共通処理
- `topics/`: トピック別設定(スクレイプ対象ソースとDiscord出力設定、TopicConfig型に準拠)
- `tests/`: Vitestテスト(scrapers/pipeline/discord)とfixtures
- `output/`: スクレイプ結果JSON(生成物、.gitignore対象)

## 規約・注意
- Discord webhookは`.env`の`DISCORD_WEBHOOK_<TOPIC_ID>`(トピックIDのハイフン→アンダースコア、大文字化)で設定。`.env`自体は.gitignore対象で直接編集・コミット禁止(このセッションでも内容は開いていない)
- `output/`配下(例: tech.json)はスクレイプ結果の生成物で.gitignore対象。手動編集・コミット対象にしない
- 新規トピック追加は`topics/<topic-id>.ts`を作成しTopicConfig型に従う(README記載の雛形あり)
- README(README.md)には`npm run start`、`src/index.ts`、`subscriptions.json`への言及があるが、現在のsrc構成・package.jsonには存在しない。git履歴上`subscriptions.json`は「モノリシックなpipelineをscrape/notify/cleanupに分割」するリファクタ時点で姿を消しており、READMEが古いままである可能性が高い。README内容を鵜呑みにせず、実装(src/commands, package.json)を正とする
- 本プロジェクトはClaude Code Routines(クラウド環境)での定期実行を想定した設計(README記載)

## 現在の状態
- 直近のコミットはPlaywrightのクラウド実行対応が中心: `--no-sandbox`フラグ追加、MediumスクレイパーをPlaywrightからRSSへ切替(クラウド互換性のため)、JSON出力時のタイトル内二重引用符サニタイズ修正
- 推測: モノリシックなpipelineをscrape/notify/cleanupの3コマンドに分割するリファクタ(8bcb210)の後、nikkei/world-economy/japan-news/world-newsなど複数の日本語・経済系ニューストピックが追加され(e64e40f)、economyトピックはnikkeiへ統合(7067e0f)。現在はクラウド(サンドボックス制約のある環境)での安定動作に向けた細かい修正が継続中の段階
