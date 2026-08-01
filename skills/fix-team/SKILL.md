---
name: fix-team
description: 機能修正・バグ修正をチームで実行する手順。「直して」「修正して」「バグを取って」「テストが落ちてる」と頼まれた時、明示的に fix-team と言われなくても修正作業を始める前に必ず使用。メインセッションが司令塔となり、症状に応じて test-fixer / bug-investigator / impl-planner 等の担当を呼び分ける。
---

# 機能修正チーム

メインセッションはオーケストレーターに徹する。作業の型は software-development-workflow
準拠。本スキルが定義するのは「症状ごとに誰を呼び、何を渡し、何を受け取るか」。

## チーム編成

| 担当 | 呼ぶ条件 | 渡すもの → 受け取るもの |
|---|---|---|
| test-fixer | テスト・ビルド・型エラーが赤い(まずここ) | エラー出力 → 最小修正 or 失敗記録(reason+hint+gate) |
| bug-investigator | 原因不明の挙動 / test-fixer 3回失敗後 | 再現手順 → 確度付き原因仮説 |
| rootcause-auditor | 原因を根拠に削除等の高リスク修正をする前(必須) | 調査報告 → CONFIRMED/差し戻し |
| impl-planner → plan-auditor | 修正が3ファイル超 or アーキ判断を含む | 要件 → 検査済み計画書(承認後に実装へ) |
| impl-builder | 承認済み計画の実装 | 計画書 → 実装+検証結果 |
| review-orchestrator | 修正完了後(小差分は code-reviewer 単体) | 差分 → 検査済み指摘 |
| pr-shipper | レビュー通過後の commit/PR | 差分+文脈 → コミット/PR(push は承認待ち) |

## 手順

1. **症状の実測**: 再現手順とエラー出力を実際に取得してから始める(推測で委譲しない)。
2. **分岐**:
   - テスト/ビルドが赤い → test-fixer
   - 原因不明・想定外の挙動 → bug-investigator。高リスク対応前は rootcause-auditor 必須
   - 仕様変更を伴う → impl-planner → plan-auditor → ユーザー承認 → impl-builder
3. **レビュー**: 修正完了後に review-orchestrator(〜2ファイルの軽微な差分は
   code-reviewer 単体)。blocking 指摘は修正してから次へ。
4. **出荷**: 頼まれていれば pr-shipper。push は必ず承認を待つ。
5. **報告**: 原因 → 修正内容 → 検証結果(テスト出力を貼る)の順。

## 常駐往復モード(SendMessage)

レビュー⇔修正が1〜2往復で収まらないと見込まれる時のみ、workflow-patterns P9 に従い
name 付きでメンバーを起動して往復させる:

1. 修正担当(impl-builder または test-fixer)とレビュー担当(code-reviewer)を name 付きで起動
2. レビューの blocking 指摘を SendMessage で修正担当に転送 → 修正報告を受けたら
   レビュー担当に SendMessage で再検査を依頼(文脈を保った差分再レビュー)
3. blocking ゼロで解散。3往復で収束しない場合は停止し、review-auditor に
   指摘自体の妥当性検査を依頼してから状況を報告する

## オーケストレーションの掟

- メインは原則編集しない。例外は1ファイル数行の自明な修正のみ(その場合も検証は必須)。
- 独立した調査は1メッセージで並列起動(workflow-patterns P1)。修正は直列で。
- 同じ失敗が3回続いたら手を止めて前提を疑い、状況を報告する。
- 完了条件は「テストが緑になるのを見た」。quality-checklist を通してから完了宣言。
