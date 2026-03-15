---
title: ".npmrc 設定完全ガイド ── プロジェクト・ユーザー・グローバルの使い分け【2026年版】"
emoji: "⚙️"
type: "tech"
topics: ["npm", "nodejs", "javascript", "security"]
published: true
---

## 「チームメンバーの環境だけ挙動が違う」問題

「自分のマシンでは `npm install` が通るのに、同僚の環境ではエラーになる」「ローカルでは問題ないのに、CIだけレジストリの認証に失敗する」──こうしたトラブルの原因は、コードでもパッケージでもなく、`.npmrc` の設定差異であることが少なくありません。

`.npmrc` はnpmの挙動を制御する設定ファイルですが、**4つの階層**が存在し、それぞれ異なるスコープで適用されます。どの階層にどの設定を書くかを間違えると、チーム内で挙動がバラつき、再現困難なトラブルの原因になります。

この記事では `.npmrc` の仕組みを体系的に整理し、チーム開発で「全員同じ挙動」を実現するための設定テンプレートを提供します。

## .npmrcとは

`.npmrc` はnpmの設定ファイルです。**ini形式**で `key=value` の組み合わせを1行ずつ記述します。

```ini
# コメントは # または ; で始める
save-exact=true
engine-strict=true
fund=false
```

npmはこのファイルを複数の場所から読み込み、設定をマージして最終的な挙動を決定します。設定項目は `npm config` コマンドでも変更できますが、チーム開発ではファイルとしてバージョン管理に含めるのが基本です。

## 4つの階層と優先順位

`.npmrc` には4つの階層があり、番号が小さいほど優先されます。同じ設定項目が複数の階層に存在する場合、より優先度の高い値で上書きされます。

```
┌─────────────────────────────────────────────────┐
│  1. プロジェクト  ./project/.npmrc     ← 最優先  │
├─────────────────────────────────────────────────┤
│  2. ユーザー      ~/.npmrc                       │
├─────────────────────────────────────────────────┤
│  3. グローバル    $PREFIX/etc/npmrc              │
├─────────────────────────────────────────────────┤
│  4. ビルトイン    npm本体に同梱        ← 最低優先 │
└─────────────────────────────────────────────────┘

  上書きの方向: 上の階層が下の階層を上書きする
```

### 1. プロジェクトレベル（最優先）

**場所**: プロジェクトルート（`package.json` と同じディレクトリ）の `.npmrc`

```bash
my-project/
├── .npmrc          # ← これ
├── package.json
└── package-lock.json
```

**用途**: チーム全員で共有すべき設定。Gitにコミットして全メンバーに適用します。

```ini
# プロジェクトの .npmrc
save-exact=true
engine-strict=true
```

**これが最も重要な階層です**。プロジェクトの `.npmrc` に書いた設定は、そのプロジェクトで `npm` コマンドを実行するすべてのメンバーとCIに適用されます。

### 2. ユーザーレベル

**場所**: `~/.npmrc`（ホームディレクトリ直下）

```bash
# ファイルの場所を確認
npm config get userconfig
# → /Users/yourname/.npmrc
```

**用途**: 個人の認証トークンや個人的な好み。**Gitにコミットしてはいけません**。

```ini
# ~/.npmrc（個人の設定）
//registry.npmjs.org/:_authToken=npm_XXXXXXXXXXXX
init-author-name=Your Name
init-license=MIT
```

### 3. グローバルレベル

**場所**: `$PREFIX/etc/npmrc`

```bash
# ファイルの場所を確認
npm config get globalconfig
# → /usr/local/etc/npmrc（環境により異なる）
```

**用途**: そのマシン上のすべてのプロジェクト・すべてのユーザーに適用したい設定。通常はほとんど使いません。

### 4. ビルトインレベル（最低優先）

npm本体に同梱されたデフォルト設定です。ユーザーが直接編集することはありません。すべてのデフォルト値はここで定義されています。

### 優先順位の確認方法

