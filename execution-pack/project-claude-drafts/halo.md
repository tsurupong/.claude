# CLAUDE.md

## このプロジェクトについて

HALO（Harness for Autonomous Loop Orchestration）は、Claude Code（headless）が GitHub Issue を起点に「実装 → 品質検証 → PR作成」までを人間の介在なしに反復実行するための、汎用の自律開発ハーネスである。夜間無人稼働で品質ゲートを通過したPRを継続的に生成することを目標とする。

## 技術スタック

- 言語: TypeScript（`packages/`）、Bash（`plugins/` の各アダプタ）
- ランタイム: Node.js >= 22、パッケージマネージャは pnpm 10.14.0（pnpm workspace monorepo）
- テスト: Vitest（`@vitest/coverage-v8` によるカバレッジ計測）
- Lint/Format: ESLint（typescript-eslint）、Prettier
- スキーマ: ajv、ts-json-schema-generator（`@halo/contracts` でJSON Schema生成）
- 主要内部パッケージ: `halo`(CLI) / `@halo/core` / `@halo/contracts`

## コマンド

すべて `product/` ディレクトリ直下で実行（ルートに package.json は無い）。

- ビルド: `pnpm build`（`tsc -b`）
- テスト: `pnpm test`（`vitest run`）
- 契約テストのみ: `pnpm test:contract`
- テスト監視: `pnpm test:watch`
- カバレッジ: `pnpm coverage`
- Lint: `pnpm lint`（`eslint packages`）
- フォーマット確認/適用: `pnpm format` / `pnpm format:write`
- E2Eドライラン: `product/scripts/e2e-dry-run.sh`（未確認: モックのみで実GitHub連携は未検証）

## 構造

- `docs/` : 要件定義書・設計書一覧・タスクリスト（`docs/tasks/phase1-tasks.md`）。git非管理（product外）。
- `product/` : 実装リポジトリのルート（git管理はここのみ）。
- `product/packages/{cli,core,contracts}` : CLIコマンド・コア状態機械・コントラクト(型/JSON Schema)。
- `product/plugins/*` : ポート実装（task-source-github, executor-claude, gate-loop-audit, gate-runtime-check, sink-progress-log, on-fail-record, runtime-node-pnpm, trigger-polling, trigger-schedule）。各々 `plugin.json` + シェルスクリプトで構成。
- `product/docs/{adr,design}` : ADR(0001-0012)と設計書(D1-D9)。設計判断の一次情報源。

## 規約・注意

- **自己改変の禁止（ADR-0004）**: `CLAUDE.md` / `PROMPT.md` / `.harness.yml` / テストファイルへのエージェントによる変更は `gate-loop-audit` で fail 扱いとする安全不変条件。無人ループ中にこれらを書き換えてはならない。
- プラグイン命名規則: `<port種別>-<実装名>`（例: `task-source-github`, `gate-loop-audit`, `sink-progress-log`, `on-fail-record`, `trigger-polling`）。各プラグインは `plugin.json`（name/version/port/exec）+ `test.contract.sh` + `contract.fixtures.json` を持つ。
- モノレポは pnpm workspace（`packages/*`）。ルートに `packageManager` 固定指定あり、バージョン不一致に注意。
- `product/docs/design/d6-graph-design.md` は設計書一覧上「私有区分」だがgit追跡中 — OSS公開前に除外要否の判断待ち（`.remember/now.md` より）。
- `node_modules/`・`.git/` は読み取り対象外（本ドラフト作成時も未参照）。
- GitHubリモート未設定（ローカルリポジトリのみ）。push時に `product/.github/workflows/ci.yml` が初稼働する想定。

## 現在の状態

- 直近のコミット（`product` の `git log --oneline -10`）は、pnpm isolated node_modules 対応の `@types/node` 追加修正、リポジトリ再編（`docs/`と`product/`分離、git管理をproductのみに変更）、Phase 1完了に伴うレビュー修正・CLI/コア/プラグイン実装（M1〜M6）が中心。
- 推測: Phase 1（monorepo scaffold・contracts・core・CLI・見本プラグイン・テスト整備）は完了済みで、299〜303件のテストが green の状態（`.remember/now.md` の記録に基づく）。
- 推測: 現在はPhase 2に向けた繰越タスク（core-ext のcore昇格、実GitHubでのE2Eスモークテスト、自律度プロファイルのチューニング、OSS公開前のリモート未設定・私有ドキュメント除外判断）が残っている段階。
