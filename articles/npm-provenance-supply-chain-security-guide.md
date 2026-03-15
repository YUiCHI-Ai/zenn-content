---
title: "npm provenance入門：パッケージの出所証明でサプライチェーン攻撃に備える"
emoji: "🔐"
type: "tech"
topics: ["npm", "nodejs", "security", "githubactions", "supplychain"]
published: true
---

## はじめに — サプライチェーン攻撃の脅威

2018年のevent-stream事件では、人気パッケージのメンテナ権限が悪意ある第三者に移譲され、暗号通貨ウォレットを狙うマルウェアが埋め込まれた。2021年のua-parser-js事件では、週間700万ダウンロードを超えるパッケージが乗っ取られ、クリプトマイナーが仕込まれた。

これらの事件に共通するのは、**npmレジストリに公開されたパッケージが、本当にそのリポジトリのコードからビルドされたものなのか検証する手段がなかった**ことだ。

2023年にnpmが導入した **provenance（出所証明）** は、この問題に対する回答だ。パッケージが「どのリポジトリの、どのコミットから、どのCI環境でビルドされたか」を暗号学的に証明し、改ざんを検出可能にする。

この記事では、npm provenanceの仕組みから実際の設定方法、検証手順までを実務で使えるレベルで解説する。

:::message
この記事は「HOW（設定方法・検証方法）」にフォーカスしている。パッケージマネージャの依存解決がなぜ複雑になるのか、サプライチェーン攻撃がどのような構造的問題から生じるのかといったWHY（設計思想）については、本記事の範囲外とする。
:::

## 1. npm provenanceとは

### 基本的な仕組み

npm provenanceは、パッケージの「出所（provenance）」を証明する機能だ。具体的には、以下の情報を暗号署名付きで記録する。

```text
出所証明に含まれる情報:
  ┌─────────────────────────────────────────┐
  │ ソースリポジトリ: github.com/user/repo  │
  │ コミットSHA: a1b2c3d4...                │
  │ ビルド環境: GitHub Actions              │
  │ ワークフロー: .github/workflows/publish  │
  │ ビルドトリガー: push to main            │
  │ 署名: Sigstore (Fulcio + Rekor)         │
  └─────────────────────────────────────────┘
```

### SLSA frameworkとの関係

provenanceは **SLSA（Supply-chain Levels for Software Artifacts）** フレームワークのBuild Level 2に対応する。SLSAはGoogleが提唱したサプライチェーンセキュリティの段階的フレームワークだ。

| SLSAレベル | 要件 | npm provenanceの対応 |
|---|---|---|
| Level 1 | ビルドプロセスの記録 | ✅ ビルド環境・トリガー情報を記録 |
| Level 2 | ホスティングされたビルドサービス | ✅ GitHub Actions等のCI環境を要求 |
| Level 3 | 改ざん防止されたビルド環境 | ❌ 別途slsa-github-generator等が必要 |

### Sigstore連携

npm provenanceは **Sigstore** プロジェクトの技術基盤を使用している。

- **Fulcio**: OIDC tokenに基づく短命証明書を発行するCA（認証局）
- **Rekor**: 署名の透明性ログ（改ざん不可能な公開台帳）

従来のPGP鍵による署名と異なり、開発者が長期的な鍵を管理する必要がない。CI環境が自動的にOIDC tokenを取得し、それに基づいて短命の署名鍵が生成される。

## 2. GitHub Actionsでのprovenance設定

### 基本設定

npm provenanceを有効にするには、GitHub Actionsワークフローに以下の設定を追加する。

