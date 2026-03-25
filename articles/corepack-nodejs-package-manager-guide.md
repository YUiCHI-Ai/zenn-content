---
title: "corepack入門：Node.js標準のパッケージマネージャ管理ツールを完全理解する"
emoji: "📦"
type: "tech"
topics: ["nodejs", "corepack", "pnpm", "yarn", "npm"]
published: true
---

## はじめに ── 「チームで使うパッケージマネージャのバージョンが揃わない」問題

「自分の環境ではpnpm 9で動いていたのに、同僚のpnpm 10では`pnpm install`が通らない」

「CIのyarnバージョンとローカルが違って、lockfileが毎回変わる」

Node.jsでチーム開発をしていると、パッケージマネージャのバージョン不一致によるトラブルに一度は遭遇する。この問題を解決するのが **Corepack** だ。

Corepackは、Node.js 14.19 / 16.9以降に同梱されている「パッケージマネージャのバージョンを管理するツール」だ。`package.json`にパッケージマネージャとそのバージョンを宣言しておくと、プロジェクトに入ったメンバー全員が自動的に同じバージョンを使えるようになる。

この記事では、Corepackの基本的な使い方からCI/CDでの活用、トラブルシューティングまで、**実務で必要な知識をすべてカバーする**。

:::message
この記事は「HOW（使い方・設定方法）」にフォーカスしている。「なぜNode.jsはnpmだけを同梱してきたのか」「Corepack非バンドル化の技術的背景は何か」といったWHY（設計思想・歴史的経緯）については、**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**で詳しく解説している。
:::

## 1. Corepackとは何か

### 基本コンセプト

Corepackは一言でいえば「**パッケージマネージャのマネージャ**」だ。

Node.jsにはnpmが同梱されているが、yarn や pnpm を使いたい場合は別途インストールが必要になる。さらに、プロジェクトごとに「pnpm 9.x を使う」「yarn 4.x を使う」と指定バージョンが異なるケースもある。Corepackはこのバージョン管理を自動化する。

```
従来の問題:
  開発者A: pnpm 9.15.0 → pnpm install ✅
  開発者B: pnpm 10.2.0 → pnpm install ❌ (lockfile差分発生)

Corepackによる解決:
  package.json: "packageManager": "pnpm@9.15.0"
  開発者A: pnpm install → Corepackが9.15.0を自動取得 ✅
  開発者B: pnpm install → Corepackが9.15.0を自動取得 ✅
```

### 対応バージョン

Corepackは以下のNode.jsバージョンに同梱されている。

| Node.js バージョン | Corepack の状態 |
|---|---|
| 14.19以降 / 16.9以降 | 実験的（experimental）として同梱 |
| 18.x / 20.x / 22.x | 実験的として同梱（LTS） |
| 24.x | 同梱（Active LTS） |
| 25以降 | **非バンドル化**（後述） |

Corepackが管理できるパッケージマネージャは以下の2つだ。

- **yarn**（Classic 1.x / Berry 2.x〜4.x）
- **pnpm**（6.x〜10.x）

npmはNode.jsに直接同梱されているため、Corepackの管理対象外だ。

### 現在のステータス

Corepackは2021年の導入以来、一貫して **experimental（実験的）** ステータスのままだ。Node.jsの起動時に `--experimental-corepack` フラグは不要だが、明示的に有効化する手順が必要になる（次章で解説）。

## 2. corepack enable ── Corepackを有効化する

### 基本的な有効化

Corepackは同梱されているが、デフォルトでは無効になっている。使い始めるには `corepack enable` を実行する。

```bash
# Corepackを有効化する
corepack enable
```

このコマンドは、Node.jsのインストールディレクトリに `yarn` と `pnpm` のシム（shim）を作成する。シムとは、コマンド実行時にCorepackを経由させるための小さなラッパースクリプトだ。

```bash
# 有効化後の確認
which pnpm
# → /usr/local/bin/pnpm（Corepackのシム）

which yarn
# → /usr/local/bin/yarn（Corepackのシム）
```

### 特定のパッケージマネージャのみ有効化

yarn だけ、または pnpm だけを有効化することもできる。

```bash
# pnpmのみ有効化
corepack enable pnpm

# yarnのみ有効化
corepack enable yarn
```

