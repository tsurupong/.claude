# npm 公開の後始末プラン

## Context
@tsurupong/halo / halo-core / halo-contracts v0.1.0 の npm 公開は完了済み（レジストリで `dist-tags.latest: 0.1.0` を確認）。
ただしローカルリポジトリには公開時のリネーム変更（`@halo/*` → `@tsurupong/*`、バージョン 0.1.0 統一、publish メタデータ追加、ruleset JSON）が**未コミット**のまま残っている。これを確定させ、公開物が実際に使えることを検証し、漏洩したトークンを無効化する。

## 手順

1. **公開後スモークテスト**（読み取りのみ）
   - クリーンな scratchpad ディレクトリで `pnpm --package=@tsurupong/halo dlx halo --version`（または help）を実行し、レジストリからのインストール→CLI 起動を「動くのを見た」状態にする。

2. **コミット & push**（product/ リポジトリ、main へ）
   - 対象: `package.json`×4、`pnpm-lock.yaml`、`packages/*/src` の import 変更、`.github/rulesets/main-protection.json`
   - コミット例: `feat: publish packages to npm as @tsurupong scope (v0.1.0)`
   - push 後、CI 3ジョブ（unit / loop-regression / contract）が green になるのを確認。
   - 注意: main-protection ruleset を既に有効化している場合は直 push がブロックされるため、ブランチ + PR 経由に切り替える。

3. **README 更新（軽微）**
   - Setup セクションに `npm i -D @tsurupong/halo` によるインストール手順を追記（同一コミットに含める）。

4. **トークン失効の案内**（ユーザー操作）
   - 2つのトークン（1つ目はチャットに平文で貼られた）を npmjs.com → Access Tokens から Delete するよう依頼。`~/.npmrc` の当該行の削除もユーザー承認後に実施。

## 検証
- スモークテスト: 公開版 CLI がコマンドとして起動する出力を確認
- push 後: GitHub API で CI conclusion=success を確認
