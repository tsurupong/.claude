# CLAUDE.md

## このプロジェクトについて
「touchual」は、お題の文章をタイピングして正誤判定・WPM/正確率などの指標を計測するタイピングゲームである。フロントエンド(React)・API(AWS Lambda)・通知・IaC・ローカルDB・テストデータ生成の各リポジトリで構成されるマルチリポジトリ構成のプロジェクト。

## 技術スタック
- 言語: TypeScript(api/client/notification)、JavaScript(testdata-generator)
- フロントエンド: React 19 / React Router v7(SSR)、Zustand、SWR、Tailwind CSS、Sass、Storybook、Vitest + Playwright
- バックエンド: AWS Lambda(Node.js 20)、esbuild、DynamoDB(@aws-sdk/client-dynamodb)、形態素解析ライブラリ `manimani`(モーラ分割用)
- インフラ: AWS SAM(CloudFormation)、S3、CloudFront、WAF想定(featurespec記載、現状はLambda構成)
- ローカル環境: Docker Compose(DynamoDB Local)
- 通知: axios + dotenv(Lambda、詳細未確認)

## コマンド
### touchual-client
- 開発: `npm run dev`(ローカル)/ `npm run dev-cloud`
- ビルド: `npm run build-dev` / `npm run build-prod`
- デプロイ: `npm run deploy-dev` / `npm run deploy-prod`(S3 sync)
- 型チェック: `npm run typecheck`
- テスト: Vitest/Playwright設定あり(vitest.workspace.ts)。具体的な `test` scriptはpackage.jsonに未定義 — 未確認
- Storybook: `npm run storybook` / `npm run build-storybook`

### touchual-api
- 開発: `npm run dev`(tsx watch、ローカルExpressサーバー)
- ビルド: `npm run build`(esbuild)
- パッケージング/デプロイ: `npm run packing`、`npm run deploy:dev` / `deploy:prod`(S3へzip配置)
- テスト: `npm test` は未実装(`exit 1` のプレースホルダ)

### touchual-notification
- ビルド: `npm run build`、`npm run packing`
- テスト: 未実装(プレースホルダ)

### touchual-testdata-generator
- 実行: `npm start`(`node src/index.js`)

### touchual-IaC
- Node/npmスクリプトなし。`.bat`スクリプト群(Windows前提): `scripts/build.bat`, `deploy.bat`, `deploy-stack.bat`, `destroy.bat`, `destroy-stack.bat`, `validate.bat`, `setup-local-db.bat`, `insert-testdata.bat`

### touchual-dynamodb-local
- `docker-compose.yml` によるDynamoDB Local起動。テーブル作成/データ投入は `scripts/*.bat`

## 構造
- `touchual-client/` : React Router製フロントエンド(app/routes配下にtop, countdown, game, resultの画面遷移)
- `touchual-api/` : タイピングお題取得API(Lambda)。`resolver.ts`がDynamoDBクエリ、`tokenizer.ts`が`manimani`でモーラ変換
- `touchual-notification/` : 通知用Lambda(詳細ロジック未確認)
- `touchual-IaC/` : SAMテンプレート・Lambdaレイヤー・環境構築スクリプト
- `touchual-doc/` : 仕様書(API/データモデル/機能/UI、drawio図含む)
- `touchual-dynamodb-local/` : ローカル開発用DynamoDB(Docker)
- `touchual-testdata-generator/` : テストデータ生成・出力(test-data-*-dev/prod.json)

## 規約・注意
- 各サブディレクトリ(touchual-IaC/api/client/doc/dynamodb-local/notification)はそれぞれ独立したgitリポジトリ(親ディレクトリ自体はgit管理外)。コミットはサブリポジトリ単位で行う慣習。
- コミットメッセージは日本語の `feat:` `fix:` `chore:` `add:` プレフィックス混在(Conventional Commits風だが厳密ではない)。
- `.gitignore` により `node_modules`, `dist`, `build`, `*.zip`, `.env.*`, `.react-router`, `.aws-sam` は生成物として除外・触らない対象。
- APIはCloudFront経由以外を403で拒否する仕組み(`x-origin-verify`ヘッダ + `ORIGIN_VERIFY_SECRET`)、CORS allow-listあり。環境変数の扱いに注意。
- ルート直下の `touchual-sam-agent_accessKeys.csv` / `touchual-sam-agent_credentials.csv` はAWS認証情報とみられるため絶対に開かない・コミットしない。
- `.env.local` / `.env.dev` / `.env.prod` が各リポジトリに存在(内容未確認・秘密情報の可能性)。
- ライセンスはMIT(各サブプロジェクトにLICENSEファイルあり)。

## 現在の状態
- 推測: touchual-IaCは直近でEC2/ALB構成からLambda構成への移行(`2a2bece`)を終え、Lambdaメモリ増強やCORS/環境変数修正など運用チューニング段階(`8f7887d`, `4c9cee8`, `b19f4c8`)。
- 推測: touchual-apiはGraphQLを廃止しREST直接ハンドラへ移行済み(`72e38da`)、直近はローカル開発用Expressサーバー追加(`bf9dd68`)。
- 推測: touchual-clientも同時期にGraphQL→REST形式へ変更(`0cf6e5e`)し、react-router v7.14.0へアップデート(脆弱性修正含む)、レベル進行速度の調整など機能仕上げ段階(`b49c50b`)。
- touchual-doc/touchual-dynamodb-localは比較的初期のセットアップ的なコミット履歴で、大きな追加変更は少ない — 情報なし(詳細な最新動向は未確認)。