### 無効化

Corepackを無効化してシムを削除するには `corepack disable` を使う。

```bash
# Corepackを無効化
corepack disable

# 特定のパッケージマネージャのみ無効化
corepack disable pnpm
```

### 権限エラーへの対処

`corepack enable` でパーミッションエラーが出る場合がある。

```bash
# エラー例
$ corepack enable
Error: EACCES: permission denied, symlink '...'
```

この場合、Node.jsのインストール方法に応じて対処が異なる。

```bash
# 方法1: sudoで実行（システムインストールのNode.jsの場合）
sudo corepack enable

# 方法2: nvm/fnm/voltaを使っている場合はsudo不要のはず
# Node.jsのインストールパスを確認
which node
# → ~/.nvm/versions/node/v22.x.x/bin/node なら権限問題は起きない

# 方法3: npmのグローバルディレクトリを変更している場合
# npmのprefixを確認
npm config get prefix
```

nvmやfnmでNode.jsを管理している場合は、ユーザーディレクトリ配下にインストールされるため、通常は権限エラーは発生しない。

## 3. corepack prepare ── バージョンをローカルキャッシュに事前取得する

### 基本的な使い方

`corepack prepare` は、指定したパッケージマネージャのバージョンをローカルにダウンロード（キャッシュ）するコマンドだ。

```bash
# pnpm 9.15.0 をキャッシュに取得
corepack prepare pnpm@9.15.0 --activate

# yarn 4.6.0 をキャッシュに取得
corepack prepare yarn@4.6.0 --activate
```

`--activate` フラグを付けると、ダウンロードしたバージョンをデフォルトとして設定する。`package.json` に `packageManager` フィールドがないプロジェクトでそのパッケージマネージャを実行したとき、このバージョンが使われる。

### バージョン指定のベストプラクティス

バージョンは必ず明示的に指定する。`@latest` の使用は避ける。

```bash
# 良い例: バージョンを明示
corepack prepare pnpm@9.15.0 --activate

# 避けるべき例: @latestは実行タイミングで結果が変わる
corepack prepare pnpm@latest --activate
```

`@latest` を使うと実行時点の最新版が取得されるため、チームメンバー間やCI実行間で異なるバージョンが使われる可能性がある。Corepackの目的である「バージョン統一」が崩れてしまう。

### オフライン環境での活用

`corepack prepare` はオフライン環境での利用にも役立つ。事前にキャッシュしておけば、ネットワーク接続なしでパッケージマネージャを利用できる。

```bash
# オンライン環境で事前にキャッシュ
corepack prepare pnpm@9.15.0 --activate

# 以降、オフラインでもpnpm 9.15.0が使える
pnpm install  # キャッシュから起動
```

## 4. package.json の packageManager フィールド

### フィールドの書き方

`packageManager` フィールドは、プロジェクトで使用するパッケージマネージャとそのバージョンを宣言する。

```json
{
  "name": "my-project",
  "version": "1.0.0",
  "packageManager": "pnpm@9.15.0"
}
```

フォーマットは `<パッケージマネージャ名>@<バージョン>` だ。

```json
// pnpmの場合
"packageManager": "pnpm@9.15.0"

// yarnの場合
"packageManager": "yarn@4.6.0"
```

### ハッシュ付き宣言

より厳密に管理したい場合は、パッケージマネージャのバイナリのハッシュ値を付加できる。

```json
"packageManager": "pnpm@9.15.0+sha512.76e2379760a4328ec4f..."
```

ハッシュ値は `corepack prepare` 実行時に自動生成される場合がある。改ざん検出に使われるが、実務では省略するプロジェクトも多い。

### corepack use コマンド

`corepack use` コマンドを使うと、`packageManager` フィールドの設定と `corepack prepare` を一度に実行できる。

```bash
# package.jsonにpackageManagerフィールドを追加し、
# 同時にキャッシュにも取得する
corepack use pnpm@9.15.0
```

このコマンドは `package.json` を直接書き換えるため、手動でフィールドを追加するよりも確実だ。

### packageManagerフィールドの効果

Corepackが有効な環境で `packageManager` フィールドが設定されていると、以下の挙動になる。

