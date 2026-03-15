---
title: "Bun installガイド：Bunをパッケージマネージャとして使う実践入門【2026年版】"
emoji: "📦"
type: "tech"
topics: ["bun", "nodejs", "npm", "packagemanager"]
published: true
---

## はじめに ── Bunは「もう一つのパッケージマネージャ」である

Bunと聞くと「高速なJavaScriptランタイム」を思い浮かべる方が多いだろう。しかしBunにはもう一つの顔がある。**Node.js互換のパッケージマネージャ**だ。

`bun install` は既存のNode.jsプロジェクトにそのまま導入できる。`package.json` があれば動く。新しいプロジェクト構成に書き換える必要はない。ランタイムとしてのBunを採用しなくても、パッケージマネージャだけを `bun install` に置き換えるという選択肢があるのだ。

この記事では、2026年3月時点の最新情報をもとに、Bunのパッケージマネージャ機能に絞った実践的な使い方を解説する。

**対象読者**: Node.jsでnpm/yarn/pnpmを使っている中級エンジニア。Bunのパッケージマネージャ機能に興味があるが、まだ試していない方。

**この記事で扱う範囲**:

- `bun install` / `bun add` / `bun remove` / `bun update` の実用的な使い方
- lockfileの仕組みと変遷（`bun.lockb` から `bun.lock` へ）
- 既存プロジェクトからの移行手順
- workspacesによるモノレポ対応
- CI/CDでの導入
- 現時点での制限事項と、npm/pnpmとの使い分け判断基準

## 1. Bunのパッケージマネージャ機能概要

Bunのパッケージマネージャは、npm互換のパッケージインストーラだ。以下の特徴がある。

**npm互換**

- `package.json` の `dependencies` / `devDependencies` / `optionalDependencies` / `peerDependencies` をすべて解釈する
- `overrides`（npm形式）と `resolutions`（Yarn形式）の両方に対応する
- `node_modules` ディレクトリを生成する（npmやyarnと同じ構造）
- `.npmrc` の設定を読み取る

**速度面の設計**

- Zig言語で実装されたネイティブバイナリ
- OS固有の最適化（macOSでは `clonefile`、Linuxでは `hardlink`）
- npmレジストリのメタデータをバイナリキャッシュで保持
- lockfileが存在し `package.json` が変更されていない場合、不足パッケージのみを遅延ダウンロード

**セキュリティ面の設計**

- 依存パッケージの `postinstall` などのライフサイクルスクリプトをデフォルトで実行しない
- 実行を許可するパッケージは `trustedDependencies` で明示的に指定する
- `--minimum-release-age` による公開直後パッケージのブロック機能

```json
// package.json
{
  "trustedDependencies": ["esbuild", "sharp"]
}
```

`esbuild` や `sharp` のようにネイティブバイナリのダウンロードが必要なパッケージは、`trustedDependencies` に追加しないとpostinstallスクリプトが実行されず、正しくインストールされない。Bunは主要なパッケージについて自動最適化を行うが、初回導入時にビルドエラーが出た場合はこの設定を確認すること。

## 2. bun install の仕組み

### 基本的な使い方

```bash
# プロジェクトの全依存をインストール
bun install

# 特定のパッケージをインストール（bun addと同等）
bun install react
bun install react@19.0.0    # バージョン指定
bun install react@latest    # タグ指定
```

`bun install` を実行すると、以下の処理が行われる。

1. `package.json` を読み取り、すべての依存関係を解決する
2. `peerDependencies` を自動的にインストールする（npmと異なりデフォルトで有効）
3. プロジェクトの `{pre|post}install` と `{pre|post}prepare` スクリプトを実行する
4. `node_modules` ディレクトリを生成する
5. `bun.lock` をプロジェクトルートに書き出す

### node_modules の生成方式

Bun v1.3.2以降では、2つのインストール戦略が選べる。

```bash
# ホイスティング方式（npm/Yarnと同じフラット構造）
bun install --linker hoisted

# 分離方式（pnpmに近い厳密な依存分離）
bun install --linker isolated
```

**hoisted**（ホイスティング方式）は、依存を `node_modules` 直下にフラットに配置する。npmやYarn Classicと同じ方式で、既存プロジェクトとの互換性が高い反面、Phantom Dependency（宣言していない依存を `require` できてしまう問題）が発生する。

