# HALO v0.3.0 監査欠陥 修正計画

## Context

2026-07-20 の設計 vs 実装監査（実コードで裏取り済み）で、コアの「純粋部分」は健全だが、
**プラグインとコアをつなぐ配線層に、設計が要求する安全・運用機能が到達していない欠落が集中**
していることが判明した。特に selfhost（local queue）を実運用で壊す欠陥（`op=fail` 未送出による
エスカレーション死）を含む。本計画は監査で挙げた全指摘（Critical/High → Security → Medium → Low）を
優先度順に修正する。詳細な監査根拠は `memory/halo-audit-2026-07-20.md` を参照。

対象コード: `product/packages/{core,cli,contracts}/src`, `product/packages/plugins/src`,
文書 `product/docs/design/{d1,d5,02}`, `product/plugins/*/plugin.json`。
自己改変禁止(ADR-0004)は無人ループの制約であり、本作業(人間駆動の開発)には適用されない。
各修正は「小さく編集→`pnpm test`／該当テスト確認」で進める。

---

## Tier 1: 実運用を壊す（最優先）

### C1. コアの失敗経路で `op=fail` を発火（エスカレーション復活）
- **問題**: `loop.ts` は next/complete しか送らず、3回失敗→needs-human 遮断がデッドコード。
- **変更**:
  - `core/src/loop.ts`: `runTaskSourceComplete` に倣い `runTaskSourceFail(deps, taskId, reason, retryCount)` を追加（best-effort、try/catch でループを止めない）。失敗2分岐（executor 非done: loop.ts:461-491 / gate fail: loop.ts:530-561）の `runBestEffort(onFail)` 直後に呼ぶ。
  - 送出ペイロードは既存 `OnFailIn` ではなく task-source 契約の `{op:'fail', task_id, reason, retry_count}`（`contracts/src/ports.ts` の `TaskSourceFail` 型）。
- **retry_count の永続化**: run 跨ぎで `retryCount` が in-memory Map(loop.ts:372)でリセットされる問題は、task-source 側を真実源とする現行設計に合わせ、**送出する `retry_count` はそのまま渡し、閾値判定・needs-human 遷移は各 task-source 実装(既存)に委ねる**（local:97-100 / github:86-88 が発火するようになる）。
- **検証**: `loop.regression.test.ts` に「gate 3回 fail 後に task-source へ op=fail が retry_count>=3 で届く」ケース追加。local 手動E2E: queue に必ず fail するタスクを置き `halo run` → `needs-human/` へ移動し `failures.log` に記録されることを確認。

### C3. runtime setup を worktree ライフサイクルへ配線
- **問題**: `LOOP_PORT_ORDER`(run-wiring.ts:43) に runtime が無く setup 未実行 → 依存ありリポジトリで gate 常時 fail。
- **変更（最小案）**: `cli/src/core-ext/run-wiring.ts` の `createWorktree` シーム内（`git worktree add` 成功後）で、採用 runtime の setup entry を `runPort` 実行する。runtime 発見は `discoverPort({port:'runtime'})` を追加し、kind 解決結果（C5）から採用 runtime を選ぶ。当面は既定 runtime（node-pnpm）の setup を worktree 作成直後に走らせる。
- **代替（暫定）**: gate.d に `05-setup` を追加し gate 前に依存実体化（`delegate('05-setup','setup')`）。恒久解は createWorktree 内実行。
- **検証**: 依存のある fixture リポジトリで worktree 作成→`node_modules` が実体化→gate typecheck が pass することを回帰テストで固定。

### C5. kind / `.harness.yml` 必須チェックを run 経路へ配線
- **問題**: `resolveKind`(config.ts:258)・`findHarnessYml`(discovery.ts:349)・`validateHarnessYml`(config.ts:213) が実装済みだが未呼び出し。
- **変更**: `cli/src/commands/run.ts` または `run-wiring.ts` の loop 準備段で:
  1. `findHarnessYml(cwd)` → 無ければ**タスクを実行せず** needs-human 経路（要件§4.2③）。少なくとも run を明示エラー(exit 3)で止める。
  2. `validateHarnessYml` → `resolveKind(task.kind)` で runtime 群・prompt テンプレートを解決し、C3 の runtime 選択と prompt 構築（`buildPrompt` 前）に供給。
