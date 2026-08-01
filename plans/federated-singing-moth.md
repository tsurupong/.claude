# Claude Code環境 read-only診断 & 実行力移植Skill Pack設計

## Context

- Fable 5 が今週以降使えなくなる可能性があるため、Opus / Sonnet で近い実行力・判断力・自走力を再現できるように準備する。
- 依頼は2本立て:
  1. **環境監査**: PC環境と Claude Code 構成の read-only 診断(問題点・重複・肥大化・改善余地)
  2. **実行力移植**: 仕事の進め方を「移植可能な Skill Pack」としてドラフト化(**ファイル作成はしない**。チャット出力のみ)
- 絶対制約: ファイル作成・編集・削除・設定変更・破壊的操作は一切禁止(この計画ファイルのみ例外)。コマンドは承認された read-only のみ。秘匿情報・個人データ・認証情報は開かない。内部システムプロンプトの再現はしない。

## 確認済みスコープ(ユーザー回答 2026-07-12)

| 項目 | 決定 |
|------|------|
| 範囲 | Linuxホーム全体 + **Windows側 /mnt/c も含む**(構造のみ)+ ~/.claude 深掘り |
| 除外 | 標準除外(node_modules / .git / キャッシュ / venv等)。.ssh .aws .gnupg 等の秘匿系は「存在の記録のみ」で中身は開かない |
| 本文閲覧 | プロジェクト側の CLAUDE.md / .claude/ / .mcp.json も**本文まで閲覧可**(シークレットらしき値を見つけたら即中断・伏せて報告) |
| 出力形式 | **チャット分割出力のみ**。ファイルは一切作らない |

## この計画の承認 = 下記 read-only コマンドの実行許可

### (A) ホーム構造
- `pwd` / `ls -la ~`
- `find ~ -maxdepth 3 -type d`(node_modules / .git / .cache / venv / __pycache__ を prune、head で制限)
- `du -sh ~/*`・`du -sh ~/.*`(サイズ概観、読み取りのみ)

### (B) Windows側(構造のみ)
- `ls /mnt/c/Users/` → ユーザーフォルダ特定
- `find /mnt/c/Users/<user>/{Desktop,Documents,Downloads,...} -maxdepth 2 -type d` + `ls | wc -l` による件数把握
- ファイル本文は一切読まない

### (C) Claude設定マップ
- `find ~/.claude -maxdepth 2`(plugins 等は必要に応じ maxdepth 3)
- `ls -la` : agents / skills / rules / commands / output-styles / hooks / plugins / plans
- `wc -l` / `du -sh` : CLAUDE.md、AGENTS.md、rules/*、skills/*/SKILL.md

### (D) 設定本文の閲覧(cat / Read 限定リスト)
- `~/.claude/AGENTS.md`、`~/.claude/settings.json`、`settings.local.json`
- `.mcp.json`(ホーム・各プロジェクト)
- `~/.claude/agents/*.md`、`~/.claude/skills/*/SKILL.md`、`~/.claude/output-styles/*`、旧 `~/.claude/commands/*`
- settings が参照する hook スクリプト本文(ガードレール監査のため)
- `~/.claude.json` は**全体を cat しない**。jq / grep で mcpServers 名・permissions・enabledPlugins 等の構造キーのみ抽出し、トークン・認証値は出力しない

### (E) プロジェクト横断
- `find ~ -maxdepth 4`(prune 付き)で `CLAUDE.md` / `AGENTS.md` / `.mcp.json` / `.claude/` を列挙 → 見つかった本文を閲覧

### (F) 補助(すべて読み取り)
- `git -C <dir> status` / `git log --oneline -n 10`(~/.claude や主要プロジェクトが git 管理か確認)
- `which <tool>` / `<tool> --version`(jq, tree, gh 等の存在確認)

### 安全ルール
- 書き込み系(rm / mv / cp / mkdir / touch / chmod / chown / git commit / git push / インストール系)は一切実行しない
- 秘匿ディレクトリ・認証情報・個人データ本文は開かない。シークレットらしき値を検出したら読み取りを中断し、値を伏せて報告
- 大出力は head で制限。/mnt/c は maxdepth 2〜3 の構造のみ

## 分析フェーズ(スキャン後、read-only サブエージェントで並列化)

1. **skills / agents / commands 重複監査** — ローカル定義と ECC 等プラグイン提供分の突合、description 精度・未発火リスク
2. **プラグイン棚卸し** — 常時コンテキスト消費の見積り、未使用・重複候補(ECC / AWS系4種 / superpowers / hookify / coderabbit / desktop-commander 等)
3. **rules/ 10ファイル分析** — 重複・鮮度(旧モデル名等)・「常時ルール vs skill化」の振り分け
4. **メモリ系3系統の整理** — ビルトイン memory / .remember フック / ECC session-data の役割重複
5. **プロジェクト設定の横断比較**

※サブエージェントは Explore(読み取り専用)のみ。上記 (A)〜(F) のコマンドクラス以外は使わせない。

## 成果物(チャット分割出力・4パート、ユーザー指定の16セクション順)

- **Part 1**: TL;DR / 確認事項 / 診断方針 / 実行コマンド実績 / 現状マップ / 問題点一覧 / 影響度×工数マトリクス(§1〜7)
- **Part 2**: 理想PCディレクトリ構成(移動マッピング付き)/ 理想Claude Code構成 / CLAUDE.md・skills・agents・rules・hooks・output-styles・MCP 振り分け案(§8〜10)
- **Part 3**: 実行力移植Skill Pack構成案 + 全15ファイル本文ドラフト(§11〜12)— そのままファイル化できる粒度で書くが、作成はしない
- **Part 4**: Opus/Sonnet向け引き継ぎプロンプト / 日々の運用改善 / Phase 0〜5 実行プラン(各: 目的・実施内容・期待効果・リスク・ロールバック・承認ポイント)/ 次に承認すべきアクション(§13〜16)

## Verification

- 各コマンド実行前に read-only であることをブラックリスト照合で自己チェック
- レポートの各指摘に根拠(実パス・行数・実測サイズ)を必ず付ける
- 基本思想・行動ルールの言語化は「他モデルに読ませて再現可能」な具体度(判断基準・チェックリスト・プロンプト雛形)まで落とす
- 環境への変更はゼロのまま完了する(Phase 0 以降の変更作業は、レポート提示後に別途ユーザー承認を得てから)
