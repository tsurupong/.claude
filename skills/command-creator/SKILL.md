---
name: command-creator
description: Claude Code のカスタムスラッシュコマンド(カスタムコマンド/スキル)を作成・修正するスキル。ユーザーが「/xxx コマンドを作りたい」「スラッシュコマンドを追加したい」「.claude/commands を書きたい」「定型作業をコマンド化したい」「SKILL.md の frontmatter がわからない」と言った時、または引数($ARGUMENTS)・動的コンテキスト(!`cmd`)・allowed-tools・context:fork の設定を頼まれた時に必ず使う。
argument-hint: [作りたいコマンドの説明]
---

# カスタムコマンド作成

Claude Code のカスタムコマンドを作る。**現行仕様ではカスタムコマンドはスキルに統合されている**: `.claude/commands/deploy.md` と `.claude/skills/deploy/SKILL.md` はどちらも `/deploy` を作り、同じ frontmatter が使える。公式推奨はスキル形式(補助ファイル・自動起動・起動制御が使えるため)。新規作成は原則スキル形式で行い、ユーザーが `.claude/commands/` を明示した時だけ単一ファイル形式にする。

## 手順

1. **要件を確認する**(会話から読み取れれば質問しない):
   - 何をするコマンドか / 引数は必要か
   - 起動主体: ユーザーだけ(`/name` 手動)か、Claude にも自動起動させるか
   - 適用範囲: 個人(全プロジェクト)か、このプロジェクトだけか
   - 副作用(デプロイ・送信・削除など)があるか → あるなら `disable-model-invocation: true` を必ず付ける
2. **置き場所を決める**:
   | 範囲 | パス | コマンド名の由来 |
   |---|---|---|
   | 個人 | `~/.claude/skills/<name>/SKILL.md` | ディレクトリ名 |
   | プロジェクト | `.claude/skills/<name>/SKILL.md` | ディレクトリ名 |
   | レガシー形式 | `.claude/commands/<name>.md` | 拡張子なしファイル名 |

   同名衝突時は enterprise > personal > project > バンドルスキルの優先順。既存スキル一覧を確認して名前の衝突を避ける。
3. **SKILL.md を書く**(下のテンプレートと設計原則に従う)。frontmatter の全フィールド・引数置換・動的コンテキストの詳細仕様は [references/spec.md](references/spec.md) を読む。基本フィールド以外(model, effort, hooks, paths, shell, disallowed-tools 等)を使う場合は必ず spec.md を参照してから書く。
4. **動作確認する**。スキルディレクトリはライブ検出されるので再起動不要(ただし session 開始時に `~/.claude/skills/` 等の親ディレクトリが存在しなかった場合は再起動が必要)。ユーザーに `/name 引数` で試すよう案内し、`` !`cmd` `` を使った場合はコマンド単体が動くことを事前に自分で検証する。

## 最小テンプレート

```yaml
---
description: 何をするか + いつ使うか(Claude の自動起動判断はこの文が全て)
argument-hint: "[issue-number]"
disable-model-invocation: true   # 手動専用にする場合のみ
---

GitHub issue $ARGUMENTS を修正する:
1. issue を読む
2. 修正を実装する
3. テストを書く
```

## 設計原則

- **description が起動精度を決める**。「何をするか」+「どんな依頼の時に使うか」を具体語で書く。自動起動させたいなら、ユーザーが実際に言いそうなフレーズを含める。手動専用(`disable-model-invocation: true`)なら description は listing に載らないため簡潔でよい。
- **本文は簡潔に**。起動後は本文がセッション中ずっとコンテキストに残るため、1行ごとにトークンコストがかかる。500行以内、長い参照資料は別ファイルに分離して SKILL.md からリンクする。
- **引数**: `$ARGUMENTS`(全引数)、`$0` `$1`(位置引数、0始まり)、frontmatter `arguments: [issue, branch]` で `$issue` のような名前付きも可。複数語は引用符で1引数になる。
- **動的コンテキスト**: 行頭または空白直後の `` !`git diff HEAD` `` は、Claude が読む前にシェル実行され出力に置換される(前処理であり Claude が実行するのではない)。複数行は ```` ```! ```` フェンス。ライブデータを前提にするコマンド(diff 要約、PR レビュー等)で使う。
- **権限**: 決まったコマンドを叩くスキルは `allowed-tools: Bash(git add *) Bash(git commit *)` のように付けると、そのターンだけ確認なしで実行できる。同梱スクリプトは `${CLAUDE_SKILL_DIR}` でパス解決し、allowed-tools にも同じ変数を書く。
- **隔離実行**: 会話履歴なしで独立実行させたい重いタスクは `context: fork`(+ `agent: Explore` 等)。ガイドライン系コンテンツに fork を付けると実行すべきタスクがなく空振りするので付けない。
- **危険な操作(デプロイ・push・送信・削除)には必ず `disable-model-invocation: true`**。Claude が勝手に実行してはいけない操作をコマンド化する時の鉄則。
