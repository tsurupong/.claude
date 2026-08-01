# 実行力移植 Skill Pack

Claude Fable 5 の働き方を Opus / Sonnet / 他モデルで再現するための規約・手順・雛形のセット。
2026-07-12 の環境診断セッションで抽出・作成。

## 構成

### 常時適用(自動ロード)
- `~/.claude/CLAUDE.md` — 動作規約(言語・仕事の型・自走境界・品質の掟・スキル運用)
- `~/.claude/rules/00-language.md, 10-safety.md, 20-verification.md`

### スキル(~/.claude/skills/、必要時ロード)
| スキル | 用途 |
|---|---|
| operating-principles | 基本思想(迷った時) |
| task-execution-rules | 行動ルール(複数ステップ作業) |
| prompt-templates | 依頼の定型文 T1〜T7 |
| workflow-patterns | 並列化・委譲・裏取りパターン P1〜P7 |
| quality-checklist | 完了前チェック(全成果物) |
| claude-code-environment-audit | 環境診断の手順 |
| ai-team-design | エージェント編成の設計法 |
| content-workflow / teaching-material-workflow / seminar-event-design-workflow | 制作系 |
| business-strategy-workflow | 事業壁打ち |
| software-development-workflow | 開発標準 |

**スキル自作時の規約**: description には必ず発火条件を「〜の時に使用」の形で含める
(これが合格基準。条件文言のない description は未発火リスクとして修正対象)。

### エージェント(~/.claude/agents/、2026-08-01 改編で14体)
- レビュー3層: review-orchestrator(opus) / review-auditor(opus) /
  security-reviewer・code-reviewer・doc-reviewer(sonnet)
- 不具合調査2段: bug-investigator(sonnet) / rootcause-auditor(opus)
- 実行・出荷・文章: research-explorer(haiku) / impl-planner(opus) /
  plan-auditor(opus) / impl-builder(sonnet) / test-fixer(sonnet) /
  pr-shipper(sonnet) / doc-editor(sonnet)

旧6体編成(explorer/planner/builder/build-fixer/reviewer/editor-jp)は
`~/.claude/agents-archive-2026-08-01/` にアーカイブ済み。詳細は ai-team-design スキル。

### 出力スタイル(~/.claude/output-styles/)
business-sparring / dev-review / env-audit(`/output-style` で切替)

### 本ディレクトリ
- `handoff-prompt-for-other-models.md` — 新モデルのセッション冒頭に貼る規約(CLAUDE.mdと同内容)
- `examples.md` — 模範実行例(随時追記)
- `project-claude-drafts/` — 各プロジェクト用 CLAUDE.md ドラフト(確認後に配備)

## 他モデルへの導入方法
1. この ~/.claude 構成ごと使う(推奨。CLAUDE.md が自動で効く)
2. Claude Code 以外のツールへは handoff-prompt-for-other-models.md を冒頭に貼る
3. 受講者・チーム配布はこの一式を plugin 化(marketplace形式)して配る