**1. 自動バージョン取得**: 宣言されたバージョンがローカルにない場合、自動的にダウンロードされる。

```bash
# package.jsonに "packageManager": "pnpm@9.15.0" がある状態で
pnpm install
# → pnpm 9.15.0が自動ダウンロードされ、installが実行される
```

**2. 別のパッケージマネージャの使用をブロック**: pnpmが宣言されたプロジェクトでyarnを実行しようとするとエラーになる。

```bash
# package.jsonに "packageManager": "pnpm@9.15.0" がある状態で
yarn install
# → エラー: This project is configured to use pnpm
```

**3. バージョン不一致の自動解決**: 宣言と異なるバージョンがグローバルにインストールされていても、Corepackが宣言されたバージョンを使用する。

```bash
# package.jsonに "packageManager": "pnpm@9.15.0" がある状態で
# pnpm 10.2.0がグローバルにインストールされている場合
pnpm install
# → Corepackが9.15.0を取得して使用（グローバルの10.2.0は無視される）
```

## 5. チーム開発でのバージョン統一手法

### 導入手順（既存プロジェクト）

既存プロジェクトにCorepackを導入する手順を示す。

**ステップ1: 現在のバージョンを確認する**

```bash
pnpm --version
# 9.15.0
```

**ステップ2: package.jsonにpackageManagerを追加する**

```bash
corepack use pnpm@9.15.0
```

**ステップ3: チームに周知する**

READMEに以下を追加する。

```markdown
## セットアップ

Node.js 22以上が必要です。

\```bash
# Corepackを有効化（初回のみ）
corepack enable

# 依存パッケージのインストール
pnpm install
\```
```

**ステップ4: .npmrcでの補強（pnpmの場合）**

pnpmの場合、`.npmrc` の `engine-strict` と `package.json` の `engines` を組み合わせると、Node.jsバージョンとパッケージマネージャバージョンの両方を統一できる。

```ini
# .npmrc
engine-strict=true
```

```json
// package.json
{
  "packageManager": "pnpm@9.15.0",
  "engines": {
    "node": ">=22.0.0"
  }
}
```

### only-allow スクリプトとの比較

Corepack以前からある手法として、`only-allow` パッケージを使う方法がある。

```json
{
  "scripts": {
    "preinstall": "npx only-allow pnpm"
  }
}
```

両者の違いは以下の通りだ。

| 項目 | Corepack | only-allow |
|---|---|---|
| バージョン指定 | パッケージマネージャのバージョンまで固定 | パッケージマネージャの種類のみ制限 |
| 追加依存 | 不要（Node.js同梱） | npxで毎回ダウンロード |
| 実行タイミング | コマンド実行時（シム経由） | `preinstall`フック時 |
| npm以外の自動取得 | あり（自動ダウンロード） | なし |

Corepackの方が「バージョンまで固定できる」「追加のnpx実行が不要」という点で優れている。ただし、Corepackが実験的ステータスであることと、将来の非バンドル化（後述）を考慮する必要がある。

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説しています。
:::

## 6. CI/CD（GitHub Actions）での活用

### pnpmプロジェクトの場合

GitHub Actionsでpnpmプロジェクトをビルドする際、Corepackを使う方法と専用アクションを使う方法がある。

**方法1: Corepackを使う**

```yaml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'

      # Corepackを有効化
      - run: corepack enable

      # package.jsonのpackageManagerフィールドに基づいて
      # pnpmが自動取得される
      - run: pnpm install --frozen-lockfile

      - run: pnpm test
      - run: pnpm build
```

**方法2: pnpm/action-setupを使う（Corepack非依存）**

```yaml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 9.15.0  # バージョンを明示指定

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile
      - run: pnpm test
      - run: pnpm build
```

**方法2の補足**: `pnpm/action-setup@v4` は `package.json` の `packageManager` フィールドを読み取る機能もある。`version` パラメータを省略すると `packageManager` フィールドの値が使われる。

```yaml
      - uses: pnpm/action-setup@v4
        # versionを省略 → package.jsonのpackageManagerから取得
```

### yarnプロジェクトの場合

```yaml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'

      - run: corepack enable

      - run: yarn install --immutable

      - run: yarn test
      - run: yarn build
```

