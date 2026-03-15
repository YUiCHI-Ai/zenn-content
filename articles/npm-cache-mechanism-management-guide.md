---
title: "npm cacheの仕組みと管理：キャッシュクリアから高速化まで完全ガイド"
emoji: "📦"
type: "tech"
topics: ["npm", "nodejs", "pnpm", "yarn", "cicd"]
published: false
---

## はじめに

「`npm install` が遅いからキャッシュを消そう」──そう思って `npm cache clean --force` を実行したことはないだろうか。

実はこのコマンド、**ほとんどの場合は不要であり、むしろインストール速度を低下させる**。npmのキャッシュは壊れにくい設計になっており、本当に必要な場面は限られている。

この記事では、npm/pnpm/yarnそれぞれのキャッシュの仕組みを正確に理解し、「いつキャッシュを消すべきか」「CI/CDでどうキャッシュを活用するか」「オフライン環境でどう運用するか」を実務ベースで解説する。

:::message
この記事は「キャッシュをどう使いこなすか」という**HOW**にフォーカスしている。「パッケージマネージャが内部でどのように依存を解決しているのか」「pnpmのContent-Addressable Storeがなぜ速いのか」といった**WHY（設計原理）**は、本記事の範囲外とする。
:::

## npm キャッシュの仕組み ── Content-Addressable Storage

### cacache とは何か

npmのキャッシュは **cacache**（Content-Addressable Cache）というライブラリで管理されている。名前の通り、**ファイルの内容（コンテンツ）のハッシュ値をキーとして保存**する仕組みだ。

通常のキャッシュは「URLやファイル名」でデータを特定するが、cacacheは違う。ダウンロードしたtarball（パッケージの圧縮ファイル）の **SHA-512ハッシュ** を計算し、そのハッシュ値をファイル名として保存する。

```
# 通常のキャッシュ（名前ベース）
cache/express-4.18.2.tgz  ← ファイル名でアクセス

# cacache（コンテンツベース）
cache/content-v2/sha512/ab/cd/ef1234...  ← ハッシュ値でアクセス
```

この仕組みには2つの利点がある。

1. **重複排除**: 同じ内容のファイルは同じハッシュになるため、二重保存されない
2. **改ざん検知**: ダウンロード後にキャッシュが破損していれば、ハッシュ不一致で検出できる

npmは `npm install` 時にレジストリからtarballをダウンロードすると、必ずキャッシュに保存する。次回以降の `npm install` では、キャッシュにヒットすればネットワークリクエストをスキップできる。

### キャッシュの場所と構造

npmのキャッシュはデフォルトで以下のディレクトリに保存される。

```bash
# キャッシュディレクトリを確認
npm config get cache

# 各OSのデフォルト
# macOS / Linux: ~/.npm
# Windows:       %LocalAppData%\npm-cache
```

キャッシュディレクトリの内部構造は以下の通りだ。

```
~/.npm/
├── _cacache/              # メインのキャッシュストア
│   ├── content-v2/        # 実際のtarballデータ（ハッシュ値で管理）
│   │   └── sha512/        # SHA-512ハッシュのディレクトリ
│   │       ├── 00/
│   │       ├── 01/
│   │       │   └── abc123...  # tarballの実体
│   │       └── ff/
│   ├── index-v5/          # メタデータインデックス
│   │   └── ...            # パッケージ名・バージョン → ハッシュのマッピング
│   └── tmp/               # 一時ファイル
├── _logs/                 # npmのデバッグログ
└── _update-notifier-last-checked  # 更新通知
```

**content-v2** にtarballの実体が、**index-v5** にメタデータ（どのパッケージのどのバージョンがどのハッシュに対応するか）が保存されている。

```bash
# キャッシュのサイズを確認
du -sh ~/.npm/_cacache/

# content-v2のファイル数を確認（多数のプロジェクトで作業していると数万件になることも）
find ~/.npm/_cacache/content-v2 -type f | wc -l
```