- **検証**: `.harness.yml` 無しリポジトリで run が needs-human/エラー終了するテスト、kind:docs/code で runtime が切り替わるテスト。

### C2. loop-audit を `base...HEAD` 基準に（commit バイパス封じ）
- **問題**: `gate-loop-audit/main.ts:46-48` が `git diff HEAD` のみ → executor が commit すると全チェック素通り＋完了扱い(resolvePrUrl run-wiring.ts:341)。
- **変更**:
  - worktree 作成時の base SHA を gate に渡す。`run-wiring.ts` は既に `worktreeBase` を保持(:325)。`gate.in` に `base` を追加するか、`HALO_GATE_BASE` env で注入。
  - `gate-loop-audit/main.ts` の numstat/name-status/diff を `git diff <base>`（未コミット差分を含めるなら `git diff <base>` は作業ツリー込みで base 比較になる）へ変更。`contracts` の `GateIn` に `base?` を追加。
- **検証**: gate-loop-audit.test.ts に「executor が commit したケース」を追加し、protected file を commit しても fail することを確認。

---

## Tier 2: セキュリティ配線

### S1. executor 子への env 最小化（GH_TOKEN 露出遮断） + PATH 洗浄
- **変更**: `run-wiring.ts` の `makeRunner`(:246) で、**executor ポートに限り** allowlist した env のみ渡す（`HALO_*`, `PATH`, `HOME`, `CLAUDE_*` 等の最小集合）。`GH_TOKEN`/`GITHUB_TOKEN` 等は executor へ渡さない。task-source-github には従来通り必要 env を渡す。
- **PATH 洗浄(要件§6.1)**: `baseEnv`(run-wiring.ts:79) で PATH を `/usr/local/bin:/usr/bin:/bin` + HALO runtime パスへ再構築し `/mnt/c` 由来を除去（WSL2）。`runPort.ts` の誤コメント（「洗浄済み env」）も実態に合わせ修正。
- **検証**: executor 起動時の env に GH_TOKEN が無いことをテスト。

### S2. 対象リポジトリの `.claude/settings.json` を無効化
- **変更**: `executor-claude/main.ts` の claude 起動引数に `--setting-sources` 制限（プロジェクト設定を読ませない）を追加。ADR-0019 の `--settings` 注入が入ればそちらへ統合。loop-audit の保護対象(main.ts:31-34)にも `.claude/settings.json` を追加（防御多層）。
- **検証**: worktree 内に hooks 入り `.claude/settings.json` を置いても executor が読まないことを確認。

### S3. コスト予算を実効化（DAILY_MAX_COST_USD fail-open 解消）
- **変更**: `executor-claude/main.ts` で `claude` を `--output-format json` 起動に切替え、返却 JSON から cost（`total_cost_usd` 等）を抽出し `emit` の `cost.usd_estimate` に載せる。`loop.ts:710` の `toExecutorRecord` は既に `usd_estimate`/`usd` を拾うので受け側は無変更。
- **注**: ADR-0021 の `max_budget_usd`（実行時ハード停止）は別途（Low 参照）。まず日次コスト集計を機能させる。
- **検証**: executor 出力に cost が乗り、`budget.aggregateDailyUsage` が加算されることをテスト。

---

## Tier 3: Medium

### C4. task-source-github の堅牢化
- **変更**（`task-source-github/main.ts`）:
  - `gh()`(:19) で `issue list` の非0終了を検査し die(exit2)→コアの `TASK_SOURCE_ERROR` に乗せる。`JSON.parse` 失敗も `[]` フォールバックせずエラー。
  - fail 分岐(:84-88) で閾値未満は `in-progress→ready` へ戻し再試行可能に。または next で自インスタンス取得済み in-progress も再払い出し対象に。
  - ラベル付替(:55)の exit code を検査（M8）。
- **検証**: gh 失敗モック・fail後 ready 復帰のテスト追加。

### M4/autonomy L1 意味の統一
- **変更**: 方針決定が必要（下記「未決」）。決定後、要件v1.8/D1§1.5/D2§2.5 と ADR-0006/0016 のいずれかへ統一改訂。ADR-0016 の status を accepted へ。

