---
title: "Verdaccio実践ガイド：プライベートnpmレジストリの構築・認証・CI連携まで"
emoji: "🏠"
type: "tech"
topics: ["verdaccio", "npm", "nodejs", "docker", "registry"]
published: true
---

## はじめに — なぜプライベートレジストリが必要か

Node.jsでの開発規模が一定を超えると、社内共有パッケージの管理が課題になる。複数のプロジェクトで共通ロジックを使い回したい、しかしnpmの公開レジストリに社内コードを置くわけにはいかない。そこで必要になるのが**プライベートnpmレジストリ**である。

プライベートレジストリが求められる典型的なユースケースは以下の3つだ。

| ユースケース | 具体例 |
|---|---|
| **社内パッケージ共有** | UIコンポーネントライブラリ、共通ユーティリティ、APIクライアントSDKの社内配布 |
| **オフライン・閉域網開発** | セキュリティポリシーで外部接続が制限された環境での依存解決 |
| **npmプロキシ / キャッシュ** | npm公開レジストリへのリクエストを中継し、ダウンロードを高速化・障害耐性を確保 |

本記事では、これらのユースケースを**Verdaccio**で実現する方法を、インストールからCI/CD統合まで網羅的に解説する。なお、本記事は「どう構築するか（HOW）」にフォーカスしており、パッケージマネージャの内部設計や依存解決の仕組み（WHY）については末尾で紹介する書籍を参照してほしい。

---

## Verdaccioとは