## npm cache コマンドの使い方

### npm cache verify ── まずはこれを使う

キャッシュに問題がありそうなとき、**最初に実行すべきは `npm cache verify`** であって、`npm cache clean --force` ではない。

```bash
npm cache verify
```

`npm cache verify` は以下の処理を行う。

1. キャッシュ内の全エントリのハッシュ値を再計算し、破損したエントリを削除する
2. 参照されていない孤立したコンテンツを削除する
3. インデックスの整合性を検証する
4. キャッシュの統計情報を表示する

出力例：

```
Cache verified and compressed (~/.npm/_cacache)
Content verified: 8432 (1245867520 bytes)
Content garbage collected: 12 (3456789 bytes)
Missing content: 0
Index entries: 9876
Finished in 4.253s
```

- **Content verified**: 正常なキャッシュエントリの数とサイズ
- **Content garbage collected**: 削除された不要エントリの数
- **Missing content**: インデックスはあるが実体がないエントリ（通常は0）

つまり `npm cache verify` は**壊れたエントリだけを選択的に削除し、正常なキャッシュは残す**。これが `npm cache clean --force` との決定的な違いだ。

### npm cache clean --force ── 本当に必要な場面は限られている

```bash
npm cache clean --force
```

このコマンドは `_cacache` ディレクトリ内の**全キャッシュを削除する**。`--force` フラグが必須なのは、npm側が「本当に必要ですか？」と問いかけているためだ。

npm v5以降、キャッシュはcacacheによって整合性が自動管理されているため、**手動でのキャッシュクリアが有効なケースは極めて少ない**。`--force` なしで `npm cache clean` を実行すると、以下のメッセージが表示される。

```
npm warn using --force Recommended protections against accidentally deleting data.
npm ERR! As of npm@5, the npm cache self-heals from corruption issues
npm ERR! by treating the cache as content-addressable.
npm ERR! ...
npm ERR! If you're sure you want to delete the entire cache,
npm ERR! rerun this command with --force.
```

#### npm cache clean --force が必要な場面

以下のケースでは `npm cache clean --force` が有効な場合がある。

1. **ディスク容量の逼迫**: キャッシュが数十GBに膨らんでディスクを圧迫しているとき
2. **npm自体のアップグレード後の互換性問題**: npmのメジャーバージョンアップでキャッシュ形式が変わった場合
3. **`npm cache verify` でも解決しないインストールエラー**: 極めてまれだが、キャッシュの完全再構築が必要な場合

```bash
# キャッシュサイズを確認してから判断する
du -sh ~/.npm/_cacache/

# 本当にクリアが必要なら
npm cache clean --force

# クリア後の確認
npm cache verify
```

### npm cache ls ── npm v7以降は非推奨

npm v6以前では `npm cache ls` でキャッシュの一覧を表示できたが、**npm v7以降では廃止されている**。

```bash
# npm v6以前
npm cache ls
# → キャッシュされたパッケージの一覧が表示される

# npm v7以降
npm cache ls
# → npm warn cache This command has been removed
```

npm v7以降でキャッシュの内容を確認したい場合は、`npm cache verify` の出力を確認するか、ディレクトリを直接調べる。

```bash
# キャッシュの統計を確認
npm cache verify

# インデックスの中身を直接見る（上級者向け）
# index-v5内のファイルはJSONで、パッケージのメタデータが格納されている
ls ~/.npm/_cacache/index-v5/
```

## pnpm のキャッシュ ── Content-Addressable Store

pnpmのキャッシュはnpmとは根本的に異なるアーキテクチャを持っている。npmの `_cacache` が「ダウンロードキャッシュ（次回のネットワークリクエストを省略するため）」であるのに対し、pnpmの **Content-Addressable Store** は「パッケージファイルの中央保管庫」として機能する。

### npm のキャッシュと pnpm のストアの根本的な違い

