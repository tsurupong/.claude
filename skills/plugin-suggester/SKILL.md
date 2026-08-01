---
name: plugin-suggester
description: 導入済みだが無効化されているプラグインを、タスクに合致した時に提案する。ブラウザ操作・E2Eテスト・スクリーンショット、AWS(デプロイ・Lambda・CDK・料金)、GitHub連携(PR・Issue)、PRレビュー、UIデザイン、Python開発、プラグイン/MCP開発、コード検索、リファクタリングの依頼が来たら必ずこのスキルを参照し、眠っているプラグインで楽になるものがないか確認する。ユーザーが「〜できるツールなかったっけ」「プラグイン何入ってたっけ」と聞いた時も使う。
---

# plugin-suggester

この環境には「いつか使うかも」で導入されたまま**無効化されている**プラグインが多数ある(2026-07-29 の環境監査でコンテキスト削減のため一括無効化)。無効プラグインはコンテキストに現れないため、Claude は存在を知らない。このカタログがその記憶の代わりである。

## 動き方

1. 依頼内容が下のカタログのいずれかに合致するか確認する。
2. 合致したら、作業を始める前に一言提案する:
   「この作業は **<plugin>** があると楽です(<理由>)。`/plugin` から有効化して再起動しますか? なくても進められます。」
3. **有効化はユーザー操作**。settings.json を勝手に書き換えない(環境設定変更は承認事項)。
4. ユーザーが不要と言ったら、そのまま現在の手段で進める。二度目の提案はしない。
5. 合致しなければこのスキルには言及せず普通に作業する。

提案は控えめに1回だけ。プラグインなしでも作業が成立するなら、その旨を必ず添える(押し売りしない)。

## カタログ(導入済み・無効のプラグイン)

| プラグイン | できること | 提案すべき状況 |
|---|---|---|
| playwright | ブラウザ自動操作(MCP) | E2Eテスト、Webアプリの動作確認、スクリーンショット取得、フォーム操作 |
| chrome-devtools-mcp | Chrome DevTools 操作(MCP) | パフォーマンス計測(LCP等)、メモリリーク調査、コンソール/ネットワーク解析。playwright と重複あり — 計測系はこちら、操作系は playwright |
| desktop-commander | デスクトップ・プロセス操作(MCP) | 長時間プロセスの対話操作、OS横断のファイル操作 |
| aws-core | AWS 全般(CLI プロキシMCP + IAM/CDK/Bedrock等スキル) | AWS リソース操作、IAM 設計、Bedrock、コスト分析 |
| aws-serverless | Lambda / API Gateway / Step Functions | サーバーレス構築・デバッグ、SAM |
| deploy-on-aws | AWS へのデプロイ支援・構成図・料金試算 | 「AWSに載せたい」「コスト見積もり」「構成図を描いて」 |
| aws-dev-toolkit | AWS 開発補助 | 上記 AWS 系とセットで検討 |
| github | GitHub 連携 | PR・Issue の一括操作、リポジトリ横断作業(gh CLI で足りない時) |
| code-review | PR の多角レビュー(専用エージェント) | 「PRをレビューして」。自作 review-orchestrator / code-reviewer で足りるなら不要 |
| coderabbit | CodeRabbit 連携レビュー | CodeRabbit を使う方針の時のみ |
| feature-dev | 機能開発ガイド(explorer/architect/reviewer エージェント) | 大きめの機能追加。自作 impl-planner/impl-builder/code-reviewer と重複 — 通常は自作側を使う |
| code-simplifier | 機能を変えないリファクタリング | 「コードを整理して」「簡潔にして」 |
| frontend-design | 差別化された UI デザイン指針 | 新規 UI 構築、デザイン刷新 |
| python-development | Python 専門エージェント(Django/FastAPI)+ パターン集 | Python 本格開発、Django/FastAPI 案件 |
| plugin-dev | Claude Code プラグイン開発 | プラグイン・フック・コマンドを作る時 |
| mcp-server-dev | MCP サーバ開発 | MCP サーバ・MCPB を作る時 |
| skill-creator (plugin版) | スキル作成・評価 | 自作スキル `skill-creator` と重複 — 通常は自作側で足りる |
| sourcegraph | Sourcegraph コード検索(MCP) | 巨大リポジトリ・組織横断のコード検索 |
| context7 | ライブラリ最新ドキュメント参照(MCP) | 新しめのライブラリの API を正確に知りたい時 |
| exa | Web 検索(MCP) | 組み込み WebSearch で不足する高度な検索 |
| hookify | 会話から再発防止フックを生成 | 「この失敗を今後防ぎたい」「フールプルーフにして」 |
| agent-sops | SOP 駆動のエージェント手順書 | 定型業務の手順書化・SOP 実行 |
| atomic-agents | エージェント構成部品 | マルチエージェント設計の実験時 |
| ecc | オーケストレーション(計画・ビルド・レビュー) | 過去に多用。大規模オーケストレーションを試す時 |
| explanatory-output-style | 解説多めの出力スタイル | 学習目的で挙動の説明を増やしたい時 |

## カタログの保守

プラグインを入れ替えたらこの表を更新する。実態は `~/.claude/settings.json` の `enabledPlugins`(true/false)が真実 — 提案の前にここで無効のままか確認すると確実(有効化済みならプラグインは既にコンテキストにいる)。
