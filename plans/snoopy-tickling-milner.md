# 設計書修正計画 — Agent SDK 乖離調査の反映

## Context

deep-research による Agent SDK(agent-loop/hooks/permissions)調査で、HALO との乖離3点が確定した:

- **A. 安全不変条件の事前強制**: SDK は PreToolUse hook / deny ルール(bypassPermissions でも有効)で実行前ブロックを提供。HALO 実装は gate-loop-audit の事後検査のみ。ただし **D4 設計書上は deny + hook の事前層が既に設計済み**で、実装(executor-claude は settings 注入なし)と docs が乖離している。
- **B. permission mode 方針の不在**: 実装は `--permission-mode acceptEdits`(`packages/plugins/src/executor-claude/main.ts:17`)だが、docs 全体に permission mode の概念が不在。SDK 公式のロックダウン推奨は `allowedTools` + `dontAsk`(列挙外は即 deny)。
- **C. コスト上限停止の空白**: SDK は maxBudgetUsd を提供。HALO は回数ベース日次予算 + 月次人手判断のみで、`executor.out.cost.usd` を停止判定に接続する設計がどの文書にも無い。

ユーザー決定: ADR は新規起票で supersede(既存は status 更新のみ)/ C もスコープに含める。

**本計画は docs の修正のみ。実装変更は含まない**(docs 確定後の別タスク)。

## 新規作成(3件)

### 1. ADR-0019: 実行前権限強制(pre-execution enforcement)
`product/docs/adr/0019-pre-execution-permission-enforcement.md`(template.md 準拠)
- Decision: 安全不変条件(ADR-0004 の保護対象)の第一層を settings deny ルール(実行前)、第二層を gate-loop-audit(事後)とする二層強制。executor-claude が起動時に settings を注入する。
- 根拠: deny は bypassPermissions でも有効(SDK permissions doc、3-0 検証済み)。事後 gate はイテレーション丸損のコストを払う。
- ADR-0004 を supersede ではなく **補強**(0004 の Decision は有効のまま、強制手段を拡張)。関係を Links に明記。

### 2. ADR-0020: executor permission 構成(allowedTools + dontAsk)
`product/docs/adr/0020-executor-permission-profile.md`
- Decision: executor-claude の既定を `acceptEdits` から `allowedTools 明示 + --permission-mode dontAsk` へ変更(列挙外ツールは即 deny)。`HALO_CLAUDE_PERMISSION_MODE` env は override として残す。
- ADR-0006(自律度)との直交関係(autonomy レベル × permission 構成)を Consequences に記載。

### 3. ADR-0021: 金額ベース停止条件(max budget USD)
`product/docs/adr/0021-cost-based-stop-condition.md`
- Decision: `executor.in.budget` に `max_budget_usd`(optional)を追加し、`executor.out.cost.usd` の累積を PreflightLight の残予算判定に接続。契約変更は d1 §7 semver 上 minor。
- ADR-0012(数値の事前固定禁止)との整合: 閾値既定値は固定せず「機構」のみ規定、値は config で与える。

## 既存文書の改訂(7件)

| 文書 | 改訂内容 | 対応 |
|---|---|---|
| `adr/0004-self-modification-prohibition.md` | status に「amended by 0019」追記、Decision は不変。Risks の「gate のみ」前提を 0019 参照に更新 | A |
| `adr/README.md` | 表に 0019-0021 追加。**supersede/amend 運用の流儀を冒頭に定義**(status 列の拡張: accepted / superseded-by-NNNN / amended-by-NNNN) | 共通 |
| `design/d4-security-design.md` | §2 の deny 標準セットに「executor-claude が起動時注入する」実行時フローを追記(現状は静的配置のみで注入主体が未定義)。§6 に permission mode(dontAsk)方針を追記。§Pending の暫定マーク解消 | A, B |
| `design/d1-contract-spec.md` | §1.3 `executor.in.budget` に `max_budget_usd`(optional)追加、起動コマンド例に settings 注入と `--permission-mode dontAsk` を反映。§7 semver で minor bump 明記 | B, C |
| `design/06-security-cost-observability.md` | §5 コスト制御表に `MAX_BUDGET_USD` 行を追加、iter_N.json の `usd_estimate` → 停止判定の接続を記述。§2 の deny/hook 記述を D4 改訂に追随 | C, A |
| `design/d3-cli-spec.md` | フラグ表に `--max-budget-usd` 追加(§71/284行付近)、§終了コード「budget超過=exit 0」にコスト超過を含める | C |
| `design/d5-plugin-dev-guide.md` | executor 入力例(251行付近)の budget を3フィールドに更新、permission 構成の解説を追記 | B, C |

d2(日次予算)・d9(watchdog)・02-executor は主管外のため**改訂しない**(必要なら参照追記のみ)。

## 作業順序

1. ADR-0019/0020/0021 起草(template.md 準拠、Proposed で起票)
2. adr/README.md に運用流儀 + 3件追加
3. D4 → d1 → 06 → d3 → d5 の順で改訂(依存: ADR 確定 → 主管設計書 → 派生文書)
4. adr/0004 の status 追記

## 検証

- 各 ADR が template.md の章立てに準拠していること
- d1 の budget 契約変更が semver 方針(§7)と矛盾しないこと
- `grep -r "permission" product/docs/` で新旧記述の矛盾(acceptEdits 残存等)が無いこと
- 相互参照(0019↔0004、d1↔d3↔06 の max_budget_usd)の整合を読み合わせで確認
- **注意**: docs は `product/` 内 = git 管理下。コミットは type: docs で、push は承認後
