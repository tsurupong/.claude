# HALO 優先度高3機能の実装計画

## Context

前段のギャップ分析で特定した「無人稼働の目・耳・記憶」の欠落3点に対応する。
1. **context-recent-failures**: context ポート契約は定義済み・core 配線済み(loop.ts:657 runContext)なのに実装プラグインがゼロ。過去失敗が executor プロンプトに再注入されず、同じ失敗を繰り返す。
2. **on-fail-notify**: エスカレーション(リトライ上限到達)時に人間へ知らせる手段がファイル記録のみ。夜間の「静かな全滅」を防ぐ。通知先はユーザー決定済み: **汎用 Webhook**(`HALO_NOTIFY_URL` へ HTTP POST、Node>=22 の fetch、依存ゼロ)。
3. **halo history + コスト集計**: `halo status`(status.ts:137 aggregateRuns)は集計済みだがコスト合計が無く、時系列一覧コマンドも無い。集計源は `.halo/logs/iter_N.json`(sink-progress-log の jsonl ではない — 当初想定の訂正)。

core のポート契約(ports.ts)変更なし → スキーマ再生成不要、全て追加的(MINOR)。

## 実装タスク(TDD: 各項目テスト先行)

### Feature 1: context-recent-failures(port: context, order: 50)

- **on-fail-record 拡張**: MD 追記に加え構造化 JSONL `.halo/failure-catalog.jsonl`(env `HALO_CATALOG_JSONL`)へ `{ts, task_id, gate, retry_count, reason}` を1行追記。MD は人間用に維持。
  - 変更: `packages/plugins/src/on-fail-record/main.ts` + `on-fail-record.test.ts`
- **新プラグイン**: stdin の TaskSourceOut から task_id を取得 → JSONL を同一 task_id でフィルタ → 直近 N 件(`HALO_RECENT_FAILURES_MAX` 既定5)→ `ContextOut {fragments:[{source:'recent-failures', content, priority:50}]}` を stdout。ファイル無し/該当無し → `{fragments:[]}` exit 0。不正行はスキップ。
  - 新規: `packages/plugins/src/context-recent-failures/{main.ts, context-recent-failures.test.ts}`(lib/io.ts の readStdinJson/writeStdoutJson/diag 使用、spawnSync dist テスト方式)
  - 新規: `plugins/context-recent-failures/plugin.json`(entry `../../packages/plugins/dist/context-recent-failures/main.js`)+ `contract.fixtures.json`(stdin: task-source.out.json / stdout: context.out.json スキーマ参照。context ポート初のプラグインなので fixture は既存の task-source-local を雛形に)
  - `packages/plugins/src/registry.ts` BUNDLED_PLUGINS へ追加(registry.test.ts が drift 検査)

### Feature 2: on-fail-notify(port: on-fail, order: 30 — record=10, requeue=20 の後)

- OnFailIn を読み、`HALO_NOTIFY_URL` 未設定 or `retry_count < HALO_NOTIFY_THRESHOLD`(既定3、task-source-local の `>=` 判定と整合)なら黙って exit 0。条件成立時のみ JSON `{task_id, reason, retry_count, gate, ts}` を POST(AbortController 10s、`HALO_NOTIFY_TIMEOUT_MS`)。全エラーは diag→stderr、常に exit 0(best-effort)。stdout 空。
- テストはモック不使用: テスト内で `node:http` サーバを port 0 で立て URL を env 注入し受信 body を検証。閾値未満→リクエスト無し / 到達不能 URL→exit 0 + stderr の各ケース。
  - 新規: `packages/plugins/src/on-fail-notify/{main.ts, on-fail-notify.test.ts}`、`plugins/on-fail-notify/{plugin.json, contract.fixtures.json}`(stdin: on-fail.in、出力無し — on-fail-record の fixture 形式を踏襲)、registry 追加(HALO_NOTIFY_URL はユーザー設定値なので env テンプレート化しない)

### Feature 3: halo history + status コスト集計

- `packages/cli/src/commands/status.ts`: `RunSummary` に `cost: {input_tokens, output_tokens, usd}` を追加、`aggregateRuns` で `entry.executor?.cost` を合算(欠損は0扱い — 旧ログ耐性)。人間向け出力に1行追加、`--json` は summary 経由で自動反映。`classifyFailure` を export。
  - テスト: `status-aggregation.test.ts` 拡張
- 新規 `packages/cli/src/commands/history.ts` + `history.test.ts`: statusCommand と同じ deps 形(`fs`, `now`)。`loadRuns` 再利用、`--days`(既定7)フィルタ、started_at 昇順、`--limit`(既定20、末尾から)。行: `iter | started_at | outcome | task_id | retry | gate/分類 | usd`。`--json` で配列出力。
- `packages/cli/src/index.ts`: switch(~:98-138)と usage(~:43-53)へ `history` 登録。

## 検証(product/ 直下)

1. `pnpm build`(プラグインテストは dist 必須 — 先にビルド)
2. `pnpm test`(registry drift・新規テスト含む、現行533件+新規が green であること)
3. `pnpm test:contract`(新 fixtures のスキーマ適合)
4. 手動スモーク: tmp dir に catalog.jsonl を置き `echo '<TaskSourceOut json>' | node dist/context-recent-failures/main.js` で fragments 出力確認、on-fail-notify は手元 http サーバで受信確認

## リスク・注意

- registry.ts のエントリパス表現は plugin.json(リポ相対)と registry(dist ルート相対)で異なる — 既存ペアをそのまま踏襲。
- 契約変更なし。gate-loop-audit の自己改変禁止対象(CLAUDE.md/PROMPT.md/.harness.yml/テスト)には触れない(テスト追加は人間主導の通常開発なので対象外)。
- 有効化は別途 `halo enable context-recent-failures` / `halo enable on-fail-notify`(ユーザー環境の .halo/ports/ への書き込みなので実装完了後に確認して実施)。