**isolated**（分離方式）は、`node_modules/.bun/` に中央ストアを作り、シンボリックリンクでパッケージを配置する。pnpmのcontent-addressable storeに近い設計で、Phantom Dependencyを防ぐ。

デフォルトの戦略は以下のルールで決まる。

| プロジェクトの状態 | デフォルト |
|---|---|
| 新規のワークスペース/モノレポ | `isolated` |
| 新規の単一パッケージ | `hoisted` |
| 既存プロジェクト（v1.3.2以前から存在） | `hoisted` |

`bunfig.toml` で固定することもできる。

```toml
# bunfig.toml
[install]
linker = "isolated"
```

### lockfile: bun.lock

`bun install` を実行すると `bun.lock` が生成される。このファイルはGitにコミットすること。lockfileがあることで、チームメンバーやCI環境で同一バージョンの依存が再現される。

```bash
# lockfileだけ生成（node_modulesは作らない）
bun install --lockfile-only

# lockfileを生成しない
bun install --no-save
```

lockfileの詳細は次のセクションで解説する。

## 3. bun add / bun remove / bun update

### bun add ── パッケージの追加

```bash
# dependencies に追加
bun add express

# devDependencies に追加
bun add -d typescript
bun add --dev @types/node

# optionalDependencies に追加
bun add --optional fsevents

# peerDependencies に追加
bun add --peer react

# バージョンを固定（^ なしで記録）
bun add react --exact
# → "react": "19.1.0"（^19.1.0 ではなく）

# グローバルインストール
bun add -g tsx
```

npmとの対応表だ。

| 操作 | npm | bun |
|---|---|---|
| 依存追加 | `npm install express` | `bun add express` |
| dev依存追加 | `npm install -D typescript` | `bun add -d typescript` |
| バージョン固定 | `npm install react --save-exact` | `bun add react --exact` |
| グローバル | `npm install -g tsx` | `bun add -g tsx` |

### bun remove ── パッケージの削除

```bash
# パッケージを削除（package.json + node_modules + lockfile を更新）
bun remove ts-node

# グローバルから削除
bun remove -g tsx
```

シンプルだ。`npm uninstall` と同じ挙動で、`package.json` から該当エントリを削除し、`node_modules` と `bun.lock` を更新する。

### bun update ── パッケージの更新

```bash
# すべての依存を、semver範囲内で最新に更新
bun update

# 特定パッケージだけ更新
bun update react

# semver範囲を無視して最新バージョンに更新
bun update --latest

# 対話的に更新対象を選択
bun update --interactive
```

`bun update` と `bun update --latest` の違いは重要だ。

```json
// package.json
{
  "dependencies": {
    "react": "^17.0.2"
  }
}
```

- `bun update`: `^17.0.2` の範囲内（17.x系）で最新に更新
- `bun update --latest`: 範囲を無視して18.x、19.x等の最新版に更新

`--interactive` モードでは、パッケージごとにcurrent/target/latestのバージョンが表示され、スペースキーで更新対象を選択できる。モノレポでは `--recursive` を付けると全ワークスペースの依存を一括で確認できる。

```bash
# モノレポの全ワークスペースで対話的に更新
bun update --interactive --recursive
```

## 4. bun.lockb から bun.lock への変遷

Bunのlockfileは、v1.2を境に大きく変わった。

### bun.lockb（v1.1以前） ── バイナリ形式

Bun v1.1以前のlockfileは `bun.lockb` というバイナリ形式だった。パース速度を最優先した設計だったが、以下の問題があった。

- **git diffで差分が読めない**: バイナリファイルのためコードレビューで変更内容を確認できない
- **手動での編集が不可能**: テキストエディタで開いても意味のある内容が表示されない
- **他ツールとの相互運用性がない**: npm/yarn/pnpmのlockfileとの変換が困難

### bun.lock（v1.2以降） ── テキスト形式（JSONC）

2024年12月、Bun v1.2でテキストベースの `bun.lock` がデフォルトになった。フォーマットはJSONC（コメント付きJSON）だ。

```jsonc
// bun.lock - テキスト形式のlockfile
{
  "lockfileVersion": 1,
  "workspaces": {
    "": {
      "dependencies": {
        "express": "^4.18.2"
      }
    }
  },
  "packages": {
    "express": ["express@4.18.2", "", { "dependencies": { ... } }]
    // ...
  }
}
```