現在の設定がどの階層から来ているか確認するには、`npm config list` を使います。

```bash
# 現在有効な設定を表示
npm config list

# すべての設定（デフォルト含む）を表示
npm config list -l

# 特定の設定値を確認
npm config get save-exact
```

出力には各設定値の横にファイルパスが表示されるため、「どの `.npmrc` から読み込まれたか」を特定できます。

## チーム開発で必須の設定項目

以下は、プロジェクトの `.npmrc` にコミットして全メンバーに適用すべき推奨設定です。そのままコピーして使えるテンプレートを用意しました。

### コピペ可能テンプレート

```ini
# ============================================
# .npmrc - チーム開発推奨設定（npm 11.10+）
# ============================================

# --- バージョン管理 ---
# package.jsonに ^5.0.0 ではなく 5.0.0 と記録する
save-exact=true

# package.jsonのengines指定を強制する
engine-strict=true

# --- セキュリティ ---
# postinstall等のライフサイクルスクリプトを無効化
ignore-scripts=true

# audit警告レベル（low/moderate/high/critical）
audit-level=moderate

# 公開3日未満のバージョンを拒否（npm 11.10+）
min-release-age=3

# --- パフォーマンス ---
# キャッシュにあるパッケージはネットワークを使わない
prefer-offline=true

# --- 表示制御 ---
# npm fund の広告を非表示
fund=false
```

各設定項目を詳しく見ていきます。

### save-exact=true

```ini
save-exact=true
```

`npm install express` を実行したとき、`package.json` にバージョンがどう記録されるかを制御します。

| 設定 | 記録されるバージョン | 意味 |
|------|---------------------|------|
| `false`（デフォルト） | `^4.21.2` | 4.x系の最新を許容 |
| `true` | `4.21.2` | この正確なバージョンのみ |

`^` はSemVerの「互換性のある更新」を許容する記法ですが、マイナーバージョンアップで非互換な変更が紛れ込むことは珍しくありません。`save-exact=true` と `package-lock.json` の併用で、「いつ `npm install` しても同じバージョンが入る」状態を強制できます。

### engine-strict=true

```ini
engine-strict=true
```

`package.json` の `engines` フィールドで指定したNode.jsバージョンを厳密にチェックします。条件を満たさない環境では `npm install` がエラーで停止します。

```json
{
  "engines": {
    "node": ">=22.0.0"
  }
}
```

チームメンバーが古いNode.jsで開発して「なぜか動かない」というトラブルを未然に防ぎます。

### ignore-scripts=true

```ini
ignore-scripts=true
```

`npm install` 時に自動実行される `preinstall`、`postinstall` などのライフサイクルスクリプトを無効化します。

2025年のShai-Hulud攻撃では、`preinstall` スクリプトを悪用してnpmトークンを窃取し、796パッケージに感染が拡大しました。デフォルトで無効化し、ネイティブモジュールのビルドが必要なパッケージだけ手動で対応するのが現在の推奨プラクティスです。

```bash
# ignore-scripts=true の状態で、必要なパッケージだけビルド
npm rebuild sharp esbuild
```

:::message
`ignore-scripts=true` の背景にある攻撃手法の詳細 -- なぜ `preinstall` スクリプトが危険なのか、Shai-Hulud攻撃の自己増殖メカニズムなど -- については、拙著 **[「なぜnode_modulesは壊れるのか？」](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)** の第9章で攻撃ベクタごとに解説しています。
:::

### audit-level=moderate

```ini
audit-level=moderate
```

`npm audit` が報告する最低深刻度を設定します。`moderate` に設定すると、`low` の脆弱性は報告されなくなり、アラート疲れを軽減できます。

| 値 | 報告される深刻度 |
|----|----------------|
| `low` | low, moderate, high, critical |
| `moderate` | moderate, high, critical |
| `high` | high, critical |
| `critical` | criticalのみ |

### min-release-age=3（npm 11.10+）

```ini
min-release-age=3
```