Yarn Berry（v2以降）では `--frozen-lockfile` ではなく `--immutable` を使う点に注意する。

### CIでのキャッシュ設定

`actions/setup-node` のキャッシュ機能を使う場合、Corepackとの組み合わせで注意点がある。`setup-node` の `cache: 'pnpm'` は `pnpm` コマンドが利用可能な状態でないとキャッシュキーの生成に失敗する。そのため、**Corepackの有効化は `setup-node` の前に行う**必要がある。

```yaml
      # 順序が重要:
      # 1. Corepack有効化
      - run: corepack enable

      # 2. setup-node（cache: 'pnpm' はpnpmコマンドに依存）
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'pnpm'

      # 3. install
      - run: pnpm install --frozen-lockfile
```

### どちらの方法を選ぶか

| 観点 | Corepack | 専用アクション |
|---|---|---|
| 設定の簡潔さ | `corepack enable` の1行 | アクションの追加が必要 |
| package.jsonとの一貫性 | `packageManager`フィールドと自動連動 | `version`パラメータの二重管理になりうる |
| Node.js 25以降 | 非バンドル化済みのため追加インストール必要 | 影響なし |
| 安定性 | experimental | 安定版 |

現時点では、**ローカル開発ではCorepack、CIでは専用アクション**という組み合わせが実用的だ。CIでは `pnpm/action-setup` のように安定したセットアップ方法を使いつつ、ローカルでは `packageManager` フィールドによるバージョン統一の恩恵を受けられる。

## 7. Node.js 25以降のCorepack非バンドル化への対応

### 何が起きるのか

Node.jsコアチームは、**Node.js 25でCorepackをNode.jsにバンドルしない**方針を決定し、実行した。Node.js 25（2025年4月リリース）以降、Corepackはデフォルトでは同梱されていない。

廃止ではなく「非バンドル化」である点に注意してほしい。Corepack自体はnpmパッケージとして存続しており、`npm install -g corepack` で引き続き利用できる。

### 非バンドル化の理由

Node.jsの公式ディスカッション（GitHub issue）で挙げられた主な理由は以下だ。

1. **スコープの問題**: パッケージマネージャの管理はNode.jsランタイムの責務ではない
2. **メンテナンスコスト**: npm / yarn / pnpmの仕様変更への追従が負担になっている
3. **experimentalから脱却できなかった**: 導入から3年以上経過しても実験的ステータスのまま

### 実務への影響と対策

**影響1: ローカル開発環境**

Node.js 25以降をインストールした環境では、`corepack enable` が「コマンドが見つからない」エラーになる。

対策は以下の通りだ（Node.js 25以降で必須の手順）。

```bash
# 対策A: npmでCorepackをグローバルインストール
npm install -g corepack
corepack enable

# 対策B: パッケージマネージャを直接インストール
npm install -g pnpm@9.15.0
```

**影響2: CI/CD環境**

`corepack enable` に依存しているCIワークフローが失敗する。

```yaml
# Node.js 25以降で失敗するワークフロー例
- uses: actions/setup-node@v4
  with:
    node-version: '25'
- run: corepack enable  # ← コマンドが存在しないためエラー
```

対策として、専用のセットアップアクションに移行する。

```yaml
# 対策: 専用アクションを使う（Node.js 25でも動作する）
- uses: pnpm/action-setup@v4
  with:
    version: 9.15.0
```

**影響3: Dockerイメージ**

公式Node.jsのDockerイメージ（`node:25`以降）にもCorepackが含まれていない。

```dockerfile
# Node.js 25以降のDockerfile
FROM node:25-slim

# Corepackを使う場合は明示的にインストール
RUN npm install -g corepack && corepack enable

# または直接pnpmをインストール
RUN npm install -g pnpm@9.15.0
```

### 移行チェックリスト

Node.js 25はすでにリリースされている。Node.js 22（LTS）からの移行に備えて、以下を確認しておくべきだ。

1. **CIワークフローを確認**: `corepack enable` に依存している箇所を洗い出す
2. **専用アクションへ移行する**: `pnpm/action-setup` などに切り替える
3. **Dockerfileを確認**: Corepackに依存している行を特定し、`npm install -g corepack` を追加するか、直接インストールに変更する
4. **`packageManager` フィールドは維持する**: Corepackが非バンドル化されても、`packageManager` フィールド自体はNode.jsエコシステムの標準として残っている。`pnpm/action-setup` などのツールがこのフィールドを読み取る

