---
title: "Changesets入門：モノレポのバージョニングとCHANGELOG自動管理の実践ガイド"
emoji: "🦋"
type: "tech"
topics: ["changesets", "monorepo", "npm", "pnpm", "githubactions"]
published: true
---

## 「バージョンいくつにする？」を仕組みで解決する

モノレポで複数パッケージを管理していると、リリースのたびに次の作業が発生する。

1. どのパッケージが変更されたか確認する
2. SemVerのmajor/minor/patchどれを上げるか判断する
3. CHANGELOG.mdを手書きする
4. npm publishを実行する
5. 内部依存パッケージのバージョンも連動して更新する

1パッケージなら手動で回せるが、5、10とパッケージが増えるにつれて人手では破綻する。この問題を「changesetファイル」という仕組みで解決するのが**Changesets**だ。

本記事では、Changesetsのセットアップから日常のワークフロー、GitHub Actionsによる自動リリースまで、実務で必要な手順を一通り解説する。

## Changesetsとは何か

[Changesets](https://github.com/changesets/changesets)は、Atlassian発のバージョニング・CHANGELOG管理ツールだ。npmパッケージとしては`@changesets/cli`で提供されており、2026年3月時点の最新安定版は**2.30.0**だ。

核となるアイデアはシンプルだ。**コード変更とは別に「何が変わったか」をMarkdownファイルとして記録し、リリース時にそのファイルを消費してバージョンとCHANGELOGを自動更新する**。

### 設計思想

- **モノレポファースト** -- 複数パッケージ間の依存関係を自動追跡する
- **人間が意図を決める** -- バージョンの上げ幅（major/minor/patch）は開発者が明示的に選ぶ
- **コミットメッセージに依存しない** -- Conventional Commitsの規約なしで運用できる
- **段階的なリリース** -- changesetを蓄積し、まとめてリリースできる

### どんなプロジェクトに向いているか

| 特性 | Changesetsとの相性 |
|------|:---:|
| モノレポで複数パッケージをnpm公開 | ◎ |
| 単体パッケージのnpm公開 | ○ |
| チームでPRベースの開発 | ◎ |
| リリースタイミングを人間が制御したい | ◎ |
| 完全自動リリース（コミット即publish） | △（semantic-releaseが向く） |

## セットアップ：npx changeset init

### 前提

- Node.js 14以上
- プロジェクトがnpm/pnpm/yarn workspacesでセットアップ済み（単体パッケージでも利用可）

### インストール

```bash
# プロジェクトのルートに開発依存としてインストール
npm install -D @changesets/cli

# pnpmの場合
pnpm add -D @changesets/cli -w

# yarnの場合
yarn add -D @changesets/cli -W
```

### 初期化

```bash
npx changeset init
```

これで`.changeset/`ディレクトリが作成される。

```
.changeset/
├── config.json    # Changesetsの設定ファイル
└── README.md      # .changesetディレクトリの説明
```

この2ファイルはGitにコミットする。`.changeset/`を`.gitignore`に入れてはいけない。

### config.jsonの設定項目

`npx changeset init`で生成されるデフォルト設定は以下の通りだ。

```json
{
  "$schema": "https://unpkg.com/@changesets/config@3.1.1/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "fixed": [],
  "linked": [],
  "access": "restricted",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": []
}
```

主要な設定項目を解説する。

| 項目 | デフォルト値 | 説明 |
|------|------|------|
| `changelog` | `"@changesets/cli/changelog"` | CHANGELOG生成ロジック。`"@changesets/changelog-github"`に変更するとPRリンクやコミットハッシュが付く |
| `commit` | `false` | `changeset version`実行時にgit commitを自動で行うか |
| `access` | `"restricted"` | npm公開時のアクセス権。公開パッケージは`"public"`に変更が必要 |
| `baseBranch` | `"main"` | 比較対象のベースブランチ |
| `fixed` | `[]` | 常に同じバージョンにするパッケージ群（例: `[["@myorg/core", "@myorg/types"]]`） |
| `linked` | `[]` | バージョンを連動させるパッケージ群（fixedと違い、変更がないパッケージは更新しない） |
| `updateInternalDependencies` | `"patch"` | 依存先パッケージが更新されたとき、依存元のバージョンをどの単位で上げるか |
| `ignore` | `[]` | Changesetsの管理対象外にするパッケージ |

## 日常ワークフロー：add → version → publish

Changesetsの操作は3つのコマンドに集約される。

### 1. changeset add：変更内容を記録する

機能の実装やバグ修正をコミットした後、次のコマンドを実行する。

```bash
npx changeset
# または
npx changeset add
```

対話式のウィザードが起動し、以下を順に聞かれる。

```
🦋  Which packages would you like to include?
   ◯ @myorg/ui
   ◯ @myorg/utils
   ◯ @myorg/api-client

🦋  Which packages should have a major bump?
🦋  Which packages should have a minor bump?

🦋  Please enter a summary for this change (this will be in the changelog):
   > Add dark mode support to Button component
```

選択が完了すると、`.changeset/`配下にランダムな名前のMarkdownファイルが生成される。

```
.changeset/
├── config.json
├── README.md
└── brave-dogs-dance.md    ← 今回生成されたファイル
```

生成されたファイルの中身はこうなる。

```markdown
---
"@myorg/ui": minor
---

Add dark mode support to Button component
```

frontmatterにパッケージ名とバージョン種別（major/minor/patch）、本文に変更の説明が記述される。

**このファイルをコミットしてPRに含める**。これがChangesetsの最も重要なポイントだ。コードの変更と「何がどう変わったか」の意図が、同じPRに同居する。

### 複数パッケージへの影響

1つのchangesetで複数パッケージの変更を記録できる。

```markdown
---
"@myorg/ui": minor
"@myorg/utils": patch
---

Add dark mode support to Button component, with color utility helpers
```

### 2. changeset version：バージョンとCHANGELOGを更新する

蓄積されたchangesetファイルを消費して、実際にバージョンを上げる。

```bash
npx changeset version
```

このコマンドは以下を自動で行う。

1. `.changeset/`配下のMarkdownファイルを読み取る
2. 各パッケージの`package.json`の`version`フィールドを更新する
3. 各パッケージの`CHANGELOG.md`にエントリを追記する
4. 内部依存パッケージのバージョン参照も更新する
5. 消費したchangesetファイルを削除する

### 3. changeset publish：npmに公開する

```bash
npx changeset publish
```

バージョンが更新されたパッケージをnpmレジストリに公開する。内部的には`npm publish`（または`pnpm publish`、`yarn npm publish`）を各パッケージで実行する。

公開後、gitタグも自動で付与される（例: `@myorg/ui@2.1.0`）。

### ワークフローの全体像

```
開発者A: feature実装 → npx changeset（changesetファイル作成）→ commit → PR
開発者B: bugfix → npx changeset → commit → PR

  ↓ PRマージ後 ↓

リリース担当: npx changeset version → 差分確認 → commit → push
           → npx changeset publish → npmに公開
```

## SemVerとの関係

Changesetsは[Semantic Versioning（SemVer）](https://semver.org/)に従ってバージョンを管理する。changesetファイル作成時に開発者が選択するmajor/minor/patchは、SemVerの定義に直接対応する。

| 種別 | SemVerの意味 | 選ぶ場面 |
|------|------|------|
| **major** | 後方互換性のない変更 | APIの削除、引数の型変更、振る舞いの破壊的変更 |
| **minor** | 後方互換性のある機能追加 | 新しいAPIの追加、オプショナルな引数の追加 |
| **patch** | バグ修正 | 既存機能の修正、ドキュメント修正、パフォーマンス改善 |

### 複数のchangesetがある場合の挙動

同じパッケージに対して複数のchangesetが蓄積された場合、**最も大きいバージョン種別が採用される**。

```
changeset A: @myorg/ui → patch
changeset B: @myorg/ui → minor
changeset C: @myorg/ui → patch

→ 結果: @myorg/ui は minor が適用される（1.2.0 → 1.3.0）
```

CHANGELOG.mdには3つすべての変更内容が記録される。

## モノレポでの複数パッケージ管理

Changesetsが最も力を発揮するのはモノレポだ。以下の構造を例に解説する。

```
my-monorepo/
├── packages/
│   ├── core/         # @myorg/core
│   ├── ui/           # @myorg/ui（coreに依存）
│   └── api-client/   # @myorg/api-client（coreに依存）
├── package.json
└── pnpm-workspace.yaml
```

### 内部依存の自動更新

`@myorg/core`をpatchで更新すると、`@myorg/core`に依存する`@myorg/ui`と`@myorg/api-client`の`package.json`内の依存バージョンも自動更新される。この挙動は`config.json`の`updateInternalDependencies`で制御できる。

```json
{
  "updateInternalDependencies": "patch"
}
```

- `"patch"` -- 依存先がpatch以上で更新された場合に、依存元のpatch版も上げる
- `"minor"` -- 依存先がminor以上で更新された場合のみ連動する

### fixedグループ

常に同じバージョン番号を維持したいパッケージ群を定義する。

```json
{
  "fixed": [["@myorg/core", "@myorg/types"]]
}
```

いずれかのパッケージが更新されると、グループ全体が同じバージョンに揃う。React本体とReact DOMのような関係で使う。

### linkedグループ

バージョンの「最大値」を共有するが、変更がないパッケージはバージョンを上げない。

```json
{
  "linked": [["@myorg/ui", "@myorg/theme"]]
}
```

fixedと違い、変更されたパッケージのみバージョンが上がる。ただし、同じlinkedグループ内では「最も高いバージョン」に合わせる形で更新される。

## GitHub Actions + changesets/action による自動リリース

手動で`changeset version`と`changeset publish`を実行する運用でも十分機能するが、GitHub Actionsで自動化すると運用負荷が大幅に下がる。

### 公式アクション：changesets/action

[changesets/action](https://github.com/changesets/action)は、Changesetsの公式GitHub Actionだ。以下の2つの動作を自動で行う。

1. **changesetファイルが存在する場合** -- `changeset version`を実行し、バージョン更新のPRを自動作成する
2. **changesetファイルが存在しない場合** -- `changeset publish`を実行し、npmにパッケージを公開する

### ワークフロー設定例

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches:
      - main

concurrency: ${{ github.workflow }}-${{ github.ref }}

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Create Release Pull Request or Publish
        id: changesets
        uses: changesets/action@v1
        with:
          publish: npx changeset publish
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### 動作フロー

```
1. 開発者がchangesetファイル付きPRをmainにマージ
   ↓
2. GitHub Actionsが起動
   ↓
3. changesetファイルが存在する → "Version Packages" PRを自動作成/更新
   ↓
4. チームが "Version Packages" PRをレビュー・マージ
   ↓
5. GitHub Actionsが再度起動
   ↓
6. changesetファイルがない → npm publishが実行される
   ↓
7. GitHubリリースが作成される（オプション）
```

### pnpmを使う場合のワークフロー

pnpm workspacesを使っている場合は、セットアップ部分を以下のように変更する。

```yaml
      - name: Install pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Create Release Pull Request or Publish
        uses: changesets/action@v1
        with:
          publish: pnpm changeset publish
          version: pnpm changeset version
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### NPM_TOKENの設定

npm公開には認証トークンが必要だ。以下の手順で設定する。

1. [npmjs.com](https://www.npmjs.com/) → Access Tokens → Generate New Token → Automation
2. GitHub リポジトリ → Settings → Secrets and variables → Actions → New repository secret
3. Name: `NPM_TOKEN`、Value: 生成したトークン

### changelog-githubプラグイン

CHANGELOGにPRリンクやcontributor情報を含めたい場合は、`@changesets/changelog-github`を使う。

```bash
npm install -D @changesets/changelog-github
```

```json
// .changeset/config.json
{
  "changelog": ["@changesets/changelog-github", { "repo": "your-org/your-repo" }]
}
```

これにより、生成されるCHANGELOGにGitHubのPRリンクが付く。

## CHANGELOG.mdの自動生成フォーマット

`changeset version`で生成されるCHANGELOG.mdのフォーマットは、使用するchangelogプラグインによって異なる。

### デフォルト（@changesets/cli/changelog）

```markdown
# @myorg/ui

## 2.1.0

### Minor Changes

- Add dark mode support to Button component

### Patch Changes

- Fix focus ring not appearing on keyboard navigation
```

### @changesets/changelog-github

```markdown
# @myorg/ui

## 2.1.0

### Minor Changes

- abc1234: Add dark mode support to Button component
  (PR: #42 by @developer-a)

### Patch Changes

- def5678: Fix focus ring not appearing on keyboard navigation
  (PR: #45 by @developer-b)

- Updated dependencies
  - @myorg/core@1.5.1
```

### カスタムchangelogフォーマット

独自のフォーマットが必要な場合、`getReleaseLine`と`getDependencyReleaseLine`の2つの関数を実装したモジュールを作成し、`config.json`の`changelog`に指定する。

```javascript
// my-changelog-config.js
async function getReleaseLine(changeset, type, options) {
  const summary = changeset.summary;
  return `- ${summary}`;
}

async function getDependencyReleaseLine(changesets, dependenciesUpdated, options) {
  if (dependenciesUpdated.length === 0) return "";
  const updates = dependenciesUpdated.map(
    (dep) => `  - ${dep.name}@${dep.newVersion}`
  );
  return ["- Updated dependencies:", ...updates].join("\n");
}

module.exports = {
  getReleaseLine,
  getDependencyReleaseLine,
};
```

```json
{
  "changelog": "./my-changelog-config.js"
}
```

## semantic-releaseとの比較

Changesetsと並んでよく比較されるのが[semantic-release](https://github.com/semantic-release/semantic-release)だ。どちらもバージョニングとリリースを自動化するツールだが、設計思想が異なる。

| 項目 | Changesets | semantic-release |
|------|------|------|
| バージョン決定方法 | 開発者がchangesetファイルで明示的に指定 | コミットメッセージ（Conventional Commits）から自動判定 |
| モノレポ対応 | ネイティブ対応 | プラグイン（semantic-release-monorepo）が必要 |
| リリースタイミング | 蓄積→手動でversion/publish | mainへのpush時に即座にリリース |
| コミットメッセージ規約 | 不要 | 必須（Conventional Commits） |
| 学習コスト | 低（コマンド3つ） | 中（プラグインシステムの理解が必要） |
| 内部依存の自動更新 | あり | なし（個別に設定が必要） |
| CHANGELOG生成 | 組み込み | プラグインで対応 |
| GitHub PR連携 | Version Packages PR自動作成 | なし（commit起点で即リリース） |

### 使い分けの指針

**Changesetsが向いているケース：**
- モノレポで複数パッケージを管理している
- リリースのタイミングをチームで制御したい
- コミットメッセージの規約を強制したくない
- PRベースでリリース内容をレビューしたい

**semantic-releaseが向いているケース：**
- 単体パッケージのリリースを完全自動化したい
- Conventional Commitsを既にチームで運用している
- mainへのマージ即リリースのフローが合っている

両ツールは「バージョン決定を人間が行うか、コミットメッセージから機械が行うか」という点で根本的に異なる。プロジェクトのリリースフローに合わせて選択すればよい。

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説しています。
:::

## パッケージマネージャ別ワークスペース連携

Changesetsは主要なパッケージマネージャのworkspaces機能と連携して動作する。各パッケージマネージャでの設定を見ていく。

### pnpm workspaces

pnpmとの連携は公式ドキュメントでも推奨されている組み合わせだ。

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
  - "apps/*"
```

```bash
# ルートにChangesetsをインストール
pnpm add -D @changesets/cli -w

# 初期化
pnpm changeset init

# changeset作成
pnpm changeset

# バージョン更新（lockfileも更新する）
pnpm changeset version
pnpm install  # lockfileの更新を反映

# 公開
pnpm changeset publish
```

pnpmの場合、`changeset version`後に`pnpm install`を実行してlockfileを更新する必要がある。GitHub Actionsで自動化する場合は、`version`コマンドをカスタマイズする。

```json
// package.json
{
  "scripts": {
    "changeset:version": "changeset version && pnpm install --lockfile-only"
  }
}
```

```yaml
# GitHub Actions
- uses: changesets/action@v1
  with:
    version: pnpm run changeset:version
    publish: pnpm changeset publish
```

### npm workspaces

npm 7以降のworkspaces機能と組み合わせて使える。

```json
// package.json（ルート）
{
  "workspaces": [
    "packages/*",
    "apps/*"
  ]
}
```

```bash
# インストール
npm install -D @changesets/cli

# 初期化・操作
npx changeset init
npx changeset
npx changeset version
npm install  # lockfileの更新
npx changeset publish
```

npm workspacesでも`changeset version`後に`npm install`でlockfileを更新する。

### yarn workspaces（v3/v4 Berry）

```json
// package.json（ルート）
{
  "workspaces": [
    "packages/*"
  ]
}
```

```bash
# インストール
yarn add -D @changesets/cli -W

# 初期化・操作
yarn changeset init
yarn changeset
yarn changeset version
yarn install  # lockfileの更新
yarn changeset publish
```

yarn BerryのPlug'n'Play（PnP）モードでも動作するが、`nodeLinker: node-modules`設定の方が互換性の問題が少ない。

### ワークスペース連携のまとめ

| パッケージマネージャ | ワークスペース定義 | lockfile更新 | 注意点 |
|------|------|------|------|
| pnpm | `pnpm-workspace.yaml` | `pnpm install --lockfile-only` | 最も安定した連携 |
| npm | `package.json`の`workspaces` | `npm install` | npm 7以降が必要 |
| yarn Berry | `package.json`の`workspaces` | `yarn install` | PnPモードでは互換性確認が必要 |

## 実践Tips

### preリリース

Changesetsはpre-release（alpha、beta、rc）もサポートしている。

```bash
# preリリースモードに入る
npx changeset pre enter beta

# 通常通りchangesetを作成してversion
npx changeset
npx changeset version
# → 1.2.0-beta.0 のようなバージョンが生成される

npx changeset publish --tag beta

# preリリースモードを終了
npx changeset pre exit
```

### snapshotリリース

PRの段階でテスト用バージョンを公開したい場合に使う。

```bash
npx changeset version --snapshot preview
# → 0.0.0-preview-20260315120000 のようなバージョンが生成される

npx changeset publish --tag preview
```

### package.jsonのscripts設定

日常的に使うコマンドはscriptsにまとめておくと便利だ。

```json
{
  "scripts": {
    "changeset": "changeset",
    "version-packages": "changeset version",
    "publish-packages": "changeset publish",
    "release": "changeset version && changeset publish"
  }
}
```

### changeset statusで確認

現在蓄積されているchangesetの状態を確認できる。

```bash
npx changeset status
```

未消費のchangesetファイルの一覧と、それぞれがどのパッケージにどのバージョン変更を適用するかが表示される。CIでchangesetの存在チェックにも使える。

```yaml
# PRにchangesetが含まれているか確認
- name: Check for changesets
  run: npx changeset status --since=origin/main
```

### privateパッケージの扱い

`package.json`に`"private": true`が設定されたパッケージは、`changeset publish`でnpmには公開されない。ただし、バージョンの更新とCHANGELOGの生成は行われる。モノレポ内のアプリケーション（apps/web等）のバージョン管理にも使える。

## まとめ

Changesetsの導入手順を振り返る。

1. `npm install -D @changesets/cli && npx changeset init` でセットアップ
2. 機能実装やバグ修正ごとに `npx changeset` でchangesetファイルを作成
3. リリース時に `npx changeset version` でバージョンとCHANGELOGを更新
4. `npx changeset publish` でnpmに公開
5. GitHub Actionsで `changesets/action@v1` を使えば、Version Packages PRの自動作成からnpm publishまで自動化できる

コミットメッセージの規約を導入する前に、まずChangesets単体で始めるのが最もスムーズだ。モノレポでパッケージ数が増えてきたとき、このツールがリリース運用の負荷を大きく下げてくれる。

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説しています。
:::

## 参考リンク

- [Changesets公式リポジトリ](https://github.com/changesets/changesets)
- [Changesets公式ドキュメント](https://changesets-docs.vercel.app/)
- [changesets/action（GitHub Actions）](https://github.com/changesets/action)
- [Using Changesets with pnpm（pnpm公式）](https://pnpm.io/using-changesets)
- [config-file-options.md（設定項目の詳細）](https://github.com/changesets/changesets/blob/main/docs/config-file-options.md)
