# サブエージェント フロントマター仕様

出典: https://code.claude.com/docs/ja/sub-agents (2026-08-01 取得)

## 保存場所と優先度(高い順)

| 場所 | スコープ | 備考 |
|---|---|---|
| 管理設定の `.claude/agents/` | 組織全体 | 管理者がデプロイ |
| `--agents` CLI フラグ(JSON) | そのセッションのみ | ディスクに保存されない |
| `.claude/agents/` | プロジェクト | 作業ディレクトリから上へ再帰スキャン。近い定義が優先 |
| `~/.claude/agents/` | 全プロジェクト | 個人用 |
| プラグインの `agents/` | プラグイン有効時 | `hooks`/`mcpServers`/`permissionMode` は無視される |

- サブフォルダに整理可(再帰スキャン)。識別は `name` フィールドのみ。
- 同一スコープ内の name 重複は片方しか読み込まれない。`/doctor` が検出する。
- ファイル追加・編集は数秒で自動検出(再起動不要)。ただし agents ディレクトリ自体の
  新規作成時と `--disable-slash-commands` 起動時は再起動が必要。

## 全フィールド(必須は name / description のみ)

| フィールド | 説明 |
|---|---|
| `name` | 小文字+ハイフンの一意識別子。hooks に `agent_type` として渡る |
| `description` | Claude が委譲判断に使う。何をする+いつ使うを書く |
| `tools` | 許可リスト。省略で全ツール継承。`Bash(git diff *)` 形式や `mcp__<server>` パターン可。全エントリが解決不能だと起動拒否(v2.1.208+) |
| `disallowedTools` | 拒否リスト。tools と併用時はこちらが先に適用。`mcp__*` で全MCP除外可 |
| `model` | `sonnet` / `opus` / `haiku` / `fable` / 完全ID / `inherit`(既定) |
| `permissionMode` | `default` / `acceptEdits` / `auto` / `dontAsk` / `bypassPermissions` / `plan`。親が bypassPermissions / acceptEdits / auto の場合は親が優先 |
| `maxTurns` | 停止までの最大 agentic ターン数 |
| `skills` | 起動時にスキル**全文**をコンテキストに注入(プリロード)。未指定でも Skill ツール経由の呼び出しは可能 |
| `mcpServers` | このサブエージェント専用の MCP。名前参照(親の接続を共有)またはインライン定義(.mcp.json と同スキーマ)。インラインならメイン会話のコンテキストを消費しない |
| `hooks` | このサブエージェント内のみで発火するフック。`Stop` は `SubagentStop` に変換。PreToolUse + 検証スクリプト(exit 2 でブロック)で「Bash は読み取り専用クエリのみ」等の条件制御が可能 |
| `memory` | 永続メモリ。`user`(~/.claude/agent-memory/<name>/) / `project`(.claude/agent-memory/<name>/、推奨) / `local`。有効時は Read/Write/Edit が自動有効化され、MEMORY.md 先頭200行が注入される |
| `background` | `true` で常にバックグラウンド実行。v2.1.198+ は既定でバックグラウンド |
| `effort` | `low`〜`max`。セッションの努力レベルをオーバーライド |
| `isolation` | `worktree` で一時 git worktree 内で実行(既定ブランチから分岐、無変更なら自動削除) |
| `color` | 表示色: red/blue/green/yellow/purple/orange/pink/cyan |
| `initialPrompt` | `--agent` でメインセッションとして起動された時のみ、最初のユーザーターンとして自動送信 |

## サブエージェントで使えないツール

`AskUserQuestion` / `EnterPlanMode` / `ExitPlanMode`(plan モード時を除く) /
`ScheduleWakeup` / `WaitForMcpServers` — tools に書いても無効。

## モデル解決順序

1. 環境変数 `CLAUDE_CODE_SUBAGENT_MODEL`
2. 呼び出し時の `model` パラメータ
3. フロントマターの `model`
4. メイン会話のモデル(継承)

## コンテキストに載るもの(起動時)

- 本文のシステムプロンプト + 環境情報(完全な Claude Code プロンプトではない)
- 委譲プロンプト(タスクメッセージ)
- CLAUDE.md 階層 + git ステータス(組み込み Explore/Plan はスキップ)
- `skills` でプリロードしたスキル全文
- 会話履歴は渡らない(例外: fork)

## 呼び出し方法

- 自然言語: "Use the <name> subagent to …"(Claude が委譲判断)
- @-mention: `@agent-<name>`(確実にそのエージェントで実行)
- セッション全体: `claude --agent <name>`(本文が Claude Code 既定プロンプトを完全置換)
- 無効化: settings の `permissions.deny` に `Agent(<name>)`

## その他の要点

- ネスト生成可(v2.1.172+、最大深さ5)。禁止するには tools から `Agent` を外す。
- 再開: 完了後もエージェントIDで再開可(SendMessage)。Explore/Plan は再開不可。
- API エラー時(v2.1.199+): 部分出力+失敗メモが返る。再試行か再開を指示する。
- fork は会話全体を継承する特殊型(通常のサブエージェントは新規コンテキスト)。
