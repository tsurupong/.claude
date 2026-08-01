# CLAUDE.md

## このプロジェクトについて
Ghost Rhythm（ゴーストリズム）は「タイピング×リズムゲーム×対戦」をコンセプトにしたゲーム企画。本ディレクトリ（実体は `ghost-rhythm-doc/`）は、その企画書・仕様書一式を管理する設計ドキュメントリポジトリであり、現時点で実装コードは存在しない。

## 技術スタック
- 実装用のマニフェスト（package.json / Cargo.toml / pyproject.toml 等）は未確認（存在しない）。
- `02_spec/03_system/技術選定一覧.md` に記載された**計画上の**スタック（未実装・未検証）:
  - フロントエンド: Next.js (App Router) / Tailwind CSS / Zustand
  - バックエンド: NestJS / Zod（バリデーション）
  - DB: PostgreSQL + Prisma
  - ストレージ: S3互換（AWS S3 / Cloudflare R2 / MinIO）
  - インフラ: Fly.io、GitHub Actions（CI/CD）
  - テスト: Jest（Unit）/ Playwright（E2E）
  - Lint/Format: ESLint / Prettier

## コマンド
未確認（build/test/run 用のマニフェストがリポジトリ内に存在しないため、実行コマンドは特定できない）。

## 構造
- `01_plan/plan.md` : 企画書（コンセプト・ゲームモード・ロードマップ）
- `02_spec/01_entire/`, `02_gameplay/` : ゲームデザイン・シナリオ・操作・譜面仕様
- `02_spec/03_system/` : ユースケース、機能仕様、エラー仕様、DB、クライアント/サーバー仕様、技術選定
- `02_spec/04_src/` : モノレポ想定のディレクトリ構成図（`パッケージ構成.md`）と、その雛形を示す空/モックファイル（`p/app/...`）

## 規約・注意
- 仕様書ファイルはすべて日本語（「〜仕様書.md」「〜定義書.md」等の命名慣習）で作成されている。
- `.drawio` ファイルと対になる `.$xxx.drawio.bkp` は draw.io の自動生成バックアップであり、`.gitignore`（`/**/*.bkp`）でも除外対象。手動編集・コミット対象にしない。
- `02_spec/04_src/p/` 配下の `.tsx`/`.ts` ファイル（root.tsx、routes.ts、layout.tsx 等）は中身が空のモック（将来のモノレポ構成を示すプレースホルダ）であり、実装コードではない。実装作業の起点として誤用しないこと。
- リポジトリ本体は `GhostRhythm/ghost-rhythm-doc/`（GitHub: `TsuruPong/ghost-rhythm-doc`、`main` ブランチ）であり、`GhostRhythm/` 直下はこの1ディレクトリのみを含む。

## 現在の状態
直近の git log（新しい順）:
1. モックのファイルを追加
2. クライアントのパッケージ構成を追加
3. パッケージ図のumlを追加
4. service層を追加
5. パッケージ構成修正
6. パッケージ構成のドラフトを追加
7. Create ユースケース定義書.md
8. ranking_idに修正
9. 機能仕様書いったん作成
10. ファイル名修正

推測: 企画・仕様策定フェーズはほぼ完了し、直近はモノレポのディレクトリ構成（`04_src/`）とクライアント側の雛形（モックファイル）整備に着手した段階。実装コード（アプリ本体）はまだ着手前と推測される。