テキスト形式への移行により、以下が改善された。

- **git diffで差分が明確に表示される**: どのパッケージのバージョンが変わったかが一目で分かる
- **コードレビューが可能**: PRで依存の変更を確認できる
- **他ツールからの移行が容易**: 既存のlockfileからの自動変換をサポート

### 既存の bun.lockb からの移行

```bash
# bun.lockb を bun.lock に変換
bun install --save-text-lockfile --frozen-lockfile --lockfile-only

# 変換完了後、旧ファイルを削除
rm bun.lockb

# .gitignore から bun.lockb を削除し、bun.lock をコミット
git add bun.lock
git rm --cached bun.lockb  # 追跡済みの場合
git commit -m "Migrate from bun.lockb to bun.lock"
```

`--frozen-lockfile` を付けることで、依存の解決結果を変えずにフォーマットだけを変換できる。

:::message
この記事はBunの **パッケージマネージャ機能の使い方（HOW）** にフォーカスしている。「なぜnpmのnode_modulesはフラットなのか」「pnpmのcontent-addressable storeはどう動くのか」といった**依存解決の仕組み（WHY）** を理解したい方は、以下の書籍が参考になる。

📖 **[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)** では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説している。
:::

## 5. npm/yarn/pnpmからの移行手順

### 移行の基本方針

Bunは既存のlockfileを自動検出して `bun.lock` に変換する。`bun.lock` が存在しない状態で `bun install` を実行すると、以下のファイルが検出された場合に自動移行が行われる。

- `package-lock.json`（npm）
- `yarn.lock`（Yarn v1）
- `pnpm-lock.yaml`（pnpm）

元のlockfileは削除されず、そのまま残る。

### npmからの移行

```bash
# 1. Bunをインストール（未インストールの場合）
curl -fsSL https://bun.sh/install | bash

# 2. プロジェクトで bun install を実行
# package-lock.json を自動検出して bun.lock を生成
bun install

# 3. 動作確認
bun run build    # npm scripts はそのまま動く
bun run test

# 4. 確認後、旧lockfileを削除
rm package-lock.json
```

### pnpmからの移行

pnpmからの移行では、追加の設定移行が自動で行われる。

```bash
# bun install を実行するだけ
# pnpm-lock.yaml + pnpm-workspace.yaml を自動検出
bun install
```

自動移行される項目は以下の通りだ。

| pnpmの設定 | 移行先 |
|---|---|
| `pnpm-workspace.yaml` の `packages` | `package.json` の `workspaces` |
| `pnpm.overrides` | `package.json` の `overrides` |
| `pnpm.patchedDependencies` | `package.json` の `patchedDependencies` |
| `catalog:` プロトコル | `package.json` の `workspaces.catalog` |

pnpmの `catalog:` プロトコル（依存バージョンの一元管理）もそのまま維持される。

```json
// 移行後の package.json
{
  "workspaces": {
    "packages": ["apps/*", "packages/*"],
    "catalog": {
      "react": "^18.0.0",
      "typescript": "^5.0.0"
    }
  }
}
```

**注意**: pnpm lockfile version 7以上が必要だ。古いバージョンの場合は、先に `pnpm install` でlockfileを更新すること。

### Yarn Classicからの移行

```bash
# yarn.lock (v1) を自動検出して変換
bun install

# 確認後、旧ファイルを削除
rm yarn.lock
```

Yarn Berry（v2以降）の `yarn.lock`（v6形式以降）からの自動移行はサポートされていない。Yarn Berryからの移行は、`yarn.lock` を削除してから `bun install` で `package.json` ベースで再解決するのが現実的だ。

### 移行時のチェックリスト

移行後に以下を確認すること。

- [ ] `bun run build` でビルドが通ること
- [ ] `bun run test` でテストが通ること
- [ ] `postinstall` が必要なパッケージが `trustedDependencies` に追加されていること
- [ ] CIの設定ファイルを更新したこと（次のセクションで解説）
- [ ] チームメンバーがBunをインストール済みであること

## 6. package.jsonの互換性

### scripts

`bun run` は `package.json` の `scripts` フィールドをそのまま実行できる。

