---
name: software-development-workflow
description: 機能開発・改修の標準フロー。コードを書く全タスクで使用。
---

# 開発標準フロー

1. **調査**: 既存の実装・ユーティリティを先に探す(車輪の再発明チェック)。
   影響範囲は推測ではなくコードを読んで特定。大量読み込みは research-explorer に委譲。
2. **計画→承認**: 3ファイル超の変更・アーキ判断・不可逆操作を含むなら計画を提示して承認を待つ
   (impl-planner で計画→plan-auditor で検査 or Plan Mode)。単発修正はスキップ可。
3. **テスト先行**: バグ修正は再現テスト→失敗確認→修正→成功確認。
   新機能はテストを先に書く(superpowers:test-driven-development があれば従う)。
4. **小さく実装**: 1ステップごとに検証。既存スタイル・命名に合わせる。
   ついで改善はしない(気づいたら報告のみ)。
5. **レビュー**: review-orchestrator(多観点)か code-reviewer 単体 or /code-review。
   CRITICAL/HIGH は修正してから次へ。
6. **自己検証**: テストだけでなく実際に動かして確認(アプリなら起動、CLIなら実行)。
7. **コミット**: conventional commits(type は英語・本文説明は日本語)。
   push・デプロイ・依存追加は承認制。

## ビルドが赤い時
test-fixer エージェントに委譲(最小差分で緑にする。設計変更はしない)。
3回失敗したら bug-investigator に引き継ぎ、原因確定は rootcause-auditor を通す。