## 8. pnpm / yarn との連携パターン

### pnpmとの連携

pnpmはCorepackとの連携が最もスムーズなパッケージマネージャだ。

**初期セットアップ**

```bash
# Corepackを有効化
corepack enable

# pnpmのバージョンを設定（package.jsonに書き込まれる）
corepack use pnpm@9.15.0

# 以降はpnpmコマンドがそのまま使える
pnpm install
pnpm add express
pnpm run dev
```

**バージョンアップ**

pnpmのバージョンを上げるときは `corepack use` で更新する。

```bash
# pnpmを10.2.0に更新
corepack use pnpm@10.2.0
# → package.jsonのpackageManagerが "pnpm@10.2.0" に更新される

# lockfileの再生成
pnpm install
```

バージョンアップ後は `pnpm install` を実行してlockfileを更新し、変更をコミットする。

**pnpm 10.xでの注意点**

pnpm 10では `postinstall` スクリプトがデフォルトで実行されなくなった。ネイティブモジュール（`sharp`、`bcrypt`、`esbuild` など）を使っている場合は、`package.json` で明示的に許可する。

```json
{
  "packageManager": "pnpm@10.2.0",
  "pnpm": {
    "onlyBuiltDependencies": ["sharp", "esbuild"]
  }
}
```

### yarnとの連携

Yarn Berry（v2以降）はCorepackと連携しつつも、独自のバージョン管理機能（`yarn set version`）を持っている。

**Corepack経由でのセットアップ**

```bash
corepack enable
corepack use yarn@4.6.0
yarn install
```

**yarn set version（Yarn独自機能）**

Yarn Berryには `yarn set version` という独自のバージョン管理コマンドがある。これはCorepackなしでも動作する。

```bash
# プロジェクトにYarnの特定バージョンを固定
yarn set version 4.6.0
# → .yarn/releases/ にバイナリがダウンロードされる
# → .yarnrc.yml に yarnPath が記録される
```

**CorepackとYarn独自機能の使い分け**

| 方法 | メリット | デメリット |
|---|---|---|
| Corepack | チーム全体で統一的な管理方法 | Node.js 25以降で追加インストール必要 |
| yarn set version | Corepack非依存で動作 | Yarn固有の仕組みに依存 |

Yarnプロジェクトでは、**`packageManager` フィールドと `yarn set version` を併用**するケースが多い。`packageManager` で宣言しつつ、`yarn set version` でプロジェクトローカルにもバイナリを持つことで、Corepackの有無に関わらず動作する。

### npm との関係

npmはCorepackの管理対象外だ。npmはNode.jsに直接同梱されているため、Corepackを経由する必要がない。

ただし、`packageManager` フィールドにnpmを指定することはできる。

```json
"packageManager": "npm@10.9.0"
```

この場合、Corepackが有効な環境で yarn や pnpm を実行しようとするとブロックされる。「このプロジェクトはnpmを使う」という宣言としての効果がある。

## 9. トラブルシューティング

### キャッシュ関連のトラブル

**症状: 古いバージョンが使われ続ける**

`packageManager` フィールドを更新したのに、古いバージョンが使われる場合がある。

```bash
# キャッシュの場所を確認
# デフォルト: ~/.cache/node/corepack（Linux）
#            ~/Library/Caches/node/corepack（macOS）

# COREPACK_HOME環境変数で変更可能
echo $COREPACK_HOME

# キャッシュをクリアして再取得
corepack prepare pnpm@9.15.0 --activate
```

**症状: ダウンロードに失敗する**

ネットワーク環境によってはCorepackのダウンロードが失敗する場合がある。

```bash
# エラー例
Error: Failed to fetch pnpm@9.15.0 from the registry
```

Corepackはデフォルトで npm レジストリ（`https://registry.npmjs.org`）からパッケージマネージャをダウンロードする。プロキシ環境やプライベートレジストリを使っている場合は、環境変数で制御する。

### プロキシ設定

企業のプロキシ環境でCorepackを使う場合、以下の環境変数を設定する。