```json
{
  "scripts": {
    "build": "tsc && vite build",
    "test": "vitest",
    "lint": "eslint src/",
    "start": "node dist/index.js"
  }
}
```

```bash
bun run build   # npm run build と同等
bun run test    # npm run test と同等
```

`pre` / `post` スクリプトもnpmと同様に動作する。`prebuild` があれば `bun run build` の前に自動実行される。

### dependencies フィールド群

`dependencies` / `devDependencies` / `optionalDependencies` / `peerDependencies` のすべてに対応している。

`peerDependencies` については、npmがv7以降でデフォルト自動インストールに変わったのと同様に、Bunもデフォルトで自動インストールする。`peerDependenciesMeta` の `optional` 指定も認識する。

### overrides と resolutions

npm形式の `overrides` とYarn形式の `resolutions` の両方を認識する。

```json
{
  "overrides": {
    "lodash": "4.17.21"
  },
  "resolutions": {
    "minimist": "1.2.8"
  }
}
```

間接依存のバージョンを強制的に固定したい場合に使う。どちらの書き方でも動作するため、npmから移行してもYarnから移行しても `package.json` の修正は不要だ。

### engines フィールド

`package.json` の `engines` フィールドは読み取るが、2026年3月時点ではデフォルトで制約の強制はしない。

### その他の互換性

| フィールド | 対応状況 |
|---|---|
| `workspaces` | 対応（globパターン、ネガティブパターンも可） |
| `private` | 対応 |
| `bin` | 対応 |
| `files` | 対応 |
| `exports` / `imports` | 対応（ランタイム側の機能） |
| `type: "module"` | 対応（ランタイム側の機能） |
| `packageManager` | 読み取るがCorepackとしての強制はしない |
| `bundleDependencies` | 対応 |

## 7. workspaces対応 ── モノレポでの利用

### 基本設定

```
my-monorepo/
├── package.json          # workspaces 定義
├── bun.lock
├── packages/
│   ├── core/
│   │   └── package.json
│   ├── ui/
│   │   └── package.json
│   └── api/
│       └── package.json
```

ルートの `package.json` で `workspaces` を定義する。

```json
// ルート package.json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": ["packages/*"]
}
```

各ワークスペースのパッケージは `workspace:` プロトコルで相互参照できる。

```json
// packages/api/package.json
{
  "name": "@my-monorepo/api",
  "dependencies": {
    "@my-monorepo/core": "workspace:*"
  }
}
```

`bun install` を実行すると、ルートと全ワークスペースの依存が一括でインストールされる。共通の依存はルートの `node_modules` にホイスティングされ、ディスク使用量が節約される。

### フィルタリング

特定のワークスペースだけインストールしたい場合は `--filter` を使う。

```bash
# pkg-a のみインストール
bun install --filter './packages/pkg-a'

# pkg-c 以外をインストール
bun install --filter '!pkg-c'

# 名前パターンでフィルタ
bun install --filter "pkg-*" --filter "!pkg-c"
```

スクリプト実行時にもフィルタが使える。

```bash
# 全ワークスペースでビルド
bun run --filter '*' build

# 特定ワークスペースでテスト
bun run --filter '@my-monorepo/api' test
```

### workspace: プロトコルの挙動

`workspace:` で指定したバージョンは、パッケージ公開時に自動で実バージョンに置き換えられる。

| 指定 | 公開時の変換（パッケージのversionが1.0.1の場合） |
|---|---|
| `workspace:*` | `1.0.1` |
| `workspace:^` | `^1.0.1` |
| `workspace:~` | `~1.0.1` |
| `workspace:1.0.2` | `1.0.2`（明示的なバージョンが優先） |

### Catalogs ── バージョンの一元管理

モノレポで複数のワークスペースが同じ依存を使う場合、`catalog` でバージョンを一元管理できる。

```json
// ルート package.json
{
  "workspaces": {
    "packages": ["packages/*"],
    "catalog": {
      "react": "^19.0.0",
      "typescript": "^5.7.0"
    }
  }
}
```

各ワークスペースでは `catalog:` プロトコルで参照する。

```json
// packages/ui/package.json
{
  "dependencies": {
    "react": "catalog:"
  },
  "devDependencies": {
    "typescript": "catalog:"
  }
}
```

バージョンを更新する際はルートの `catalog` を1箇所変更するだけで、全ワークスペースに反映される。