```
■ npm のキャッシュフロー
レジストリ → tarball DL → _cacache に保存 → 展開 → node_modules にコピー
                           ↑                              ↑
                      キャッシュ                     実際のファイル
                    (tarball丸ごと)                  (プロジェクトごと)

■ pnpm のストアフロー
レジストリ → tarball DL → CAS に個別ファイルで保存 → node_modules にハードリンク
                           ↑                                ↑
                    中央ストア                        同じ実体を参照
                  (ファイル単位、SHA-512)             (コピーなし)
```

この違いを理解することが重要だ。

| 比較項目 | npm `_cacache` | pnpm CAS |
|---|---|---|
| 保存単位 | tarball（パッケージ丸ごと） | 個別ファイル |
| ハッシュ対象 | tarball全体のSHA-512 | 各ファイルのSHA-512 |
| node_modulesへの展開 | tarballを展開してコピー | ハードリンクを作成 |
| ディスク使用量 | キャッシュ + node_modules分 | ストアのみ（ハードリンクは容量ゼロ） |
| 複数プロジェクト間の共有 | キャッシュは共有、node_modulesは各プロジェクトにコピー | ストアのファイルをハードリンクで直接共有 |

### pnpm ストアの場所と管理コマンド

```bash
# ストアの場所を確認
pnpm store path
# macOS/Linux: ~/.local/share/pnpm/store/v3
# Windows:     %LocalAppData%\pnpm\store\v3

# ストアの状態を確認
pnpm store status
# → 破損したパッケージがあれば報告される

# どのプロジェクトからも参照されていないパッケージを削除
pnpm store prune
```

`pnpm store prune` は、ストア内のパッケージのうち、現在どのプロジェクトの `node_modules` からもハードリンクされていないものを削除する。ディスク容量を節約したいときに使う。

:::message alert
`pnpm store prune` は、すべてのプロジェクトの `node_modules` が正常な状態であることを前提としている。一部のプロジェクトの `node_modules` を手動で削除した後に `pnpm store prune` を実行すると、そのプロジェクトで使っていたパッケージがストアから消えてしまい、次回 `pnpm install` でダウンロードが再発生する。
:::

### npmキャッシュとの使い分け

pnpmにもnpmと同様のHTTPレスポンスキャッシュがある。これはメタデータのキャッシュで、レジストリへのリクエストを削減する。

```bash
# pnpmのメタデータキャッシュディレクトリ
# macOS/Linux: ~/.local/share/pnpm/cache  (store-dirとは別)
# 環境変数: PNPM_HOME で確認可能
```

まとめると、pnpmには「ストア（CAS）」と「キャッシュ（メタデータ）」の2層があり、npmの `_cacache` は両方の役割を1つのディレクトリで担っている。

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説しています。
:::

## yarn のキャッシュ

Yarn Classic (v1) とYarn Berry (v2+) ではキャッシュの仕組みが大きく異なる。

### Yarn Classic (v1)

```bash
# キャッシュディレクトリの場所を確認
yarn cache dir
# macOS: ~/Library/Caches/Yarn/v6
# Linux: ~/.cache/yarn/v6

# キャッシュの一覧を確認
yarn cache list

# 特定パッケージのキャッシュを確認
yarn cache list --pattern express

# キャッシュを全削除
yarn cache clean

# 特定パッケージのキャッシュだけ削除
yarn cache clean express
```

Yarn Classicのキャッシュは、npmと同様に**tarball単位で保存**される。ただし、npmのcacacheとは異なり、パッケージ名とバージョンがディレクトリ名に含まれる構造になっている。

```
~/Library/Caches/Yarn/v6/
├── npm-express-4.18.2-abcdef1234.../
│   ├── node_modules/
│   │   └── express/
│   │       ├── index.js
│   │       ├── package.json
│   │       └── ...
│   └── .yarn-tarball.tgz
└── npm-lodash-4.17.21-fedcba9876.../
    └── ...
```