```bash
# HTTPプロキシ
export HTTP_PROXY=http://proxy.example.com:8080
export HTTPS_PROXY=http://proxy.example.com:8080

# プロキシ除外
export NO_PROXY=localhost,127.0.0.1

# Corepackのダウンロード元を変更（社内レジストリを使う場合）
export COREPACK_NPM_REGISTRY=https://registry.your-company.com
```

`COREPACK_NPM_REGISTRY` 環境変数を使うと、パッケージマネージャのダウンロード元を社内のnpmレジストリに変更できる。

### よくあるエラーと対処法

**エラー1: `command not found: corepack`**

```bash
$ corepack enable
zsh: command not found: corepack
```

原因と対処:
- Node.jsのバージョンが古い（14.19未満 / 16.9未満） → Node.jsをアップデートする
- PATHにNode.jsのbinディレクトリが含まれていない → `which node` で確認し、同じディレクトリにcorepackがあるか確認する
- Node.js 25以降を使っている → `npm install -g corepack` でインストールする

**エラー2: `This project is configured to use pnpm`**

```bash
$ npm install
Usage Error: This project is configured to use pnpm
```

プロジェクトの `packageManager` フィールドで指定されたパッケージマネージャと異なるコマンドを使おうとしている。`package.json` を確認し、指定されたパッケージマネージャを使う。

**エラー3: `EACCES: permission denied` でシムが作成できない**

```bash
$ corepack enable
Error: EACCES: permission denied
```

前述の「権限エラーへの対処」セクションを参照。nvm/fnmを使っている場合は権限問題が起きにくい。システムインストールのNode.jsの場合は `sudo corepack enable` を使う。

**エラー4: Corepackのシムが既存のグローバルインストールと競合する**

```bash
# npmで直接インストールしたpnpmが残っている
$ which pnpm
/usr/local/bin/pnpm  # Corepackのシムではなく直接インストールされたバイナリ
```

この場合、先にグローバルのパッケージマネージャをアンインストールする。

```bash
# グローバルのpnpmを削除
npm uninstall -g pnpm

# Corepackを再度有効化
corepack enable

# 確認
which pnpm
# → Corepackのシムが優先されるようになる
```

### COREPACK_ENABLE_STRICT 環境変数

`COREPACK_ENABLE_STRICT` 環境変数を使うと、`packageManager` フィールドが存在しないプロジェクトでの挙動を制御できる。

```bash
# strict モード: packageManagerフィールドがないプロジェクトでは
# yarn/pnpmの実行をブロックする
export COREPACK_ENABLE_STRICT=1
```

デフォルトでは `0`（non-strict）で、`packageManager` フィールドがなくてもyarn/pnpmを実行できる。

## 10. まとめ ── Corepackの現在地と実務での使いどころ

### Corepackを使うべきケース

- **チーム開発でパッケージマネージャのバージョンを統一したい**場合（最大のメリット）
- **pnpmやyarnを使うプロジェクト**で、メンバーに個別のインストール手順を強制したくない場合
- **複数プロジェクトで異なるバージョン**のパッケージマネージャを使い分けたい場合

### Corepackを使わなくてもよいケース

- **個人開発でバージョン統一の必要がない**場合
- **CIで専用アクション（`pnpm/action-setup`等）を使っている**場合（CIではCorepack不要）
- **Node.js 25以降を使っている**プロジェクトで、追加インストールの手間を避けたい場合

### 推奨構成（2026年時点）

```json
// package.json
{
  "packageManager": "pnpm@9.15.0",
  "engines": {
    "node": ">=22.0.0"
  }
}
```

```yaml
# .github/workflows/ci.yml
- uses: pnpm/action-setup@v4  # CIではCorepack非依存
- uses: actions/setup-node@v4
  with:
    node-version: '22'
    cache: 'pnpm'
- run: pnpm install --frozen-lockfile
```

```markdown
<!-- README.md のセットアップセクション -->
corepack enable && pnpm install
```

この構成なら、ローカルではCorepackの恩恵を受けつつ、CIではCorepackに依存しない安定した環境を構築できる。Node.js 25でCorepackが非バンドル化された現在も、CIへの影響はない。

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説しています。
:::