```yaml
# .github/workflows/publish.yml
name: Publish to npm
on:
  release:
    types: [published]

permissions:
  contents: read
  id-token: write  # ← OIDC tokenの取得に必須

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          registry-url: 'https://registry.npmjs.org'

      - run: npm ci

      - run: npm publish --provenance --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

重要なポイントは2つだ。

1. **`permissions.id-token: write`**: GitHub ActionsのOIDC tokenを取得する権限。これがないとprovenance署名が作成できない
2. **`--provenance`フラグ**: `npm publish`コマンドに付与する。これでSigstoreへの署名と透明性ログへの記録が行われる

### npm automation tokenの設定

GitHub Actionsからnpmに公開するには、npm automation tokenが必要だ。

```bash
# npmjs.comでtokenを生成する手順:
# 1. npmjs.com → Access Tokens → Generate New Token
# 2. Token type: "Granular Access Token" を選択
# 3. Packages: Read and Write
# 4. 対象パッケージを指定（全パッケージは避ける）
```

生成したtokenをGitHubリポジトリのSecretに登録する。

```bash
# GitHub CLIで設定する場合
gh secret set NPM_TOKEN --body "npm_XXXXXXXXXXXX"
```

### publishの確認

provenance付きでpublishすると、ワークフローログに以下のような出力が表示される。

```text
npm notice Publishing to https://registry.npmjs.org/ with tag latest and public access
npm notice Signed provenance statement with source and build information from GitHub Actions
npm notice Provenance statement published to transparency log: https://search.sigstore.dev/?logIndex=XXXXX
npm notice
+ my-package@1.0.0
```

`Provenance statement published to transparency log` が表示されれば成功だ。

## 3. provenanceの検証方法

### npmjs.comでの確認

provenance付きで公開されたパッケージは、npmjs.comのパッケージページに **Provenance Badge** が表示される。

```text
┌──────────────────────────────────────┐
│  my-package                          │
│  ✅ Provenance                       │
│  Build: GitHub Actions               │
│  Source: github.com/user/my-package  │
│  Commit: a1b2c3d                     │
└──────────────────────────────────────┘
```

バッジをクリックすると、ビルドの詳細情報（リポジトリ、コミット、ワークフローファイル）が確認できる。

### CLI検証: npm audit signatures

```bash
# プロジェクトの全依存パッケージの署名を検証
npm audit signatures
```

このコマンドは、`node_modules`内の全パッケージについて以下を確認する。

1. レジストリ署名の有効性
2. provenance署名の有効性（存在する場合）

```text
# 正常な出力例
audited 1247 packages in 3s
1247 packages have verified registry signatures
89 packages have verified attestations
```

`verified attestations` の数がprovenance付きパッケージの数だ。

### 特定パッケージの確認

```bash
# 特定パッケージのprovenance情報を確認
npm view express --json | python3 -c "
import sys, json
data = json.load(sys.stdin)
dist = data.get('dist', {})
print(f'Integrity: {dist.get(\"integrity\", \"N/A\")}')
print(f'Attestations: {dist.get(\"attestations\", {}).get(\"url\", \"N/A\")}')"
```

## 4. package-lock.jsonのintegrityフィールド

### SHAハッシュの仕組み

`package-lock.json`の各パッケージエントリには`integrity`フィールドがある。

```json
{
  "node_modules/express": {
    "version": "4.21.2",
    "resolved": "https://registry.npmjs.org/express/-/express-4.21.2.tgz",
    "integrity": "sha512-28HqgMZAmih1Czt9ny7qr6ek2qddF4FclbMzwhCREB6OFfH+rXAnuNCwo1/wFvrtbgsQDb4kSbX9de9lFbrXnA=="
  }
}
```

`integrity`はSubresource Integrity（SRI）形式で、`sha512-<base64ハッシュ>` の形式だ。`npm install`時に、ダウンロードしたtarballのハッシュとこの値を比較し、一致しなければインストールを拒否する。

### lockfile改ざん検知

lockfileのintegrityフィールドが改ざんされた場合、`npm ci`は以下のエラーで停止する。

```bash
npm ci
# npm ERR! Verification failed while extracting express@4.21.2:
# npm ERR! Integrity check failed:
# npm ERR!   Wanted: sha512-28HqgMZ...
# npm ERR!   Found:  sha512-XXXXXXX...
```

CIでは`npm install`ではなく`npm ci`を使うべき理由の一つがこれだ。`npm ci`はlockfileを厳密に尊重し、不一致があれば即座に失敗する。

### --frozen-lockfileの重要性

```bash
# npm: lockfileを厳密に使用
npm ci

