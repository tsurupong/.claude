# CLAUDE.md

## このプロジェクトについて
Hisseki（筆跡）は、動画編集用に「手書き風フォントの見た目のまま、書き順通りにストロークが走り書きされていく」テキストアニメーションを作成するブラウザアプリ。漢字はKanjiVGの書き順データを用いる。

## 技術スタック
- 言語/フレームワーク: TypeScript, React 18, Vite, Tailwind CSS
- 状態管理: Zustand
- 主要依存: opentype.js（フォントアウトライン抽出）, @madcat/kanjivg（書き順データ）, upng-js（APNG書き出し）, pako, jszip, @dnd-kit/*（ドラッグ&ドロップ）, @ffmpeg/ffmpeg・@ffmpeg/util（動画書き出し、今後対応予定）
- インフラ: Terraform（AWS S3 + CloudFront + ACM + Route53 + Budgets によるキルスイッチ構成）

## コマンド
アプリ本体（`app/` ディレクトリ、package.jsonで確認済み）:
- `npm install`
- `npm run dev` — Vite開発サーバー（http://localhost:5173）
- `npm run build` — `tsc -b && vite build`
- `npm run preview` — ビルド結果のプレビュー
- `npm run lint` — `tsc --noEmit`（型チェックのみ、ESLint等は未確認）
- テスト実行コマンド: 未確認（テストスクリプト・テストファイルとも見当たらず）

インフラ（`infra/terraform/`、README記載を確認済み）:
- `terraform init` / `terraform plan` / `terraform apply`
- デプロイ: `infra/scripts/deploy.sh`（Windows向けに `.ps1` / `.bat` も存在）
- 更新/削除用スクリプトも `infra/scripts/update.*` `infra/scripts/destroy.*` として存在

## 構造
- `app/src/components/` — UIコンポーネント（`preview/` サブディレクトリにプレビュー関連をまとめる）
- `app/src/lib/` — アニメーションエンジン・フォント処理・各種エクスポーター等のロジック層
- `app/src/store/` — Zustandベースの状態管理（設定・プリセット・アプリ全体の状態を分割）
- `app/public/kanjivg/` — KanjiVGのSVGデータ本体（約6,744文字分、静的アセット）
- `infra/` — Terraformコードとデプロイ用シェル/PowerShell/バッチスクリプト
- ルート直下の `要件定義書_v1.0〜v1.3.md` — 要件定義書（バージョン管理されたドキュメント群）

## 規約・注意
- コミットメッセージは日本語 + Conventional Commits風プレフィックス（例: `fix(preview): ...`, `feat(infra): ...`, `refactor(infra): ...`）でスコープを括弧内に明記する慣習。
- 触ってはいけない/生成物として扱うべきもの:
  - `app/dist/`, `app/node_modules/`, `app/tsconfig.tsbuildinfo`（ビルド生成物、.gitignore対象）
  - `infra/terraform/.terraform/`, `infra/terraform/*.tfstate*`, `infra/terraform/*.tfvars`（stateや実際の設定値、.gitignore対象・機密含む可能性）
  - `.playwright-mcp/`（動作確認ログ・スクリーンショット等の一時生成物、.gitignore対象）
- ライセンス制約に注意: KanjiVGはCC BY-SA 3.0、フォント（Klee One / Yuji Mai等）はSIL OFL 1.1。アプリ本体はMIT。派生物公開時はクレジット表記が必要（README記載）。
- ルートREADMEの機能一覧は「v0.1.0」表記だが `app/package.json` は `0.4.0` — 推測: READMEの更新がバージョン管理に追随していない可能性がある。

## 現在の状態
- 推測: 直近のコミット履歴（`fix(preview)`, `feat(ui): スマホ向けレスポンシブ対応`, `feat(infra): subdomain delegation構成`, kill switch追加、WAF撤去→AWS Budgets移行）から、UIのモバイル対応とAWSインフラのコスト管理・自動停止機構の整備が直近の主な作業テーマと見られる。
- 解析時点で `git status` は未ステージの変更ありと報告しており、推測: 作業途中のブランチ状態である可能性がある。
- アプリREADME記載によれば Phase 1〜4（MVP、複数文字対応、手書き感・装飾、静止画/SVG/APNG書き出し）までは実装済みで、Phase 4.3以降（ffmpeg.wasmによる透過WebM書き出し）は未実装・今後対応予定。