### Yarn Berry (v2+)

Yarn Berryではキャッシュの考え方が根本的に変わる。**Zero-Installs** を実現するため、キャッシュされたzipファイルをプロジェクト内（`.yarn/cache/`）に保存し、リポジトリにコミットする設計だ。

```bash
# Yarn Berryのキャッシュ場所
# プロジェクトローカル: .yarn/cache/
# グローバル: yarn config get cacheFolder

# キャッシュを全削除
yarn cache clean --all

# グローバルキャッシュのみ削除（ローカルは残す）
yarn cache clean --mirror
```

```
my-project/
├── .yarn/
│   ├── cache/                 # パッケージのzipファイル
│   │   ├── express-npm-4.18.2-abcdef.zip
│   │   ├── lodash-npm-4.17.21-fedcba.zip
│   │   └── ...
│   └── unplugged/             # zipで動作しないパッケージ
├── .pnp.cjs                  # PnP解決マップ
└── yarn.lock
```

Yarn BerryのZero-Installsを使う場合、`.yarn/cache/` をGitにコミットすることで **CI/CDでのinstallステップが不要** になる。ただし、リポジトリサイズが大幅に増加する（数百MBになることもある）トレードオフがある。

## CI/CD でのキャッシュ戦略

CI/CDでは「インストール時間の短縮」が直接的にビルド時間とコストの削減につながる。適切なキャッシュ戦略を選ぶことが重要だ。

### GitHub Actions でのキャッシュ

GitHub Actionsでは、`actions/setup-node` の `cache` オプションを使うのが最もシンプルだ。

#### npm の場合

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '22'
    cache: 'npm'   # ~/.npm をキャッシュ

- run: npm ci
```

`cache: 'npm'` を指定すると、`~/.npm` ディレクトリが `package-lock.json` のハッシュ値をキーとしてキャッシュされる。キャッシュがヒットすれば、`npm ci` 実行時にレジストリへのネットワークリクエストが削減される。

#### pnpm の場合

```yaml
- uses: pnpm/action-setup@v4
  with:
    version: 10

- uses: actions/setup-node@v4
  with:
    node-version: '22'
    cache: 'pnpm'  # pnpm store をキャッシュ

- run: pnpm install --frozen-lockfile
```

#### yarn の場合

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '22'
    cache: 'yarn'  # yarn cache をキャッシュ

- run: yarn install --immutable
```

### node_modules vs ~/.npm ── 何をキャッシュすべきか

`actions/cache` を直接使って細かく制御する場合、何をキャッシュするかの選択が重要になる。

| キャッシュ対象 | メリット | デメリット |
|---|---|---|
| `~/.npm`（npmキャッシュ） | lockfileが変わってもキャッシュの一部がヒットする / `npm ci` の整合性保証が維持される | `npm ci` の展開処理は毎回実行される |
| `node_modules` | キャッシュヒット時はinstallステップ自体をスキップできる | lockfileが変わったときに不整合が起きるリスクがある / キャッシュサイズが大きい |

**推奨は `~/.npm`（または各パッケージマネージャのキャッシュディレクトリ）のキャッシュ**だ。`node_modules` のキャッシュは速度面では有利だが、lockfileとの不整合リスクがあるため、`npm ci` の「既存node_modulesを削除してクリーンインストールする」という設計意図と矛盾する。

`node_modules` をキャッシュする場合は、**lockfileのハッシュをキャッシュキーに含め、キャッシュヒット時のみinstallをスキップ**するパターンを使う。

```yaml
- name: Cache node_modules
  id: cache-modules
  uses: actions/cache@v4
  with:
    path: node_modules
    key: node-modules-${{ hashFiles('package-lock.json') }}

- name: Install dependencies
  if: steps.cache-modules.outputs.cache-hit != 'true'
  run: npm ci
```