# pnpm: lockfileを厳密に使用
pnpm install --frozen-lockfile

# yarn: lockfileを厳密に使用
yarn install --immutable
```

いずれのコマンドも、lockfileに記録されたintegrityハッシュとダウンロードしたパッケージのハッシュが一致しない場合、エラーで停止する。

:::message
📖 lockfileのintegrityハッシュがパッケージの安全性を保証できるのは、依存解決の仕組みに基づいている。**なぜ**この仕組みが必要なのか、3つのパッケージマネージャの設計思想の違いから理解したい方は **[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)** を参照してほしい。
:::

## 5. npm auditの活用

### 基本的な使い方

```bash
# 全依存パッケージの脆弱性をチェック
npm audit

# JSON形式で出力（CI向け）
npm audit --json

# 本番依存のみチェック（devDependenciesを除外）
npm audit --omit=dev
```

出力例:

```text
# 脆弱性レポート
┌───────────────┬──────────────────────────────────────┐
│ Moderate      │ Prototype Pollution in lodash         │
├───────────────┼──────────────────────────────────────┤
│ Package       │ lodash                                │
│ Dependency of │ my-lib                                │
│ Path          │ my-lib > lodash                       │
│ More info     │ https://github.com/advisories/GHSA-x  │
└───────────────┴──────────────────────────────────────┘
```

### npm audit fixの使い方

```bash
# 自動修正可能な脆弱性を修正
npm audit fix

# SemVer互換の範囲外の更新も許可（メジャーバージョンアップを含む）
npm audit fix --force
```

`--force`は依存関係の互換性を壊す可能性があるため、CIではなく手動で実行すべきだ。実行後は必ずテストを通す。

### CI/CDでのaudit統合

```yaml
# .github/workflows/ci.yml
- name: Security audit
  run: npm audit --audit-level=high --omit=dev
```

`--audit-level`で閾値を設定する。

| audit-level | 動作 |
|---|---|
| `info` | 全ての脆弱性でexit code 1 |
| `low` | low以上の脆弱性でexit code 1 |
| `moderate` | moderate以上でexit code 1 |
| `high` | high以上でexit code 1（推奨） |
| `critical` | criticalのみでexit code 1 |

実務では`high`を推奨する。`moderate`以下はfalse positiveが多く、CIが頻繁に失敗してノイズになる。

### 脆弱性SLA設定

チームで脆弱性対応のSLAを定めておくとよい。

```text
脆弱性対応SLA（推奨）:
  critical: 24時間以内に対応開始
  high:     1週間以内に対応完了
  moderate: 次回スプリントで検討
  low:      バックログに記録
```

## 6. Socket / Snykとの連携

### Socket

**Socket**（socket.dev）はnpmパッケージのサプライチェーンリスクを検出するサービスだ。npm auditが既知の脆弱性（CVE）のみを検出するのに対し、Socketは以下のリスクも検出する。

- 新バージョンでの不審なコード変更（ネットワークアクセス追加、環境変数読み取り等）
- メンテナ変更による乗っ取りリスク
- typosquatting（名前の似たパッケージ）

```bash
# Socket CLIのインストール
npm install -g @socketsecurity/cli

# プロジェクトのリスクスキャン
socket scan .
```

GitHub Appとして導入すると、PRに自動でリスク分析コメントが付く。

### Snyk

**Snyk**はより広範なセキュリティプラットフォームで、npm以外のエコシステム（Python、Java、Go等）もカバーする。

```bash
# Snyk CLIのインストール
npm install -g snyk

