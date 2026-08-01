# カスタムコマンド/スキル 詳細仕様

出典: https://code.claude.com/docs/en/skills (2026-08-01 取得)

## 目次
1. [Frontmatter 全フィールド](#frontmatter-全フィールド)
2. [文字列置換](#文字列置換)
3. [動的コンテキスト注入](#動的コンテキスト注入)
4. [起動制御](#起動制御)
5. [サブエージェント実行 (context: fork)](#サブエージェント実行)
6. [配置とコマンド名](#配置とコマンド名)
7. [ライフサイクルと注意点](#ライフサイクルと注意点)

## Frontmatter 全フィールド

すべて省略可。`description` のみ推奨(自動起動判断に使われる)。boolean は `true/false` のほか `yes/no/on/off/1/0` も可(v2.1.218+)。

| フィールド | 説明 |
|---|---|
| `name` | 表示名。既定はディレクトリ名。個人/プロジェクトスキルではコマンド名は変わらず表示のみ。プラグインスキルではコマンド名の最終セグメントになる |
| `description` | 何をするか+いつ使うか。省略時は本文の最初の段落。`when_to_use` と合わせて listing 上 1,536 文字で切られるため主用途を先頭に |
| `when_to_use` | 起動トリガーの追加文脈(トリガーフレーズ・依頼例)。description に連結される |
| `argument-hint` | 補完時に表示する引数ヒント。例: `[issue-number]` |
| `arguments` | 名前付き位置引数の宣言。`arguments: [issue, branch]` → `$issue` `$branch`。スペース区切り文字列か YAML リスト |
| `disable-model-invocation` | `true` で Claude の自動起動を禁止(手動 `/name` 専用)。description は listing に載らなくなる。サブエージェントへのプリロードやスケジュールタスク起動(v2.1.196+)も防ぐ |
| `user-invocable` | `false` で `/` メニューから隠す(Claude 専用の背景知識向け)。Skill ツール経由のアクセスは制限しない |
| `allowed-tools` | 起動したターンだけ確認なしで使えるツール。次のユーザーメッセージでクリア。例: `Bash(git add *) Bash(git commit *)`。スペース/カンマ区切りか YAML リスト |
| `disallowed-tools` | スキル有効中に利用不可にするツール。次のメッセージでクリア |
| `model` | 有効中のモデル上書き(そのターンのみ)。`/model` と同じ値、`inherit` 可 |
| `effort` | 有効中の effort 上書き: `low`〜`max` |
| `context` | `fork` で隔離サブエージェント実行 |
| `agent` | `context: fork` 時のエージェント種別。`Explore` / `Plan` / `general-purpose`(既定)/ `.claude/agents/` のカスタム |
| `background` | fork 時のみ。`false` で結果を同ターンで待つ。既定 `true`(v2.1.218+) |
| `hooks` | このスキルのライフサイクルに限定した hooks |
| `paths` | glob パターン。合致するファイルを扱う時だけ自動起動対象にする |
| `shell` | `` !`cmd` `` の実行シェル。`bash`(既定)/ `powershell` |

## 文字列置換

| 変数 | 説明 |
|---|---|
| `$ARGUMENTS` | 全引数(入力そのまま)。本文に無い場合は末尾に `ARGUMENTS: <値>` が自動付加 |
| `$ARGUMENTS[N]` / `$N` | 0 始まりの位置引数。`$0` が第1引数 |
| `$name` | `arguments` frontmatter で宣言した名前付き引数。未指定なら空文字列に展開 |
| `${CLAUDE_SESSION_ID}` | セッションID |
| `${CLAUDE_EFFORT}` | 現在の effort レベル |
| `${CLAUDE_SKILL_DIR}` | SKILL.md のあるディレクトリ。同梱スクリプト参照に使う。本文と `allowed-tools` の Bash ルールの両方で展開される(v2.1.129+) |
| `${CLAUDE_PROJECT_DIR}` | プロジェクトルート(v2.1.196+) |

- 位置引数はシェル風クォート: `/cmd "hello world" second` → `$0`=`hello world`
- 対応する引数がない `$N` はそのまま残る。名前付きは空になる
- リテラル `$1` などはバックスラッシュでエスケープ: `\$1.00`

同梱スクリプトを確認なしで実行させるパターン:

```yaml
---
name: render-chart
description: CSV からチャートを描画
allowed-tools: Bash(${CLAUDE_SKILL_DIR}/scripts/render.sh *)
---
`${CLAUDE_SKILL_DIR}/scripts/render.sh <csv-file>` を実行する。
```

## 動的コンテキスト注入

`` !`command` `` は Claude が読む**前**に実行され、出力に置換される(前処理。Claude は結果だけを見る)。

- 行頭または空白直後の `!` のみ認識。`KEY=!`cmd`` はリテラルのまま
- 複数行は ```` ```! ```` フェンスブロック
- 置換は元ファイルに対して1回だけ。出力内のプレースホルダは再展開されない
- 設定 `disableSkillShellExecution: true` で無効化可能(管理設定向け)

```yaml
---
name: pr-summary
description: PR の変更を要約
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---
- PR diff: !`gh pr diff`
- コメント: !`gh pr view --comments`
```

## 起動制御

| frontmatter | ユーザー起動 | Claude 起動 | コンテキスト |
|---|---|---|---|
| (既定) | 可 | 可 | description 常駐、本文は起動時 |
| `disable-model-invocation: true` | 可 | 不可 | description 非常駐 |
| `user-invocable: false` | 不可 | 可 | description 常駐 |

権限ルールでも制御可: `Skill(commit)`(完全一致)、`Skill(deploy *)`(引数付きプレフィックス)、deny に `Skill` で全面禁止。settings の `skillOverrides`(`"on"` / `"name-only"` / `"user-invocable-only"` / `"off"`)でも frontmatter を変えずに制御できる。

## サブエージェント実行

`context: fork` でスキル本文がサブエージェントのプロンプトになる。会話履歴は渡らない。既定でバックグラウンド実行(結果は完了時に届く。`background: false` で同ターン待機)。

- 明示的なタスク指示があるスキル専用。ガイドライン集に付けると空振りする
- `agent: Explore` / `Plan` は CLAUDE.md を読み込まず軽量
- バックグラウンド fork の編集はチェックポイント(/rewind)対象外

## 配置とコマンド名

| 場所 | パス | コマンド名 |
|---|---|---|
| 個人 | `~/.claude/skills/<name>/SKILL.md` | ディレクトリ名 |
| プロジェクト | `.claude/skills/<name>/SKILL.md` | ディレクトリ名 |
| ネスト | `apps/web/.claude/skills/deploy/` | 衝突時 `/apps/web:deploy` |
| レガシー | `.claude/commands/deploy.md` | ファイル名(拡張子なし) |
| プラグイン | `<plugin>/skills/<name>/SKILL.md` | `/plugin-name:name` |

- 同名優先順: enterprise > personal > project > バンドル。skill と command が同名なら skill 優先
- プロジェクトスキルは起動ディレクトリからリポジトリルートまでの各親の `.claude/skills/` から読み込まれる。ネストしたものは該当サブディレクトリのファイルを触った時に有効化
- `<skill-name>` エントリはシンボリックリンク可
- ディレクトリ変更はライブ検出(再起動不要)。ただしセッション開始時に存在しなかったトップレベル skills ディレクトリの新規作成は再起動が必要

## ライフサイクルと注意点

- 起動するとレンダリング済み本文が会話に入り、セッション中残り続ける → 本文は簡潔に(500行以内、詳細は補助ファイルへ)
- `allowed-tools` の許可は本文と違い**次のユーザーメッセージでクリア**される
- 同一内容の再起動は重複コピーされない(v2.1.202+)。引数や動的コンテキストが変わると再添付
- 自動コンパクション時は直近起動スキルから合計 25,000 トークン(各 5,000)まで再添付
- frontmatter YAML が壊れていると本文は空メタデータで読み込まれ、`/name` は動くが自動起動しなくなる。`--debug` でパースエラー確認
- 起動しない時: description にユーザーが言いそうなキーワードを足す、`What skills are available?` で存在確認