この方法なら、lockfileが変わるとキャッシュが無効化されるため、不整合のリスクを最小化できる。ただし、lockfileが変わった場合はフルインストールが走る点に注意が必要だ。

### GitHub Actions のキャッシュ上限

GitHub Actionsのキャッシュには **リポジトリあたり10GB** の上限がある。上限を超えると古いキャッシュから自動的に削除される。また、7日間アクセスのないキャッシュも削除される。

```bash
# 現在のキャッシュ使用状況を確認（GitHub CLI）
gh cache list
gh cache delete --all  # 全キャッシュを削除（問題発生時）
```

モノレポなど大規模プロジェクトでは、キャッシュサイズが上限に達しやすい。その場合は `restore-keys` を活用して部分ヒットを狙う。

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-${{ runner.os }}-
```

`restore-keys` を指定すると、完全一致するキャッシュがなくても前方一致でキャッシュを復元する。lockfileが変わっても、以前のキャッシュをベースに差分だけダウンロードすればよいため、フルダウンロードより高速だ。

## Docker でのキャッシュマウント

Dockerビルドでは、BuildKitの `--mount=type=cache` を使うことで、ビルド間でパッケージキャッシュを共有できる。

### npm の場合

```dockerfile
FROM node:22-slim AS builder
WORKDIR /app
COPY package.json package-lock.json ./

# BuildKitキャッシュマウントを使用
RUN --mount=type=cache,target=/root/.npm \
    npm ci

COPY . .
RUN npm run build
```

`--mount=type=cache,target=/root/.npm` は、`/root/.npm` ディレクトリの内容をビルド間で永続化する。Dockerのレイヤーキャッシュとは独立した仕組みなので、`package.json` が変わってレイヤーキャッシュが無効化されても、ダウンロード済みのtarballはキャッシュから取得できる。

### pnpm の場合

```dockerfile
FROM node:22-slim AS builder

# pnpmをインストール
RUN corepack enable && corepack prepare pnpm@10.5.0 --activate

WORKDIR /app
COPY package.json pnpm-lock.yaml ./

# pnpmストアをキャッシュマウント
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile

COPY . .
RUN pnpm run build
```

### BuildKit を有効にする

`--mount=type=cache` を使うには、BuildKitが有効になっている必要がある。

```bash
# 環境変数で有効化
DOCKER_BUILDKIT=1 docker build -t myapp .

# Docker 23.0以降はデフォルトで有効
docker buildx build -t myapp .
```

### CI/CD と Docker キャッシュマウントの組み合わせ

GitHub ActionsでDockerビルドを行う場合、キャッシュマウントの内容はデフォルトではジョブ間で共有されない。`docker/build-push-action` の `cache-from` / `cache-to` を使ってレイヤーキャッシュをGitHub Actionsのキャッシュストレージに保存する。

```yaml
- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

なお、`--mount=type=cache` で指定したキャッシュマウントの中身は `cache-from`/`cache-to` では保存されない。キャッシュマウント自体はあくまで「同一ビルダー（同一ホスト）上の複数ビルド間」で共有される仕組みであることに注意が必要だ。

## キャッシュが原因のトラブルシューティング

### 症状1: integrity checksum failed

```
npm ERR! code EINTEGRITY
npm ERR! sha512-xxxx... integrity checksum failed when using sha512:
npm ERR! wanted sha512-xxxx... but got sha512-yyyy...
```

**原因**: キャッシュ内のtarballが破損しているか、レジストリのパッケージが更新された（unpublish後の再publishなど、極めてまれ）。

**対処**:

```bash
# まずverifyを試す
npm cache verify

# 解決しなければ該当パッケージのキャッシュを再取得
# （npm v7以降は個別削除ができないため、全クリアが必要）
npm cache clean --force
npm install
```

### 症状2: ENOENT / ENOTEMPTY

```
npm ERR! code ENOENT
npm ERR! syscall rename
npm ERR! path ~/.npm/_cacache/tmp/...
```