公開からの経過日数が指定値未満のバージョンを、インストール対象から除外します。値の単位は**日数**です。`min-release-age=3` は「公開から3日未満のバージョンを拒否する」という意味になります。

2025年9月のchalk侵害事件では、悪意あるバージョンは公開から約2時間で除去されました。3日間のバッファがあれば、そもそもインストール対象にならなかったことになります。

```bash
# min-release-ageが効いているか確認
npm install --dry-run
# → 最新バージョンが3日未満の場合、それ以前のバージョンが選択される
```

**注意**: この設定は `--before` オプションとは併用できません。また、`~`（チルダ）バージョン範囲と組み合わせた場合に問題が報告されています（npm/cli#9005）。

### prefer-offline=true

```ini
prefer-offline=true
```

ローカルキャッシュにパッケージが存在し、SemVer範囲を満たす場合、レジストリへのネットワークリクエストをスキップします。2回目以降の `npm install` が目に見えて速くなります。

### fund=false

```ini
fund=false
```

`npm install` 後に表示される「N packages are looking for funding」メッセージを非表示にします。パッケージへの寄付は重要ですが、CIログやターミナル出力がノイズで埋まるのを防ぎます。

## 社内レジストリとDependency Confusion対策

社内（プライベート）パッケージを使っている場合、スコープ別レジストリの設定は**セキュリティ上必須**です。

### Dependency Confusionとは

攻撃者が社内パッケージと同名のパッケージを公開npmレジストリに登録し、設定の隙を突いて悪意あるコードをインストールさせる攻撃です。スコープ別レジストリを明示設定していなければ、この攻撃が成立し得ます。

### スコープ別レジストリの設定

```ini
# プロジェクトの .npmrc
# @mycompany スコープは社内レジストリから取得
@mycompany:registry=https://npm.internal.example.com/

# その他のパッケージは公開レジストリから取得（デフォルト）
# registry=https://registry.npmjs.org/  ← 明示しなくてもデフォルト
```

この設定により、`@mycompany/auth` や `@mycompany/utils` は社内レジストリからのみ取得され、公開レジストリにたとえ同名パッケージが存在しても参照されません。

### 社内レジストリの認証

社内レジストリへの認証トークンは、プロジェクトの `.npmrc` ではなく環境変数経由で渡すのが安全です。

```ini
# プロジェクトの .npmrc（Gitにコミットする）
@mycompany:registry=https://npm.internal.example.com/
//npm.internal.example.com/:_authToken=${INTERNAL_NPM_TOKEN}
```

```bash
# ローカル開発（~/.bashrc や ~/.zshrc に追加）
export INTERNAL_NPM_TOKEN="your-token-here"

# CI環境（GitHub Actionsの場合）
# Settings → Secrets → INTERNAL_NPM_TOKEN に設定
```

## 認証トークンの安全な管理

npmレジストリへの認証トークンの管理は、セキュリティ上の最重要事項の一つです。

### やってはいけないこと

```ini
# 危険: トークンをプロジェクトの.npmrcに直書き
# このファイルをGitにコミットするとトークンが漏洩する
//registry.npmjs.org/:_authToken=npm_XXXXXXXXXXXX
```

### 安全な方法: 環境変数での参照

```ini
# プロジェクトの .npmrc（Gitにコミットして良い）
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```

`${NPM_TOKEN}` はnpmが実行時に環境変数から展開します。トークンの実体は `.npmrc` ファイルに含まれません。

### ユーザーレベルの.npmrcに書く場合

個人の開発マシンでは `~/.npmrc` にトークンを書くのが一般的です。

```bash
# npm login コマンドで自動設定される
npm login

# 手動で設定する場合
npm config set //registry.npmjs.org/:_authToken=npm_XXXXXXXXXXXX
# → ~/.npmrc に書き込まれる
```

**注意点**:
- `~/.npmrc` は `.gitignore` に含める必要がないファイルです（プロジェクト外にあるため）
- ただし、ドットファイルのバックアップ時にトークンが含まれていないか注意してください
- npm 11ではClassic Tokenが失効済み（2025年12月revoked）であり、Granular Access Tokenへの移行が必須です

