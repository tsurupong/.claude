# CLAUDE.md

## このプロジェクトについて
日本語の文(漢字・カタカナ・ひらがな)を形態素解析し、モーラ単位のローマ字候補木に変換するNode.jsライブラリ「manimani」。

## 技術スタック
- 言語: JavaScript (実装本体は `src/` 配下、CommonJS形式)
- 型定義: TypeScript(`src/types/manimani.d.ts`、`tsconfig.json`でdeclaration出力のみ設定。トランスパイルには使っていない)
- ランタイム: Node.js
- 主要依存: `kuromoji` ^0.1.2 (形態素解析器本体。`src/lib/kuromoji.js` はこれをラップし辞書パスごとにトークナイザーをキャッシュする自前の薄いラッパー)
- テスト: Jest ^29.7.0 (`jest.config.js`、coverageProvider: v8、カバレッジ収集有効)
- devDependencies: typescript ^5.0.0

## コマンド
- ビルド: `npm run build` (実体は `node scripts/build.js`。`src/` を再帰的に `dist/` へコピーするだけの簡易スクリプトで、TypeScriptのコンパイルは行っていない)
- テスト: `npm test` (実体は `jest`。coverage/ 配下にレポート出力)
- 実行(run): 未確認 (package.jsonに start/run スクリプトなし。利用形態はライブラリとして `require("manimani").tokenize(dicPath, sentence, callback)` を呼び出す方式。`example/index.ts` がサンプルコード)

## 構造
- `src/` : 実装本体。`manimani.js`(エントリ)、`Tokenizer.js`、`lib/kuromoji.js`(kuromojiラッパー+キャッシュ)、`mora/`(モーラ木構築ロジック群)、`dict/`(mecab-ipadic由来の辞書データ `.dat.gz`)、`types/`(型定義)
- `dist/` : `npm run build` で `src/` をそのままコピーした配布物。git管理下だが手で編集する対象ではない
- `test/` : Jestテスト(`Tokenizer.test.js`、`mora/*.test.js`)
- `example/` : 使用サンプル(`index.ts`)
- `scripts/build.js` : ビルドスクリプト本体

## 規約・注意
- `dist/` は生成物。編集は必ず `src/` 側に行い `npm run build` で再生成すること(手動編集禁止)
- `src/dict/*.dat.gz` は mecab-ipadic 由来の辞書データで `NOTICE.md`/`LICENSE` のライセンス表記対象。改変・削除は要注意
- コールバックは一貫してNode.js流儀の `(err, result)` エラーファースト形式(直近の履歴でこの形に統一された)
- `coverage/` はテスト実行時の生成物(`.gitignore`でも除外対象)。`.npmignore` では `.vscode`/`coverage`/`example`/`node_modules`/`test`/`jest.config.js` をnpm公開物から除外し、npm公開物は package.json の `files: ["dist"]` のみ
- リポジトリ直下に空ディレクトリ `-p`(未追跡、中身なし)が存在。推測: 誤操作(`mkdir -p`類)の残骸であり触れる必要はない

## 現在の状態
- 推測: 直近コミット `ab5bb5c fix!: resolve critical bugs and switch to Node-style error callback` により、コールバックAPIをエラーファースト形式へ統一する破壊的変更を行った直後
- 推測: その前段の `88b0f16 perf: cache kuromoji tokenizer per dicPath` でトークナイザーの再利用によるパフォーマンス改善が行われており、直近はAPI安定性と性能改善を交互に進めている
- package.json上のバージョンは `2.0.0` だが、git log内の最新バージョン更新コミットは `7b2cdfa ver: 1.2.0` であり、バージョン更新コミット自体は直近10件の履歴に見当たらない(未確認: 2.0.0への更新がどのコミットで行われたかはこのログ範囲では特定できず)