**原因**: キャッシュの一時ファイルが残った状態で別のnpmプロセスが同時実行された、またはファイルシステムの問題。

**対処**:

```bash
# 一時ファイルのクリーンアップ
npm cache verify

# 解決しなければ
rm -rf node_modules package-lock.json
npm cache clean --force
npm install
```

### 症状3: npm ci がローカルでは速いのにCIで遅い

**原因**: CIでキャッシュが効いていない。

**確認方法**:

```bash
# npm ciの詳細ログを出力
npm ci --timing

# キャッシュヒット率を確認（npm v7+）
# reifying: ... のログでcachedか否かがわかる
npm ci --loglevel verbose 2>&1 | grep -c "cache hit"
```

**対処**: 前述のCI/CDキャッシュ戦略を確認し、`~/.npm` が正しくキャッシュ・復元されているか確認する。

### 症状4: 古いバージョンのパッケージがインストールされる

**原因**: npmキャッシュが原因であることは**ほぼない**。cacacheはintegrityハッシュで管理されており、lockfileが指定するバージョンと異なるtarballがインストールされることは構造上起こらない。

**本当の原因**: `package-lock.json` が古い、`node_modules` に以前のバージョンが残っている、またはnpmrcの `registry` 設定がプライベートレジストリを指していてそちらに古いバージョンがある。

**対処**:

```bash
# lockfileの該当パッケージのバージョンを確認
npm ls <package-name>

# クリーンインストール
rm -rf node_modules
npm ci
```

### トラブルシューティングのフローチャート

```
npm install でエラーが出た
  │
  ├── EINTEGRITY → npm cache verify → 解決しなければ npm cache clean --force
  │
  ├── ENOENT/ENOTEMPTY → npm cache verify → 解決しなければ
  │                      rm -rf node_modules && npm cache clean --force
  │
  ├── CIが遅い → setup-nodeのcache設定確認 → restore-keys設定
  │
  └── 原因不明 → npm cache verify → rm -rf node_modules package-lock.json
                                   → npm install
```

## オフラインインストール

ネットワーク接続が不安定な環境や、セキュリティポリシーで外部レジストリへのアクセスが制限されている環境では、オフラインインストールが有用だ。

### --prefer-offline

```bash
npm install --prefer-offline
```

**キャッシュにあればキャッシュを使い、なければネットワークにフォールバック**する。完全なオフラインではないが、ネットワーク接続が不安定な場合に有用だ。

npmrcに設定として書いておくこともできる。

```ini
# .npmrc
prefer-offline=true
```

### --offline

```bash
npm install --offline
```

**ネットワークアクセスを一切行わない**。キャッシュにないパッケージがあるとエラーになる。

```
npm ERR! code ENOTCACHED
npm ERR! request to https://registry.npmjs.org/... failed:
npm ERR! cache mode is 'only-if-cached' but no cached response is available.
```

このモードを使うには、事前にすべてのパッケージがキャッシュに存在している必要がある。

### オフラインインストールの実践パターン

**パターン1: 事前キャッシュ + --prefer-offline**

```bash
# オンライン環境で事前にキャッシュを構築
npm ci

# キャッシュをコピーしてオフライン環境に持ち込む
tar czf npm-cache.tar.gz ~/.npm/_cacache/

# オフライン環境で展開
tar xzf npm-cache.tar.gz -C ~/

# オフラインインストール
npm ci --prefer-offline
```

**パターン2: Verdaccio（ローカルレジストリ）**