# 認証
snyk auth

# 脆弱性スキャン
snyk test

# 継続的モニタリングに登録
snyk monitor
```

GitHub Actions統合:

```yaml
- name: Snyk Security Check
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

### ツール比較

| 観点 | npm audit | Socket | Snyk |
|---|---|---|---|
| 価格 | 無料 | 無料プラン有 | 無料プラン有 |
| 検出対象 | 既知CVE | 行動分析+CVE | CVE+ライセンス |
| CI統合 | 組み込み | GitHub App | CLI/GitHub App |
| 自動修正PR | ❌ | ❌ | ✅ |
| 推奨用途 | 基本チェック | サプライチェーン監視 | エンタープライズ |

最低限のセキュリティ対策としては`npm audit`で十分だ。サプライチェーン攻撃への深い防御を求める場合にSocketやSnykを追加する。

## 7. pnpm / yarnでのセキュリティ対応

### pnpm audit

pnpmにも`audit`コマンドがある。内部的にnpmレジストリの脆弱性データベースを参照する。

```bash
# 脆弱性チェック
pnpm audit

# 本番依存のみ
pnpm audit --prod

# JSON出力
pnpm audit --json
```

pnpmの`--frozen-lockfile`は`npm ci`と同様の厳密モードだ。

```bash
# CIでの推奨コマンド
pnpm install --frozen-lockfile
```

### pnpmのセキュリティ上の利点

pnpmは設計上、いくつかのセキュリティ上の利点を持っている。

1. **Content-Addressable Store**: パッケージはハッシュベースで保存されるため、同じハッシュのパッケージが改ざんされていれば検出できる
2. **厳密なnode_modules構造**: フラットではないためPhantom dependency（宣言していない依存への暗黙アクセス）を防止する
3. **postinstallスクリプトの制御**: pnpm 10以降、`onlyBuiltDependencies`で明示的に許可したパッケージのみpostinstallが実行される

```json
{
  "pnpm": {
    "onlyBuiltDependencies": ["sharp", "esbuild"]
  }
}
```

### yarn audit

```bash
# 脆弱性チェック（Yarn Classic / Berry共通）
yarn audit

# Yarn Berry（v2以降）の場合
yarn npm audit

# 重要度フィルタ
yarn npm audit --severity high
```

Yarn Berryの`--immutable`は`npm ci`相当だ。

```bash
# CIでの推奨コマンド
yarn install --immutable
```

### Bun環境での注意点

Bun v1.2.15以降では`bun audit`コマンドが利用可能である。npmと同じレジストリAPIを使用して脆弱性を検出する。

```bash
# 脆弱性チェック
bun audit

# 重要度フィルタ
bun audit --audit-level=high

# 本番依存のみ
bun audit --omit=dev
```

v1.2.15より前のBunを使用している場合は、`npx npm@latest audit`（`package-lock.json`の生成が必要）またはSnyk/Socket CLIで代替する。

## 8. provenance対応のpublishワークフロー実践

### monorepoでのprovenance

monorepo（Turborepo、Nx、Lerna等）でprovenanceを使う場合、各パッケージのpublishで個別にprovenance署名が付与される。

```yaml
# monorepoでのprovenance付きpublish
name: Publish Packages
on:
  release:
    types: [published]

permissions:
  contents: read
  id-token: write

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          registry-url: 'https://registry.npmjs.org'

      - run: pnpm install --frozen-lockfile

      # Changesetsを使っている場合
      - run: pnpm changeset publish --provenance
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### 段階的なセキュリティ強化

プロジェクトの規模に応じた段階的なセキュリティ対策を推奨する。

```text
Level 1（最低限）:
  ├── package-lock.json をコミット
  ├── CIで npm ci (--frozen-lockfile) を使用
  └── npm audit --audit-level=high をCIに統合