### CI環境でのシークレット管理

```yaml
# GitHub Actions
jobs:
  install:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          registry-url: https://registry.npmjs.org
      - run: npm ci
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

`actions/setup-node` の `registry-url` を指定すると、自動的に `.npmrc` が生成され `NODE_AUTH_TOKEN` 環境変数のトークンが参照されます。シークレットをYAMLファイルにハードコードする必要はありません。

## pnpmの.npmrc設定

pnpmは `.npmrc` の設定を読み込みますが、pnpm固有の設定項目が多数存在します。npm用の `.npmrc` と同居可能ですが、npmが認識しない設定項目については警告が出る場合があります（npm v11.2+）。

### pnpm固有の主要設定

```ini
# pnpmの .npmrc
# --- ホイスティング制御 ---
# node_modulesをフラット化する（非推奨だが互換性のために必要な場合がある）
shamefully-hoist=false

# 公開パッケージをルートのnode_modulesに引き上げるパターン
# デフォルト: ['*eslint*', '*prettier*']
public-hoist-pattern[]=*eslint*
public-hoist-pattern[]=*prettier*

# --- peer dependency ---
# 不正なpeer dependencyでエラーにする
strict-peer-dependencies=false

# peer dependencyを自動インストール
auto-install-peers=true

# --- リンク方式 ---
# isolated（デフォルト）/ hoisted / pnp
node-linker=isolated
```

### shamefully-hoist

```ini
shamefully-hoist=true
```

pnpmの最大の特徴である厳密な依存分離を無効化し、npm互換のフラットな `node_modules` を生成します。「shamefully（恥ずかしながら）」という名前が示す通り、**互換性のためのやむを得ない設定**です。

特定のツール（一部のESLintプラグイン、React Nativeなど）がフラットな `node_modules` を前提としている場合にのみ使います。新規プロジェクトでは `false`（デフォルト）のまま運用し、問題が出たパッケージだけ `public-hoist-pattern` で個別対応するのが推奨です。

### strict-peer-dependencies

```ini
strict-peer-dependencies=true
```

peer dependencyの不整合をエラーとして扱います。デフォルトは `false`（警告のみ）です。

`true` に設定すると、peer dependencyのバージョンが合わないパッケージがあればインストールが停止します。厳格に管理したいチームでは有効ですが、サードパーティライブラリのpeer dependency宣言が緩いケースも多いため、段階的に導入するのが現実的です。

### minimumReleaseAge（pnpm-workspace.yaml）

pnpmでは `minimumReleaseAge` は `.npmrc` ではなく `pnpm-workspace.yaml` に設定します。**単位は分**です。

```yaml
# pnpm-workspace.yaml
minimumReleaseAge: 4320   # 3日 = 4320分
```

npmの `min-release-age`（日数）とは単位が異なるため、混同に注意してください。

| パッケージマネージャ | 設定名 | 単位 | 3日間の設定値 |
|---------------------|--------|------|-------------|
| npm 11.10+ | `min-release-age` | 日 | `3` |
| pnpm | `minimumReleaseAge` | 分 | `4320` |

## よくあるトラブルと対処法

### 1. 設定が効いていない

**症状**: `.npmrc` に書いた設定が反映されない。

**原因**: より優先度の高い階層の設定で上書きされている可能性があります。

```bash
# 全階層の設定を確認
npm config list