完全なオフライン運用が必要な場合は、[Verdaccio](https://verdaccio.org/) などのローカルレジストリプロキシを立てるのがより堅牢な方法だ。

```bash
# Verdaccioをインストール
npm install -g verdaccio

# 起動（デフォルトでnpmjs.orgをプロキシ）
verdaccio

# プロジェクトのnpmrcでローカルレジストリを指定
echo "registry=http://localhost:4873/" > .npmrc

# 一度オンラインでインストールするとVerdaccioにキャッシュされる
npm install

# 以降はVerdaccioがキャッシュしたパッケージを返す
```

### pnpm のオフラインインストール

pnpmはストアにファイル単位で保存しているため、オフラインインストールとの相性が良い。

```bash
# ストアにパッケージがあればネットワークを使わない
pnpm install --offline

# キャッシュ優先、なければネットワーク
pnpm install --prefer-offline
```

## npm キャッシュの高速化 Tips

### 1. npm ci を使う

`npm install` ではなく `npm ci` を使うことで、キャッシュの恩恵を最大限に受けられる。`npm ci` はlockfileを厳密に解釈するため、依存解決のステップをスキップできる。

```bash
# 開発環境でも、lockfileが確定しているなら
npm ci

# devDependenciesを除外してさらに高速化
npm ci --omit=dev
```

### 2. .npmrc で prefer-offline を有効化

```ini
# プロジェクトの .npmrc
prefer-offline=true
```

キャッシュにヒットすればネットワークリクエストをスキップするため、特にネットワーク遅延が大きい環境で効果的だ。

### 3. キャッシュディレクトリを高速なディスクに配置

```bash
# RAMディスクやSSDにキャッシュを配置
npm config set cache /mnt/fast-disk/.npm
```

大規模プロジェクトでキャッシュの読み書きがボトルネックになっている場合に有効だ。ただし、RAMディスクの場合は再起動でキャッシュが消えるため、用途を限定すること。

### 4. 不要なキャッシュを定期的に整理

```bash
# 月1回程度のメンテナンス
npm cache verify
```

`npm cache verify` は破損エントリの削除と孤立コンテンツのガベージコレクションを行う。定期的に実行することで、キャッシュの肥大化を防ぎ、検索性能を維持できる。

## 3つのパッケージマネージャのキャッシュ比較まとめ

| 項目 | npm | pnpm | Yarn Classic | Yarn Berry |
|---|---|---|---|---|
| キャッシュの場所 | `~/.npm/_cacache` | `~/.local/share/pnpm/store/v3` | `~/Library/Caches/Yarn/v6` | `.yarn/cache/`（プロジェクトローカル） |
| 保存方式 | Content-Addressable (tarball単位) | Content-Addressable (ファイル単位) | tarball + 展開済みファイル | zipアーカイブ |
| node_modulesへの展開 | コピー | ハードリンク | コピー | PnP (展開なし) |
| キャッシュ確認 | `npm cache verify` | `pnpm store status` | `yarn cache list` | `yarn cache list` |
| キャッシュ削除 | `npm cache clean --force` | `pnpm store prune` | `yarn cache clean` | `yarn cache clean --all` |
| オフラインインストール | `--prefer-offline` / `--offline` | `--offline` / `--prefer-offline` | `--offline` | Zero-Installs (キャッシュをコミット) |
| CI/CDキャッシュ対象 | `~/.npm` | pnpm store path | `yarn cache dir` | `.yarn/cache/` (リポジトリ内) |

## まとめ ── キャッシュを正しく理解して「消さない」運用へ

この記事で伝えたかったことは3つだ。

1. **`npm cache clean --force` は最後の手段**。まず `npm cache verify` を使う
2. **npm/pnpm/yarnのキャッシュは設計が異なる**。npmはtarball単位のキャッシュ、pnpmはファイル単位の中央ストア、Yarn Berryはプロジェクトローカルのzipアーカイブ
3. **CI/CDでは `~/.npm`（パッケージマネージャのキャッシュディレクトリ）をキャッシュする**のが安全かつ効果的

キャッシュの仕組みを正しく理解していれば、「とりあえずキャッシュを消す」という対処療法から卒業し、問題の本質を見極めた対応ができるようになる。

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説しています。
:::