Level 2（推奨）:
  ├── Level 1 の全項目
  ├── npm publish --provenance で出所証明を付与
  ├── Dependabot / Renovate で自動更新PR
  └── 脆弱性SLAの策定

Level 3（エンタープライズ）:
  ├── Level 2 の全項目
  ├── Socket or Snyk によるサプライチェーン監視
  ├── npm audit signatures をCIに統合
  └── プライベートレジストリ（Verdaccio等）でのプロキシキャッシュ
```

## 9. トラブルシューティング

### provenanceが付与されない

**症状**: `npm publish --provenance`を実行しても署名が付かない

```text
npm ERR! Provenance generation in a publish command is only supported on GitHub Actions
```

**原因と対策**:

1. **CI環境の問題**: provenanceは現在GitHub ActionsとGitLab CI/CDで対応している。ローカル環境からは使用できない
2. **permissions不足**: `id-token: write`がワークフローに設定されていない
3. **npmバージョンが古い**: npm 9.5.0以降が必要

```yaml
# 確認チェックリスト
permissions:
  id-token: write    # ← 必須
  contents: read
```

### npm audit signaturesが失敗する

```bash
npm audit signatures
# npm ERR! audit signatures Verification failed for express@4.21.2
```

**原因**: レジストリのキーがローテーションされた可能性がある。以下を試す。

```bash
# npmのキャッシュをクリア
npm cache clean --force

# レジストリ設定を確認
npm config get registry
# → https://registry.npmjs.org/ であること

# 再度実行
npm audit signatures
```

### GitHub Actions OIDCエラー

```text
Error: Unable to get OIDC token
```

**原因**: ワークフローのpermissions設定が不正。

```yaml
# 修正: トップレベルまたはジョブレベルでpermissionsを設定
jobs:
  publish:
    permissions:
      contents: read
      id-token: write  # ← ジョブレベルでも指定可能
```

また、GitHub Environmentを使用している場合は、Environment側のprotection rulesも確認する。

### npm auditのfalse positive対応

特定の脆弱性がプロジェクトに影響しないことが確認できた場合、`package.json`のoverridesで対応する。

```json
{
  "overrides": {
    "vulnerable-package": "2.0.1"
  }
}
```

pnpmの場合は`pnpm.overrides`を使う。

```json
{
  "pnpm": {
    "overrides": {
      "vulnerable-package": "2.0.1"
    }
  }
}
```

## 10. まとめ — セキュリティチェックリスト

### publish前チェックリスト

- [ ] `npm publish --provenance`でprovenanceを付与しているか
- [ ] GitHub Actionsの`permissions.id-token: write`が設定されているか
- [ ] npm automation tokenが適切なスコープで設定されているか
- [ ] `package.json`の`files`フィールドで公開ファイルを限定しているか
- [ ] `.npmignore`で機密ファイルを除外しているか

### install前チェックリスト（依存追加時）

- [ ] `npm audit`で既知の脆弱性がないか確認したか
- [ ] npmjs.comでパッケージのProvenance Badgeを確認したか
- [ ] パッケージのダウンロード数・メンテナ情報を確認したか
- [ ] 不要な依存を追加していないか（`npx depcheck`で確認）

### CI/CDチェックリスト

- [ ] `npm ci`（または`--frozen-lockfile`）を使用しているか
- [ ] `npm audit --audit-level=high --omit=dev`をCIに組み込んでいるか
- [ ] Dependabot / Renovateによる自動更新が設定されているか
- [ ] 脆弱性対応SLAがチームで合意されているか

サプライチェーンセキュリティは一度設定して終わりではなく、継続的な監視と更新が必要だ。最低限のレベル（lockfile固定 + audit + frozen-lockfile）から始め、プロジェクトの成長に応じて段階的に強化していくことを推奨する。

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説している。
:::

---

:::message
この記事はAI（Claude）による生成を含む。技術的な正確性については可能な限り検証しているが、最新情報は公式ドキュメントを確認してほしい。
:::