# 特定の設定値がどこから来ているか確認
npm config list -l | grep save-exact
```

**チェックポイント**:
- `.npmrc` ファイルがプロジェクトルート（`package.json` と同じ階層）にあるか
- `~/.npmrc` に同じ設定項目が異なる値で書かれていないか
- 設定値の前後に余分なスペースがないか（`save-exact = true` ではなく `save-exact=true`）

### 2. CIで認証エラーが発生する

**症状**: ローカルでは `npm install` が通るが、CIで `401 Unauthorized` になる。

```
npm ERR! code E401
npm ERR! 401 Unauthorized - GET https://npm.internal.example.com/@mycompany/auth
```

**原因**: ローカルの `~/.npmrc` に認証トークンがあるが、CI環境には存在しない。

**対処**:

```ini
# プロジェクトの .npmrc に環境変数参照を追加
//npm.internal.example.com/:_authToken=${INTERNAL_NPM_TOKEN}
```

```yaml
# CI（GitHub Actions）でシークレットを渡す
- run: npm ci
  env:
    INTERNAL_NPM_TOKEN: ${{ secrets.INTERNAL_NPM_TOKEN }}
```

### 3. npmが.npmrcの未知の設定を警告する

**症状**: npm v11.2以降で、pnpm固有の設定項目に対して警告が出る。

```
npm warn config unknown config "shamefully-hoist"
```

**原因**: npm v11.2以降は未知の設定キーに対して警告を出します。pnpm固有の設定がnpm実行時にも読み込まれてしまうためです。

**対処**: npmとpnpmを同一プロジェクトで併用している場合は仕方ありませんが、パッケージマネージャを統一するのが根本対策です。`package.json` の `packageManager` フィールドとCorepack（またはVolta）でバージョンを固定してください。

```json
{
  "packageManager": "pnpm@10.12.1"
}
```

### 4. npmのグローバルインストールで権限エラーが出る

**症状**: `npm install -g` で `EACCES` エラーが出る。

**原因**: グローバルの `$PREFIX` ディレクトリに書き込み権限がない。

**対処**: `sudo` を使うのではなく、Node.jsバージョンマネージャ（Volta、nvm）を使って `$PREFIX` をユーザー権限内に置きます。

```bash
# Voltaを使う場合
volta install node@22
# → $PREFIX が ~/.volta/tools/ 配下になり、sudo不要
```

### 5. npm config list の見方

トラブルシューティングの基本は `npm config list` です。出力の読み方を押さえましょう。

```bash
$ npm config list

; "project" config from /path/to/project/.npmrc
save-exact = true

; "user" config from /Users/yourname/.npmrc
//registry.npmjs.org/:_authToken = (protected)

; "default" config (builtin)
registry = "https://registry.npmjs.org/"
```

各セクションのヘッダーが設定の出所を示しています。`"project"` はプロジェクトの `.npmrc`、`"user"` は `~/.npmrc`、`"default"` はビルトインのデフォルト値です。認証トークンは `(protected)` と表示され、値が直接出力されることはありません。

## まとめ: プロジェクトの.npmrcテンプレート

最後に、この記事で解説した設定を統合したテンプレートを掲載します。プロジェクトルートに `.npmrc` として配置し、Gitにコミットしてください。

```ini
# ============================================
# .npmrc - プロジェクト設定テンプレート
# npm 11.10+ 対応
# ============================================

# --- バージョン管理 ---
save-exact=true
engine-strict=true

# --- セキュリティ ---
ignore-scripts=true
audit-level=moderate
min-release-age=3

# --- パフォーマンス ---
prefer-offline=true

# --- 表示制御 ---
fund=false

# --- 社内レジストリ（必要な場合のみ） ---
# @mycompany:registry=https://npm.internal.example.com/
# //npm.internal.example.com/:_authToken=${INTERNAL_NPM_TOKEN}
```

`.npmrc` の4つの階層を正しく理解し、「チーム共有の設定はプロジェクトレベル」「認証トークンはユーザーレベルまたは環境変数」という原則を守ることで、環境差異によるトラブルの大半を防げます。

この記事では.npmrcの「設定方法」にフォーカスしました。`ignore-scripts`や`min-release-age`がなぜ必要なのか、サプライチェーン攻撃の実際の攻撃ベクタとその防御策を体系的に理解したい方は、拙著 **[「なぜnode_modulesは壊れるのか？」](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)** の第9章をご覧ください。

---

*この記事はAIの支援を受けて執筆されています。*
