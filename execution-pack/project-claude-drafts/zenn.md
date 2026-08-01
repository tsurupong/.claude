# CLAUDE.md

## このプロジェクトについて
Zenn(zenn.dev)に投稿する技術記事の下書きと、執筆に使う調査素材を管理する個人用リポジトリ。現時点では npm/nvm のトラブル事例に関する記事を1本(下書き)管理している。

## 技術スタック
- 言語/フレームワーク: なし。コードのビルド対象ではなく、Markdownベースの執筆用リポジトリ
- package.json / Cargo.toml / pyproject.toml 等のマニフェストは未確認(存在しない)
- zenn-cli 標準構成(`articles/`, `books/`, package.json)は未確認(存在しない)
- 主要ファイル形式: Markdown(記事本文・調査メモ)、PDF(ChatGPT出力の保存)、npm debugログ

## コマンド
- build: 未確認(マニフェストファイルが存在しないため)
- test: 未確認
- run: 未確認
- 記事プレビュー用コマンド(`npx zenn preview` 等)の設定も未確認

## 構造
- `1.202602028/`: 記事ディレクトリ(連番+日付と思われる命名)。記事本体とアセットを格納
- `1.202602028/article.md`: フロントマター(title/emoji/type/topics/published)付きのZenn記事本文
- `1.202602028/assets/nvm_npm_global_update_defact/`: 当該記事のための調査資料(ChatGPT PDF、AIとの会話ログMarkdown、npm debugログ)を格納
- docs/ ディレクトリ: 情報なし(存在しない)

## 規約・注意
- `article.md` の frontmatter `published: false` は公開可否のフラグ。確認なく `true` に変更しない
- `assets/**/_logs/*.log` は npm/nvm 実行時の自動生成デバッグログ(生成物)。手動編集・内容の書き換え対象にしない
- `assets/**/*.pdf` や `explanation-of-*.md`(SpecStory形式のAIツール会話ログ)は執筆の一次資料。参照のみに留め、直接改変しない
- リポジトリ直下は `.git` を含まず(未初期化)、README・ライセンス等の管理ファイルも存在しない

## 現在の状態
- 本リポジトリは Git管理下にないため `git log` は取得不可(情報なし)
- 推測: ディレクトリ名および同梱ログのタイムスタンプ(2026-02-27〜28)から、直近で「`npm update -g` 実行によりnpmが消えた」事象の調査・記事化を進行中
- `article.md` 末尾に「明日続きやる...」との記述があり、推測: 記事は「原因」セクションの途中で執筆が中断している(未完成・未公開状態)
- 推測: 今後、原因説明の追記や新規記事ディレクトリの追加が行われる可能性がある
