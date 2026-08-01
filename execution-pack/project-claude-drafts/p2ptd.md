# CLAUDE.md

## このプロジェクトについて
p2ptd は、1vs1 のリアルタイムPvP/PvE対戦を想定した、タワーディフェンス要素を持つRTS(リアルタイムストラテジー)ゲームの個人開発プロジェクト。P2Pロックステップ同期方式を採用し、ブラウザ上でWebRTC/WebSocketを介して対戦する構成で実装が進められている。

## 技術スタック
- コアロジック: Rust (edition 2024) → WebAssembly。wasm-bindgen / web-sys でWebRTC DataChannel・WebSocketをRust側から直接制御。serde/serde_json、hmac+sha2+base64(改ざん検知用途と推測)
- フロントエンド: TypeScript + React 19 + Vite + PixiJS(描画) + Zustand(状態管理)。coreのwasm成果物をfile:参照(`p2ptd-core`)で取り込み
- シグナリング/マッチメイキングサーバー: TypeScript(Node.js) + Hono + @hono/node-ws + ws、tsx で実行
- 補助: server配下にPlaywright(devDependencies)。手動ブラウザ検証にも使用されている形跡あり(.playwright-mcp/ログ)

## コマンド
サーバー(`7.app/server/package.json`確認済み):
- 開発起動: `npm run dev` (tsx src/index.ts)
- ビルド: `npm run build` (tsc)
- 本番起動: `npm run start` (node --import tsx dist/index.js)
- test: 未確認(npm test相当のスクリプトなし。`server/scripts/test_matching.ts`等は個別にtsx実行する想定と推測)

Webクライアント(`7.app/clients/web/package.json`確認済み):
- 開発起動: `npm run dev` (vite)
- ビルド: `npm run build` (tsc -b && vite build)
- lint: `npm run lint` (eslint .)
- preview: `npm run preview`
- test: 未確認(テスト用スクリプトなし)

コア(`7.app/clients/core/Cargo.toml`確認済み):
- ビルド/テストコマンド: 未確認(Cargo.tomlにカスタムスクリプト定義なし。cdylib+wasm-bindgen構成のため`wasm-pack build`相当が想定されるが明示的な記載は見つからず)

## 構造
- `0.common`〜`6.datamodel`: 用語定義・ゲームデザイン・ネットワーク・非同期処理・セキュリティ・アーキテクチャ・データモデルの設計ドキュメント群(番号は工程順)
- `7.app/clients/core`: ゲームロジック本体(Rust→WASM、決定論的シミュレーション)
- `7.app/clients/web`: フロントエンド(React/Vite/PixiJS)、coreの成果物を取り込み描画・UIを担当
- `7.app/server`: シグナリング・マッチメイキング用サーバー(Hono)

## 規約・注意
- 決定論保証原則(`0.common/用語定義書.md` §7、全実装者必読): P2Pロックステップ同期のため、Rust/WASM coreではf32/f64禁止・固定小数点演算必須、乱数は`world.sharedRandom`経由、システムクロック依存禁止。TypeScript側はゲームロジック計算を行わずCoreへ委譲する方針。
- ディレクトリ名の数字接頭辞(`0.`〜`7.`)は設計・実装工程の順序を表す命名規則であり、変更しない。
- 生成物・ログ類は編集/コミット対象外と推測: `7.app/clients/core`配下の`build_errors.json`, `build_errors.txt`, `clippy_output.txt`, `error.log`, `error2.log`, `errors.json`, `error_log.txt`, `tmp_errors.txt`(いずれもビルド/lint実行時の一時出力とみられる)。
- ビルド成果物ディレクトリは直接編集しない: `core/pkg`, `core/target`, `server/dist`, `web/dist`, `web/src/wasm`(coreからコピーされるwasmバインディング一式)。
- ルート直下にpackage.json/READMEは存在せず、ビルド設定は`7.app`配下の各サブプロジェクトに閉じている。
- `7.app/clients/core`配下に独自の`.git`が存在するが、コミット履歴は0件(実質未使用)。

## 現在の状態
- git log: プロジェクトルートはgit管理下になし。`7.app/clients/core`配下に`.git`はあるがコミット0件のため、コミット履歴からの活動把握は不可(情報なし)。
- 推測: ファイル更新日時を見ると`core/pkg`(wasm再ビルド成果物)、`core/src/systems/guardian_lane.rs`、`core/src/types.rs`、`1.design/ユニット・タワー.md`が直近の更新であり、"ガーディアン/レーン"関連システムとユニット・タワー仕様の実装調整が直近の作業と考えられる。
- 推測: `.playwright-mcp/`配下に蓄積されたconsoleログ・pageスナップショットから、WebRTC/マッチメイキング周りのブラウザ動作確認をPlaywright MCP経由で行っている。