[Verdaccio](https://verdaccio.org/)は、**軽量なオープンソースのプライベートnpmレジストリ**である。主な特徴を整理する。

- **npm / yarn / pnpm完全互換** — 既存のパッケージマネージャからそのまま利用可能
- **ゼロコンフィグで起動可能** — `npx verdaccio` だけで即座にレジストリが立ち上がる
- **プロキシ機能内蔵** — npmjs.orgへのリクエストを透過的に中継し、ローカルキャッシュを保持
- **プラグインアーキテクチャ** — 認証（LDAP, GitLab, AWS Cognito等）やストレージ（S3, Google Cloud Storage等）を拡張可能
- **軽量** — Node.js単体で動作し、データベース不要。ファイルシステムベースのストレージがデフォルト

2026年3月時点の最新安定版は**Verdaccio 6.x**系（v6.1）である。本記事のコード例はこのバージョンに基づく。

### Verdaccioのアーキテクチャ概要

```text
┌─────────────────────────────────────────────────┐
│                  開発者 / CI                      │
│            npm publish / npm install              │
└───────────────────┬─────────────────────────────┘
                    │  HTTP(S)
                    ▼
┌─────────────────────────────────────────────────┐
│              Verdaccio サーバー                    │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ 認証     │  │ ストレージ│  │ Uplinks       │  │
│  │ htpasswd │  │ ファイル  │  │ (npmjs proxy) │  │
│  │ LDAP     │  │ S3       │  │               │  │
│  └──────────┘  └──────────┘  └───────┬───────┘  │
└──────────────────────────────────────┼──────────┘
                                       │  HTTP(S)
                                       ▼
                              ┌─────────────────┐
                              │  registry.      │
                              │  npmjs.org      │
                              └─────────────────┘
```

---

## インストールと起動

### 方法1: npm グローバルインストール

最も手軽な方法だ。開発環境やローカル検証にはこれで十分である。

```bash
# グローバルインストール
npm install -g verdaccio@6

# 起動
verdaccio

# 出力例:
# warn  --- config file  - /home/user/.config/verdaccio/config.yaml
# warn  --- http address  - http://localhost:4873/
```

デフォルトでポート`4873`でリッスンする。ブラウザで `http://localhost:4873/` にアクセスすると、Web UIが表示される。

### 方法2: npx で一時起動

インストール不要で即座に試したい場合はこちらだ。

```bash
npx verdaccio@6
```

### 方法3: Docker

本番運用やチーム共有にはDockerが推奨される。

```bash
docker run -it --rm \
  --name verdaccio \
  -p 4873:4873 \
  -v verdaccio-storage:/verdaccio/storage \
  -v verdaccio-conf:/verdaccio/conf \
  -v verdaccio-plugins:/verdaccio/plugins \
  verdaccio/verdaccio:6
```

### 基本設定ファイル（config.yaml）

Verdaccioの動作はすべて`config.yaml`で制御する。初回起動時に自動生成されるファイルをベースにカスタマイズしていくのが定石だ。

```yaml
# config.yaml — 基本構成
storage: ./storage
plugins: ./plugins

web:
  title: "My Private Registry"
  # Web UIを無効化する場合
  # enable: false

auth:
  htpasswd:
    file: ./htpasswd
    # 新規ユーザー登録の上限（-1で無制限、0で登録禁止）
    max_users: 100

uplinks:
  npmjs:
    url: https://registry.npmjs.org/
    timeout: 30s
    maxage: 10m
    cache: true

packages:
  # スコープ付きプライベートパッケージ
  "@mycompany/*":
    access: $authenticated
    publish: $authenticated
    unpublish: $authenticated

  # その他のパッケージはnpmjsへプロキシ
  "**":
    access: $all
    publish: $authenticated
    proxy: npmjs

# サーバー設定
server:
  keepAliveTimeout: 60

listen:
  - 0.0.0.0:4873

# ログ設定
log:
  type: stdout
  format: pretty
  level: warn
```

設定ファイルの場所はOS・インストール方法によって異なる。

| 環境 | デフォルトパス |
|---|---|
| Linux (npm global) | `~/.config/verdaccio/config.yaml` |
| macOS (npm global) | `~/.config/verdaccio/config.yaml` |
| Docker | `/verdaccio/conf/config.yaml` |
| Windows | `%APPDATA%\verdaccio\config.yaml` |

---

## 認証設定

### htpasswd認証（デフォルト）

最もシンプルな認証方式である。Apache互換のhtpasswdファイルでユーザーを管理する。

```yaml
# config.yaml
auth:
  htpasswd:
    file: ./htpasswd
    max_users: 50
    algorithm: bcrypt  # デフォルト。crypt, md5, sha1も選択可能
```

ユーザーの追加は`npm adduser`コマンドで行う。

```bash
# ユーザー登録
npm adduser --registry http://localhost:4873

# Username: developer1
# Password: ********
# Email: developer1@mycompany.com
```

`max_users: 0`に設定すると新規登録を禁止できる。この場合、管理者が手動でhtpasswdファイルにユーザーを追加する必要がある。

```bash
# htpasswdファイルへの手動追加（htpasswdコマンドが必要）
htpasswd -bB ./htpasswd newuser password123
```

### LDAP連携

企業環境ではLDAP/Active Directory連携が求められることが多い。`verdaccio-ldap`プラグインを使用する。

```bash
npm install verdaccio-ldap
```

```yaml
# config.yaml
auth:
  ldap:
    type: ldap
    client_options:
      url: "ldaps://ldap.mycompany.com:636"
      searchBase: "ou=People,dc=mycompany,dc=com"
      searchFilter: "(uid={{username}})"
      # グループベースのアクセス制御
      groupDnProperty: "cn"
      groupSearchBase: "ou=Groups,dc=mycompany,dc=com"
      groupSearchFilter: "(memberUid={{username}})"
    # TLS設定
    tlsOptions:
      rejectUnauthorized: true
```

### トークンベース認証

CI/CD環境ではトークン認証が基本だ。`npm login`後に生成されるトークンを`.npmrc`に設定する。

```bash
# ログインしてトークンを取得
npm login --registry http://localhost:4873

# トークンの確認
npm token list --registry http://localhost:4873

# トークンの新規作成（read-only）
npm token create --read-only --registry http://localhost:4873
```

生成されたトークンは以下の形式で`.npmrc`に記載する。

```text
//localhost:4873/:_authToken="YOUR_TOKEN_HERE"
```

### スコープ別アクセス制御

Verdaccioの強力な機能の一つが、パッケージパターンごとにアクセス権限を細かく制御できる点である。

```yaml
# config.yaml — アクセス制御の詳細設定
packages:
  # 社内コアライブラリ: 特定ユーザーのみpublish可能
  "@mycompany/core-*":
    access: $authenticated
    publish: admin-team
    unpublish: admin-team

  # 社内ツール: 認証済みユーザーなら誰でもpublish可能
  "@mycompany/*":
    access: $authenticated
    publish: $authenticated
    unpublish: $authenticated

  # 社内フォークしたOSSパッケージ
  "@mycompany-fork/*":
    access: $all
    publish: admin-team
    proxy: npmjs

  # 公開パッケージはプロキシ経由
  "**":
    access: $all
    publish: $authenticated
    proxy: npmjs
```

アクセス制御で使える特殊変数は以下の通りだ。

| 変数 | 意味 |
|---|---|
| `$all` | 全員（未認証含む） |
| `$authenticated` | 認証済みユーザー全員 |
| `$anonymous` | 未認証ユーザー |
| `ユーザー名` | 特定のユーザー |
| `グループ名` | 特定のグループ（LDAP等で使用） |

---

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説している。
:::

---

## パッケージのpublishと取得

### レジストリの指定方法

Verdaccioにパッケージをpublishする方法は複数ある。状況に応じて使い分けるのが良い。

#### 方法1: コマンドラインで直接指定

```bash
npm publish --registry http://localhost:4873
```

#### 方法2: package.jsonに記載

```json
{
  "name": "@mycompany/shared-utils",
  "version": "1.2.0",
  "publishConfig": {
    "registry": "http://localhost:4873"
  }
}
```

#### 方法3: .npmrcに設定（推奨）

プロジェクトルートに`.npmrc`を配置する方法が最も汎用的だ。

```text
# .npmrc — プロジェクトレベル
@mycompany:registry=http://localhost:4873
//localhost:4873/:_authToken=${VERDACCIO_TOKEN}
```

この設定により、`@mycompany`スコープのパッケージはすべてVerdaccioを経由し、それ以外はデフォルトのnpmjs.orgに向かう。

### publishの実践

具体的なワークフローを見ていく。

```bash
# 1. ログイン（初回のみ）
npm login --registry http://localhost:4873 --scope=@mycompany

# 2. パッケージの初期化
mkdir my-shared-lib && cd my-shared-lib
npm init --scope=@mycompany

# 3. package.jsonの確認
cat package.json
```

```json
{
  "name": "@mycompany/my-shared-lib",
  "version": "1.0.0",
  "description": "社内共有ユーティリティライブラリ",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": [
    "dist"
  ],
  "publishConfig": {
    "registry": "http://localhost:4873",
    "access": "restricted"
  },
  "scripts": {
    "build": "tsc",
    "prepublishOnly": "npm run build"
  }
}
```

```bash
# 4. publish
npm publish

# 出力例:
# npm notice 📦  @mycompany/my-shared-lib@1.0.0
# npm notice Tarball Contents
# npm notice 1.2kB dist/index.js
# npm notice 0.3kB dist/index.d.ts
# npm notice 0.5kB package.json
# + @mycompany/my-shared-lib@1.0.0
```

### パッケージの取得（install）

```bash
# .npmrcでスコープを設定済みの場合、通常通りinstallするだけ
npm install @mycompany/my-shared-lib

# 特定バージョンの指定も通常通り
npm install @mycompany/my-shared-lib@1.0.0
```

### パッケージ情報の確認

```bash
# パッケージ情報の表示
npm info @mycompany/my-shared-lib --registry http://localhost:4873

# 公開済みバージョン一覧
npm view @mycompany/my-shared-lib versions --registry http://localhost:4873
```

### unpublish（パッケージの削除）

```bash
# 特定バージョンの削除
npm unpublish @mycompany/my-shared-lib@1.0.0 --registry http://localhost:4873

# パッケージ全体の削除（--force必須）
npm unpublish @mycompany/my-shared-lib --force --registry http://localhost:4873
```

> **注意**: unpublishの権限は`config.yaml`の`unpublish`フィールドで制御される。本番環境では管理者のみに制限することを強く推奨する。

---

## npmプロキシとしての活用

Verdaccioの最も実用的な機能の一つが**uplinks（プロキシ）機能**である。npmjs.orgへのリクエストをVerdaccio経由で中継し、ダウンロード済みパッケージをローカルにキャッシュする。

### uplinks設定

```yaml
# config.yaml — uplinks設定
uplinks:
  npmjs:
    url: https://registry.npmjs.org/
    timeout: 30s
    maxage: 10m       # メタデータキャッシュの有効期間
    max_fails: 3      # 連続失敗でフェイルオーバー
    fail_timeout: 5m  # 失敗後の再試行までの待機時間
    cache: true       # tarballのキャッシュを有効化

  # 複数のuplinksを設定可能
  github-npm:
    url: https://npm.pkg.github.com/
    timeout: 30s
    auth:
      type: bearer
      token: "ghp_xxxxxxxxxxxx"
```

### キャッシュ戦略

Verdaccioのキャッシュは2層構造になっている。

```text
┌──────────────────────────────────────────┐
│            クライアント (npm install)      │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│  Layer 1: メタデータキャッシュ              │
│  - package.json, バージョン一覧            │
│  - maxage で制御（デフォルト: 2分）         │
│  - メモリ上に保持                          │
└───────────────────┬──────────────────────┘
                    │ キャッシュミス時
                    ▼
┌──────────────────────────────────────────┐
│  Layer 2: tarball キャッシュ               │
│  - .tgz ファイル本体                       │
│  - storage ディレクトリに永続化             │
│  - 一度取得すれば以降はローカルから配信      │
└──────────────────────────────────────────┘
```

キャッシュのチューニング例を示す。

```yaml
uplinks:
  npmjs:
    url: https://registry.npmjs.org/
    # 開発環境: 頻繁にバージョン確認が必要
    maxage: 2m
    cache: true

  npmjs-production:
    url: https://registry.npmjs.org/
    # 本番ビルド環境: キャッシュを長めに設定して安定性を優先
    maxage: 60m
    cache: true
```

### オフライン対応

閉域網環境やネットワーク障害時にも動作させるための設定だ。

```yaml
uplinks:
  npmjs:
    url: https://registry.npmjs.org/
    timeout: 10s
    max_fails: 2
    fail_timeout: 10m
    cache: true

packages:
  "**":
    access: $all
    proxy: npmjs

# オフラインモードを明示的に有効化する場合
# uplinksを空にすれば完全オフライン動作になる
```

事前にキャッシュを温めておくスクリプトの例を示す。

```bash
#!/bin/bash
# warm-cache.sh — 依存パッケージを事前にキャッシュ

REGISTRY="http://localhost:4873"

# package-lock.jsonから全依存パッケージを抽出してインストール
cat package-lock.json \
  | jq -r '.packages | keys[] | select(startswith("node_modules/")) | sub("node_modules/"; "")' \
  | while read pkg; do
    echo "Caching: $pkg"
    npm cache add "$pkg" --registry "$REGISTRY" 2>/dev/null || true
  done

echo "Cache warming complete."
```

---

## Docker Composeでの本番構成

開発環境での検証が済んだら、本番環境にデプロイする。Docker Compose + Nginx リバースプロキシの構成を示す。

### ディレクトリ構成

```text
verdaccio-production/
├── docker-compose.yml
├── verdaccio/
│   └── config.yaml
├── nginx/
│   ├── nginx.conf
│   └── ssl/
│       ├── cert.pem
│       └── key.pem
└── data/
    └── storage/      # パッケージデータ永続化
```

### docker-compose.yml

```yaml
# docker-compose.yml
services:
  verdaccio:
    image: verdaccio/verdaccio:6
    container_name: verdaccio
    restart: unless-stopped
    environment:
      - VERDACCIO_PORT=4873
      - VERDACCIO_PROTOCOL=http
    volumes:
      - ./verdaccio/config.yaml:/verdaccio/conf/config.yaml:ro
      - verdaccio-storage:/verdaccio/storage
      - verdaccio-plugins:/verdaccio/plugins
    networks:
      - registry-net
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:4873/-/ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:1.27-alpine
    container_name: registry-nginx
    restart: unless-stopped
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      verdaccio:
        condition: service_healthy
    networks:
      - registry-net

volumes:
  verdaccio-storage:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./data/storage
  verdaccio-plugins:

networks:
  registry-net:
    driver: bridge
```

### Nginx リバースプロキシ設定

```text
# nginx/nginx.conf
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    upstream verdaccio {
        server verdaccio:4873;
    }

    # HTTPをHTTPSにリダイレクト
    server {
        listen 80;
        server_name registry.mycompany.com;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name registry.mycompany.com;

        ssl_certificate     /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_ciphers         HIGH:!aNULL:!MD5;

        # npm publishで大きなパッケージを扱うためにサイズ制限を緩める
        client_max_body_size 100m;

        location / {
            proxy_pass http://verdaccio;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # WebSocket対応（Web UIのライブ更新用）
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```

### 本番用config.yaml

```yaml
# verdaccio/config.yaml — 本番環境用
storage: /verdaccio/storage
plugins: /verdaccio/plugins

web:
  title: "MyCompany Private Registry"
  favicon: ""
  gravatar: true
  sort_packages: asc

auth:
  htpasswd:
    file: /verdaccio/storage/htpasswd
    max_users: 0  # 本番では新規登録を禁止

uplinks:
  npmjs:
    url: https://registry.npmjs.org/
    timeout: 30s
    maxage: 10m
    max_fails: 5
    fail_timeout: 10m
    cache: true
    agent_options:
      keepAlive: true
      maxSockets: 40
      maxFreeSockets: 10

packages:
  "@mycompany/*":
    access: $authenticated
    publish: $authenticated
    unpublish: admin-team

  "**":
    access: $all
    publish: $authenticated
    proxy: npmjs

middlewares:
  audit:
    enabled: true

server:
  keepAliveTimeout: 60

# url_prefix を設定（リバースプロキシ経由のため）
url_prefix: "https://registry.mycompany.com"

log:
  type: stdout
  format: json
  level: warn

# セキュリティ設定
security:
  api:
    jwt:
      sign:
        expiresIn: 7d
  web:
    sign:
      expiresIn: 7d
```

### 起動と動作確認

```bash
# ディレクトリの準備
mkdir -p data/storage

# 起動
docker compose up -d

# ログ確認
docker compose logs -f verdaccio

# 動作確認
curl -s https://registry.mycompany.com/-/ping
# => {}

# htpasswdにユーザーを手動追加（max_users: 0のため）
docker compose exec verdaccio htpasswd -bB /verdaccio/storage/htpasswd admin secretpass
```

---

## CI/CDとの統合

プライベートレジストリの真価が発揮されるのがCI/CDパイプラインとの統合だ。パッケージのpublishを自動化し、バージョン管理をコードベースに統合する。

### GitHub Actions

```yaml
# .github/workflows/publish.yml
name: Publish Package

on:
  push:
    tags:
      - "v*"

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "22"

      - name: Configure private registry
        run: |
          echo "@mycompany:registry=https://registry.mycompany.com/" > .npmrc
          echo "//registry.mycompany.com/:_authToken=${{ secrets.VERDACCIO_TOKEN }}" >> .npmrc

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Publish
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.VERDACCIO_TOKEN }}
```

GitHub Actionsのsecretsに`VERDACCIO_TOKEN`を登録する手順は以下の通りだ。

```bash
# ローカルでトークンを生成
npm login --registry https://registry.mycompany.com

# ~/.npmrcからトークンを確認
grep "_authToken" ~/.npmrc
# //registry.mycompany.com/:_authToken=XXXXXXXXXXXXXXXX

# このトークンをGitHub Secrets（VERDACCIO_TOKEN）に登録
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - publish

variables:
  NPM_REGISTRY: "https://registry.mycompany.com"

build:
  stage: build
  image: node:22-alpine
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/

publish:
  stage: publish
  image: node:22-alpine
  only:
    - tags
  script:
    - echo "@mycompany:registry=${NPM_REGISTRY}/" > .npmrc
    - echo "//${NPM_REGISTRY#https://}/:_authToken=${VERDACCIO_TOKEN}" >> .npmrc
    - npm publish
```

### .npmrc自動生成のベストプラクティス

CI環境での`.npmrc`生成には注意点がある。以下のスクリプトで安全に生成できる。

```bash
#!/bin/bash
# scripts/setup-npmrc.sh — CI用.npmrc生成スクリプト

set -euo pipefail

REGISTRY_URL="${VERDACCIO_REGISTRY:-https://registry.mycompany.com}"
AUTH_TOKEN="${VERDACCIO_TOKEN:?Error: VERDACCIO_TOKEN is not set}"

# URLからプロトコルを除去
REGISTRY_HOST="${REGISTRY_URL#https://}"
REGISTRY_HOST="${REGISTRY_HOST#http://}"

# .npmrcを生成
cat > .npmrc <<EOF
@mycompany:registry=${REGISTRY_URL}/
//${REGISTRY_HOST}/:_authToken=${AUTH_TOKEN}
always-auth=true
EOF

echo ".npmrc configured for ${REGISTRY_URL}"

# トークンがログに漏れないよう確認
if [ -f .npmrc ]; then
  echo ".npmrc created successfully ($(wc -l < .npmrc) lines)"
fi
```

### バージョン管理の自動化

CI/CDと組み合わせて、セマンティックバージョニングを自動化する構成例だ。

```json
{
  "scripts": {
    "release:patch": "npm version patch && git push --follow-tags",
    "release:minor": "npm version minor && git push --follow-tags",
    "release:major": "npm version major && git push --follow-tags"
  }
}
```

```bash
# パッチリリース（1.0.0 → 1.0.1）
npm run release:patch

# git tagがpushされ、CIがpublishを自動実行
```

---

## pnpm / yarnとの連携

Verdaccioはnpm互換のレジストリであるため、pnpmやyarnからもそのまま利用可能だ。ただし、パッケージマネージャごとにregistry設定の方法が若干異なる。

### pnpmでの設定

```text
# .npmrc（pnpmも.npmrcを参照する）
@mycompany:registry=https://registry.mycompany.com/
//registry.mycompany.com/:_authToken=${VERDACCIO_TOKEN}
```

```bash
# pnpmでのinstallとpublish
pnpm install @mycompany/shared-utils
pnpm publish --registry https://registry.mycompany.com
```

pnpm特有の注意点として、`pnpm-workspace.yaml`を使ったモノレポ構成では、ワークスペースルートの`.npmrc`が全パッケージに適用される。

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
```

```text
# プロジェクトルート/.npmrc
@mycompany:registry=https://registry.mycompany.com/
//registry.mycompany.com/:_authToken=${VERDACCIO_TOKEN}
link-workspace-packages=true
```

### yarn (v1 Classic)での設定

```text
# .npmrc
@mycompany:registry=https://registry.mycompany.com/
//registry.mycompany.com/:_authToken=${VERDACCIO_TOKEN}
```

```bash
# .yarnrcでも設定可能
echo '"@mycompany:registry" "https://registry.mycompany.com/"' >> .yarnrc

# yarnでのinstallとpublish
yarn add @mycompany/shared-utils
yarn publish --registry https://registry.mycompany.com
```

### yarn (v4 Berry)での設定

yarn v4では`.yarnrc.yml`での設定が必要だ。

```yaml
# .yarnrc.yml
npmScopes:
  mycompany:
    npmRegistryServer: "https://registry.mycompany.com"
    npmAuthToken: "${VERDACCIO_TOKEN}"
    npmAlwaysAuth: true

# publish設定
npmPublishRegistry: "https://registry.mycompany.com"
```

```bash
# yarn v4でのinstallとpublish
yarn add @mycompany/shared-utils
yarn npm publish
```

### パッケージマネージャ別の設定比較

| 設定項目 | npm | pnpm | yarn v1 | yarn v4 |
|---|---|---|---|---|
| 設定ファイル | `.npmrc` | `.npmrc` | `.npmrc` / `.yarnrc` | `.yarnrc.yml` |
| スコープ設定 | `@scope:registry=URL` | `@scope:registry=URL` | `@scope:registry=URL` | `npmScopes.scope.npmRegistryServer` |
| 認証トークン | `//host/:_authToken=` | `//host/:_authToken=` | `//host/:_authToken=` | `npmScopes.scope.npmAuthToken` |
| publish | `npm publish` | `pnpm publish` | `yarn publish` | `yarn npm publish` |
| 環境変数展開 | `${VAR}` | `${VAR}` | 非対応 | `${VAR}` |

> **注意**: yarn v1の`.npmrc`では環境変数展開（`${VAR}`）がサポートされていない。CI環境ではスクリプトで`.npmrc`を動的に生成する必要がある。

---

## まとめ — Verdaccio vs npm Enterprise vs GitHub Packages

最後に、プライベートレジストリの選択肢を比較する。

### 機能比較

| 項目 | Verdaccio | GitHub Packages | npm Teams (npmjs) |
|---|---|---|---|
| **料金** | 無料（OSS） | Free枠あり / Team $4/user/月 | $7/user/月 |
| **ホスティング** | セルフホスト | GitHub管理 | npm管理 |
| **npmプロキシ** | あり | なし | なし |
| **オフライン対応** | あり | なし | なし |
| **認証** | htpasswd / LDAP / プラグイン | GitHub Token | npm Token |
| **ストレージ** | ファイルシステム / S3 / GCS | GitHub管理 | npm管理 |
| **Web UI** | あり | GitHubと統合 | npmjs.comと統合 |
| **セットアップ難度** | 低〜中 | 低 | 低 |
| **運用負荷** | 中（自己管理） | 低 | 低 |

### どれを選ぶべきか

**Verdaccioが最適なケース**:
- npmプロキシ/キャッシュ機能が必要（閉域網・オフライン環境）
- ランニングコストを抑えたい（サーバー費用のみ）
- 既存のLDAP/ADと統合したい
- パッケージデータを完全に自社管理したい

**GitHub Packagesが最適なケース**:
- 既にGitHub Enterprise/Teamを利用しており、追加の運用負荷を避けたい
- GitHub Actionsとのシームレスな統合を優先する
- プロキシ/キャッシュ機能は不要

**npm Teams（npmjs）が最適なケース**:
- npm CLIとの最高の互換性を求める
- 自社でのインフラ運用を一切避けたい
- npmjsの既存エコシステム（セキュリティ監査等）をフル活用したい

### 運用のポイント

Verdaccioを本番運用する際に押さえておくべきポイントをまとめる。

1. **バックアップ**: storageディレクトリを定期的にバックアップする。パッケージのtarballとメタデータはすべてここに格納される
2. **監視**: `/-/ping`エンドポイントをヘルスチェックに使い、`/-/verdaccio/data/sidebar/`でパッケージ統計を確認する
3. **ストレージの拡張**: パッケージ数が増えた場合、S3プラグイン（`verdaccio-s3-storage`）への移行を検討する
4. **アップデート**: Verdaccioのメジャーバージョンアップ時はconfig.yamlの互換性を確認する。v5→v6では設定フォーマットの変更があった
5. **セキュリティ**: 本番では必ずHTTPSを有効化し、`max_users: 0`で不要なユーザー登録を防ぐ

```bash
# storageのバックアップスクリプト例
#!/bin/bash
BACKUP_DIR="/backups/verdaccio/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"
docker compose exec verdaccio tar czf - /verdaccio/storage > "$BACKUP_DIR/storage.tar.gz"
echo "Backup saved to $BACKUP_DIR/storage.tar.gz"
```

本記事で解説した内容を実践すれば、チーム規模を問わず、安全で効率的なプライベートパッケージ管理基盤を構築できる。まずはローカルで`npx verdaccio`を実行し、パッケージのpublish/installを体験するところから始めてみてほしい。

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説している。
:::

---

:::message
この記事はAI（Claude）による生成を含む。技術的な正確性については可能な限り検証しているが、最新情報は公式ドキュメントを確認してほしい。
:::