### M5. flock にプロジェクトスコープ付与
- **変更**: `core/src/lock.ts` の `defaultLockPath` に cwd（リポジトリパス）のハッシュを加える。ADR-0009 のプロジェクト間非干渉を満たす。
- **検証**: 別リポジトリ同名プロファイルでロック衝突しないテスト。

### M6. `plugin.json` env の `${...}` 解決
- **変更**: `run-wiring.ts:246` の env 構築で `${VAR}` を `process.env` から展開するヘルパを追加。未定義は空 or エラー。D1§2 の例と挙動を一致させる。

### M7. loop-audit の git diff エラー検査
- **変更**: `gate-loop-audit/main.ts` の各 `run('git', …)` の code を検査し、非0なら fail（安全側）。空 stdout で素通りさせない。

### M9. gate fail 時の worktree 保持（設計判断）
- **変更**: 中間 fail(retry<閾値)で worktree を保持し再実行する ADR-0002 挙動に寄せるか、フレッシュコンテキスト原則を優先し現状維持とするかを決定（下記「未決」）。保持する場合 loop.ts:563 の finally を条件分岐化。

---

## Tier 4: Low / 文書

- **task_id 検証**: `run-wiring.ts:147` の worktree/branch 構築前に `task_id` を `[A-Za-z0-9._-]` 等でバリデート。不正なら安全にスキップ（run 全体クラッシュを防ぐ）。`createWorktree` 呼び出し(loop.ts:434)を try で囲むことも検討。
- **HALO_FAIL_THRESHOLD NaN**: `Number()` 結果が NaN なら既定3へフォールバック（task-source 両実装 + loop）。
- **`halo --version`**: `HALO_CORE_VERSION` を package.json 版へ同期（ビルド時注入 or 参照）。
- **argv プロンプト上限**: 大きい prompt を `claude -p` へ argv 渡しすると ARG_MAX(約128KiB)超過。stdin 渡し（`claude -p` の stdin 経路）へ変更を検討。
- **signal=timeout 分類**: `executor-claude/main.ts:64` で SIGKILL(timeout)と他シグナル(crash)を区別し、crash は timeout でなく stuck/error 側へ。
- **ajv stdin 検証(D11§2)**: 未実装。導入は別タスクとして切り出し（本計画では文書化のみ、実装は要判断）。
- **クレジット probe(要件§4.4)**: 重量プリフライトにシーム追加（Phase 4 寄り、優先度最低）。
- **`--max-budget-usd`(ADR-0021)**: `contracts` の `budget` に `max_budget_usd` 追加 + `run.ts` にフラグ配線 + executor での実行時停止。d1-contract-spec の「MINOR で追加済み」記述と実体を一致させる。
- **文書整合**:
  - **C6**: `d1-contract-spec.md §2`(:607,638) を entry/aux 契約へ改訂（exec 廃止・MAJOR 明記）。`d5-plugin-dev-guide.md`（sh launcher/必須4フィールド）と `d2` §3 の exec 記述、`registry.ts` ヘッダコメントを追随。
  - `design/02` の worktree 置き場記述（~/halo/wt vs $TMPDIR、.sh 三点セット）を現状へ更新。

---

## 未決（実装前に方針決定が必要）

1. **autonomy L1 の意味**（M4）: 「L1=報告のみ（コミットもしない）」へ戻すか、「L1=外部公開なし（ローカルコミット可）」を正式化して上位文書を改訂するか。安全境界の定義なので要合意。
2. **gate fail 時の worktree**（M9）: 保持して再実行 vs フレッシュ破棄。フレッシュコンテキスト原則との整合判断。
3. **ajv 契約検証・クレジット probe・max_budget_usd**: v0.4 スコープに入れるか、本修正から切り離すか。

---

## 全体検証

- 単体/回帰: `product/` で `pnpm test`（現行 522 件 green を維持 + 各修正の新規テスト）、`pnpm lint`、`pnpm build`。
- 契約: `pnpm test:contract`。C6 で契約変更が入る場合はスキーマ生成（`schema-gen`）と drift テストを更新。
- E2E: `product/scripts/e2e-dry-run.mjs`（モック）と、local queue での実起動（必ず fail するタスク・`.harness.yml` 有無・依存ありリポジトリ）で C1/C3/C5 の挙動を目視確認。
- コミットは Tier 単位（最低でも C1 は独立コミット）で分割。