## 8. CI/CDでの活用

### GitHub Actions での設定

公式アクション [`oven-sh/setup-bun`](https://github.com/oven-sh/setup-bun) を使う。

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Bun
        uses: oven-sh/setup-bun@v2

      - name: Install dependencies
        run: bun ci

      - name: Lint
        run: bun run lint

      - name: Test
        run: bun run test

      - name: Build
        run: bun run build
```

### bun ci ── CI向けインストール

`bun ci` は `bun install --frozen-lockfile` と同等だ。以下の挙動をする。

- `bun.lock` の内容を厳密に再現する
- `package.json` と `bun.lock` の間に不整合があればエラーで停止する
- lockfileを更新しない

npmの `npm ci` に相当するコマンドで、CI環境では `bun install` ではなく `bun ci` を使うことを推奨する。

```bash
# CI環境向け（lockfileの厳密な再現）
bun ci

# 上と同等
bun install --frozen-lockfile
```

### キャッシュの活用

`bun install` は `~/.bun/install/cache/` にパッケージキャッシュを保存する。GitHub Actionsでキャッシュを活用する場合は以下のように設定する。

```yaml
- name: Cache Bun dependencies
  uses: actions/cache@v4
  with:
    path: ~/.bun/install/cache
    key: bun-${{ runner.os }}-${{ hashFiles('bun.lock') }}
    restore-keys: |
      bun-${{ runner.os }}-

- name: Install dependencies
  run: bun ci
```

### 本番ビルド向け

本番環境では `devDependencies` を除外してインストールできる。

```bash
# devDependencies を除外
bun install --production

# さらに frozen-lockfile で厳密に
bun install --production --frozen-lockfile
```

Dockerfileでの使用例だ。

```dockerfile
FROM oven/bun:1 AS base
WORKDIR /app

# 依存のインストール（キャッシュレイヤー分離）
COPY package.json bun.lock ./
RUN bun ci --production

# アプリケーションコードのコピー
COPY . .
RUN bun run build

# 本番イメージ
FROM oven/bun:1-slim
WORKDIR /app
COPY --from=base /app/dist ./dist
COPY --from=base /app/node_modules ./node_modules
CMD ["bun", "run", "start"]
```

## 9. 現時点での制限事項・互換性の課題（2026年3月時点）

Bunのパッケージマネージャは実用レベルに達しているが、以下の制限事項を認識しておく必要がある。

### ライフサイクルスクリプトのデフォルト無効

最も影響が大きい違いだ。npm/yarn/pnpmでは依存パッケージの `postinstall` がデフォルトで実行されるが、Bunではセキュリティ上の理由でデフォルト無効だ。

ネイティブバイナリを含むパッケージ（`esbuild`、`sharp`、`better-sqlite3`、`bcrypt` など）は `trustedDependencies` への追加が必要になる場合がある。Bunは主要パッケージについて自動最適化を行うが、あまり知られていないネイティブパッケージでは手動設定が必要だ。

```json
{
  "trustedDependencies": ["better-sqlite3", "bcrypt"]
}
```

**影響**: 初回移行時に「ネイティブモジュールのビルドエラー」として表面化する。

### Yarn Berry PnPモードからの移行非対応

Yarn Berry（v2以降）の `yarn.lock`（v6形式以降）からの自動lockfile移行はサポートされていない。PnPモード特有の `.pnp.cjs` や `.yarn/cache/` の仕組みとは根本的に異なるため、段階的な移行が難しい領域だ。

### npm/Yarn固有の機能との差異

| 機能 | npm | Bun |
|---|---|---|
| `npm audit` | 対応 | 非対応（代替なし） |
| `npm pack` | 対応 | `bun pm pack` で対応 |
| `npm publish` | 対応 | `bun publish` で対応 |
| `npm link` | 対応 | `bun link` で対応 |
| `npm outdated` | 対応 | `bun outdated` で対応 |
| `npm shrinkwrap` | 対応 | 非対応 |
| `npm fund` | 対応 | 非対応 |

`npm audit` に相当する脆弱性スキャン機能は2026年3月時点でBun単体にはない。CIパイプラインで別途 `npm audit`（npmがインストールされている場合）や、Snyk、Socket.devなどのサードパーティツールを併用する必要がある。

### .npmrc の部分的な対応

`.npmrc` の主要な設定（registry、auth token、scope設定）は読み取るが、すべてのnpmrc設定項目に対応しているわけではない。Bun固有の設定は `bunfig.toml` で行う。

### エコシステムの成熟度

npm/pnpmと比較して、以下の点でエコシステムがまだ発展途上だ。

- **CIサービスのネイティブサポート**: GitHub Actions以外のCIサービス（GitLab CI、CircleCIなど）では公式アクションが提供されていない場合がある
- **エディタ/IDE統合**: VS CodeのCorepack連携などはnpm/pnpm前提のものが多い
- **ドキュメント・情報量**: トラブルシューティング情報はnpm/pnpmに比べてまだ少ない

## 10. npm/pnpmとの使い分け判断基準

「Bunをパッケージマネージャとして使うべきか？」の判断基準を整理する。

### Bunが適しているケース

**個人プロジェクト・新規プロジェクト**

チーム全体の合意が不要で、試しやすい環境だ。`bun install` の速度の恩恵をすぐに感じられる。

**インストール速度がボトルネックになっているプロジェクト**

依存パッケージが多いプロジェクトで `npm install` に時間がかかっている場合、`bun install` への切り替えで大幅な改善が見込める。

**Bunをランタイムとしても採用しているプロジェクト**

ランタイムとパッケージマネージャを統一することで、ツールチェインがシンプルになる。

### npmを継続すべきケース

**チームにBunの知見がなく、移行コストを取れない**

npm は Node.js に同梱されており追加インストールが不要だ。チーム全員が使い方を知っている。移行のメリットが明確でなければ、そのまま使い続けるのが合理的だ。

**`npm audit` を必須ワークフローに組み込んでいる**

セキュリティ監査のワークフローが `npm audit` に依存している場合、Bun単体では代替できない。

**ネイティブモジュールが多く、trustedDependenciesの管理が煩雑になる**

`postinstall` が必要なパッケージが多いプロジェクトでは、移行時の設定コストが大きくなる。

### pnpmを選ぶべきケース

**厳密な依存分離が最優先**

pnpmのcontent-addressable storeはPhantom Dependencyの完全な防止と、ディスク使用量の最適化を両立している。Bunの `--linker isolated` も同様の方向性だが、pnpmの方が実績とエコシステム連携（`pnpm deploy`、`pnpm patch` など）が豊富だ。

**大規模モノレポで、pnpm固有の機能を活用している**

`pnpm-workspace.yaml` のcatalog、`pnpm deploy`（ワークスペースの単体デプロイ）、`pnpm patch`（依存パッケージへのパッチ適用）など、pnpm固有の機能に依存しているプロジェクトでは、移行の利点が薄い場合がある。

### 判断フローチャート

```
[新規プロジェクト？]
  ├─ Yes → Bunのランタイムも使う？
  │         ├─ Yes → Bun
  │         └─ No  → npm or pnpm（チームの経験で選択）
  └─ No  → [現在の不満は？]
            ├─ インストール速度 → Bun移行を検討
            ├─ Phantom Dependency → pnpm移行を検討
            └─ 特になし → 現状維持（npmを継続）
```

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説している。
:::

## まとめ

Bunのパッケージマネージャ機能は、**既存のNode.jsプロジェクトにそのまま導入できる**のが最大の利点だ。

ポイントを振り返る。

1. `package.json` がそのまま使える ── 特別な設定ファイルは不要
2. `bun.lock` はテキスト形式（JSONC） ── git diffでレビュー可能
3. npm/yarn/pnpmのlockfileから自動移行 ── `bun install` だけで変換される
4. workspaces・overrides・resolutions に対応 ── モノレポも既存設定のまま動く
5. CI/CDは `bun ci` で再現性を確保 ── `npm ci` と同じ位置づけ
6. `trustedDependencies` の設定が移行時の最大の注意点
7. `npm audit` 相当の機能はないため、セキュリティスキャンは別途必要

まずは個人プロジェクトやサイドプロジェクトで `bun install` を試してみてほしい。`package.json` があるディレクトリで `bun install` を実行するだけで始められる。

:::message
この記事はAI（Claude）による生成を含みます。技術的な正確性については可能な限り検証していますが、最新情報は公式ドキュメントをご確認ください。
:::
