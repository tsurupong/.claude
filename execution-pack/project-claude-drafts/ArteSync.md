# CLAUDE.md

## このプロジェクトについて

ArteSync (`arsync`) は、Claude / Cursor / Gemini など AI コーディングアシスタント向けの「Agent Skills」を、複数プロジェクト間で同期・管理するための Rust 製 CLI パッケージマネージャーである。README に「実験的ツール」と明記されており、破壊的変更が予告なく発生し得る段階。

## 技術スタック

- 言語: Rust (edition 2021)
- 主要依存 (Cargo.toml): `clap`(CLIパーサ, derive), `colored`, `dirs`, `fs_extra`, `serde`/`serde_json`/`serde_yaml`, `tempfile`, `thiserror`
- 開発依存: `tempfile`
- 配布: npm 経由 (`npm/artesync`) でプラットフォーム別バイナリを `optionalDependencies` として配布するラッパー方式 (`@artesync/{os}-{arch}`)
- CI: GitHub Actions (`.github/workflows/release.yml`) — タグ push (`v*`) をトリガに Linux/Windows/macOS(x64/arm64) 向けにビルドし npm publish

## コマンド

- ビルド: `cargo build --release`(README・release.yml で確認)
- テスト: `cargo test`(`#[test]` が `src/main.rs`, `src/core/domain/validation.rs`, `src/core/usecase/install.rs` に存在することを確認。ただし専用のテスト実行スクリプトは未確認)
- 実行(バイナリ): `arsync init` / `arsync install <source>` / `arsync install`(ロックファイルから復元) / `arsync update` / `arsync list` / `arsync uninstall <skill>`(README に記載されコマンド一覧と `src/main.rs` の実装が一致)
- npm ラッパー側の `npm test`(`npm/artesync/package.json`)は `echo "Error: no test specified" && exit 1` のダミーで未実装
- lint/format 設定 (`rustfmt.toml`/`clippy.toml` 等): 未確認(リポジトリ内に見当たらず)

## 構造

- `src/main.rs`: エントリポイント。CLI引数解析とユースケース呼び出しの配線(DI)を行う
- `src/core/`: ドメイン層 — `domain/`(Manifest・Lockfile・Skill等のモデルとエラー)、`port/`(トレイトによるインターフェース定義)、`usecase/`(init/install/list/uninstall/update)というヘキサゴナルアーキテクチャ(ポート&アダプタ)構成
- `src/infra/`: `core::port` の実装 — `fs/`(ローカルファイルシステム)、`git/`(git fetcher)、`manifest/`(manifest/lockfileの読み書き)
- `src/cli/`: `clap` によるコマンドライン引数定義(`parser.rs`)
- `npm/artesync/`: npm配布用ラッパー(`bin/arsync.js` がプラットフォーム別バイナリを検出し起動)

## 規約・注意

- `target/` はビルド生成物、`.gitignore` で除外(触らない)
- `skills.arsync` / `skills-lock.arsync` はこのリポジトリ自身の設定ファイルだが `.gitignore` に含まれておりリポジトリでは追跡外扱い(ローカル専用生成物とみられる)
- `err.txt` も `.gitignore` 対象。中身は v0.1.0 時点の `tempfile` 未解決エラーのビルドログで、開発時の残骸(参照不要、削除・追跡対象外)
- ドメイン層 (`core/`) とインフラ層 (`infra/`) の分離(ポート&アダプタ)を崩さないこと。`core` から `infra` への直接依存を増やさない設計思想が読み取れる
- `docs/` ディレクトリは存在しない(情報なし)

## 現在の状態

- 直近のコミット (git log --oneline -10): バージョン0.2.0へのbump、README(EN/JA)の文体統一、`init` 時の install-dir 入力プロンプト追加、`skills-lock.arsync` のスキーマ整合、`skills.arsync.lock`→`skills-lock.arsync` へのリネームなど、リリース直前の仕上げ作業が中心
- 推測: 作業ツリーは全34ファイルで差分 (insertions=deletions=1484) があり、内容は変更なしで改行コード(CRLF/LF)のみが変わっている。WSLとWindowsを跨いだチェックアウト設定(`core.autocrlf`等)の差異によるものと推測され、実質的な未コミット変更ではない
- 推測: npmでのマルチプラットフォームバイナリ配布(`@artesync/win32-x64` 等)を前提とした設計だが、READMEでは「experimental」と明記されており、CLIのオプション体系(`--owner`/`--repository`/`--branch`/`--tag`/`--path`)は現在も調整段階にあると見られる
