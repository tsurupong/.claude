---
name: ai-team-design
description: サブエージェント編成・モデル割当・権限設計の指針。エージェントを新設する時、マルチエージェント化を検討する時に使用。
---

# AIチーム設計

## 設計原則
1. **分離の基準はコンテキスト汚染**: 「大量に読むが結論しか要らない」仕事から切り出す。
2. **権限は役割の最小**: 調査系はread-only固定。書けるのは impl-builder 系のみ。
3. **モデル割当**(2026年時点の目安。モデル名はバージョンごとに見直す):
   - 定型・大量処理 = Haiku / 実装・レビュー = Sonnet / 計画・アーキ判断 = Opus
   - 迷ったら Sonnet。
4. **1エージェント=1つの成果物形式**: 返す形式(表/判定/差分/計画書)を定義に書く。
5. **並列は独立な仕事だけ**: 依存があるなら直列+チェックポイント。
6. **検証者を分ける**: 作った本人に検証させない(レビューは担当レビュアー、その妥当性検査はさらに別の review-auditor)。

## 標準編成(~/.claude/agents/ に配備済み。2026-08-01 改編)

### レビュー3層(担当=sonnet、検査・統括=opus)
| agent | model | 役割 |
|---|---|---|
| review-orchestrator | opus | レビュー司令塔。観点判定→担当を並列起動→auditor検査→統合。自分では指摘しない |
| review-auditor | opus | レビューのレビュー。指摘の妥当性(実読照合)と影響範囲を判定。CONFIRMED/REFUTED/要人間判断 |
| security-reviewer | sonnet | セキュリティ観点の担当(脆弱性+攻撃シナリオ+修正案) |
| code-reviewer | sonnet | 汎用コード品質の担当(重大度順の指摘) |
| doc-reviewer | sonnet | 設計書・企画書の整合性担当(矛盾・欠落・未定義、blocking/advisory 区分) |

### 不具合調査2段(調査=sonnet、判定=opus)
| agent | model | 役割 |
|---|---|---|
| bug-investigator | sonnet | 原因調査。再現→仮説列挙→実読検証→確度付き報告。修正しない。test-fixer 3回失敗後の引き継ぎ先 |
| rootcause-auditor | opus | 原因判定のレビュー。独立2系統で裏取りし CONFIRMED/差し戻し/要人間判断。高リスク対応の前に必須 |

### 実行・出荷・文章
| agent | model | 役割 |
|---|---|---|
| research-explorer | haiku | 読み取り専用の大量調査 |
| impl-planner | opus | 実装計画(コードは書かない。参照設計文書を明示) |
| plan-auditor | opus | 計画のレビュー。実在性・網羅性・検証可能性・可逆性を実コード照合で判定し承認/差し戻し。実装承認の前に使用 |
| impl-builder | sonnet | 承認済み計画の実装 |
| test-fixer | sonnet | テスト・ビルドエラーの最小修正。未解決時は reason+hint+gate の失敗記録を返す |
| pr-shipper | sonnet | テスト確認→コミット→PR作成の定型出荷。push は承認待ち |
| doc-editor | sonnet | 日本語文章の推敲(内容の整合性は doc-reviewer) |

旧編成(explorer/planner/builder/build-fixer/reviewer/editor-jp の6体)は
`~/.claude/agents-archive-2026-08-01/` にアーカイブ済み。

## 新設判断
新しいエージェントを作るのは「同じ委譲を3回した時」。1回目・2回目はその場の指示で足りる。
