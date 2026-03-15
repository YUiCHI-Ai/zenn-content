---
title: "dependency-cruiser入門：依存関係の可視化とアーキテクチャルールの自動強制"
emoji: "🕸️"
type: "tech"
topics: ["dependencycruiser", "nodejs", "architecture", "npm", "cicd"]
published: false
---

## はじめに

プロジェクトが成長するにつれ、モジュール間の依存関係は暗黙のうちに複雑化していく。気づけば循環依存が発生し、本来依存してはならないレイヤーへの参照が混入し、使われていない孤立モジュールが放置される。こうした問題はコードレビューだけでは検出が困難であり、人間の注意力に頼った運用はスケールしない。

dependency-cruiserは、こうした依存関係の問題を**静的解析によって自動検出**し、アーキテクチャルールとして**コードベースに強制**するためのツールである。本記事では、インストールから可視化、カスタムルールの設計、CI/CD統合までを実践的に解説する。

本記事が扱う範囲は以下の通りである。

| 本記事の範囲 | 書籍の範囲 |
|:---|:---|
| dependency-cruiserの導入・設定方法 | なぜ依存管理が重要なのか（設計思想） |
| 依存関係の可視化手法 | パッケージマネージャの依存解決アルゴリズム |
| アーキテクチャルールの実装 | npm/yarn/pnpmの設計思想の違い |
| CI/CDへの組み込み | node_modulesの構造と歴史的経緯 |

WHY（なぜ依存関係が壊れるのか）の深掘りについては、書籍を参照してほしい。

## dependency-cruiserとは

[dependency-cruiser](https://github.com/sverweij/dependency-cruiser)は、JavaScript/TypeScriptプロジェクトの依存関係を静的に解析するツールである。主に以下の3つの機能を提供する。

1. **依存関係の可視化** — モジュール間の依存グラフをdot形式、HTML、JSON等で出力する
2. **ルールベースの検証** — 循環依存の禁止やレイヤー違反の検出を自動化する
3. **CI/CD統合** — ルール違反をビルドエラーとして検出し、マージを防止する

対応するモジュールシステムは幅広い。

| モジュールシステム | 対応状況 |
|:---|:---|
| CommonJS (`require`) | 対応 |
| ES Modules (`import`) | 対応 |
| TypeScript | 対応（`tsconfig.json`のpaths解決含む） |
| AMD | 対応 |
| dynamic import | 対応（`import()`式） |

2026年3月時点の最新安定版は**v17**系である。Node.js 18以上が必要となる。

## インストールと初期設定

### インストール

プロジェクトのdevDependenciesとしてインストールする。

```bash
npm install --save-dev dependency-cruiser
```

バージョンを確認する。

```bash
npx depcruise --version
# 17.x.x
```

### 初期設定ファイルの生成

対話型ウィザードで設定ファイルを生成する。

```bash
npx depcruise --init
```

以下のような質問に答えていく。

```text
? Where do your source files live? src
? Do you use a TypeScript config? Yes
? Where is your TypeScript config? tsconfig.json
? What extensions do you want to scan? .ts,.tsx,.js,.jsx
```

これにより、プロジェクトルートに`.dependency-cruiser.cjs`が生成される。

### 設定ファイルの基本構造

生成された`.dependency-cruiser.cjs`の骨格は以下のようになっている。

```javascript
/** @type {import('dependency-cruiser').IConfiguration} */
module.exports = {
  forbidden: [
    // ここにルールを定義する
  ],
  options: {
    doNotFollow: {
      path: "node_modules"
    },
    tsPreCompilationDeps: true,
    tsConfig: {
      fileName: "tsconfig.json"
    },
    enhancedResolveOptions: {
      exportsFields: ["exports"],
      conditionNames: ["import", "require", "node", "default"]
    },
    reporterOptions: {
      dot: {
        collapsePattern: "node_modules/(?:@[^/]+/[^/]+|[^/]+)"
      }
    }
  }
};
```

`forbidden`配列にルールを追加し、`options`で解析の挙動を制御する。この2つのセクションが設定の中心である。

### 初回実行

設定ファイルが生成されたら、早速解析を実行してみる。

```bash
npx depcruise src --config .dependency-cruiser.cjs
```

ルール違反がなければ何も出力されない。違反があれば、以下のような形式でレポートされる。

```text
  error no-circular: src/services/userService.ts → 
      src/repositories/userRepository.ts → 
      src/services/userService.ts

✖ 1 dependency violations (1 errors, 0 warnings, 0 informational).
```

## 依存関係の可視化

dependency-cruiserの可視化機能は、プロジェクトの依存構造を直感的に把握するのに極めて有用である。

### dot形式での出力

Graphvizのdot形式で依存グラフを出力する。

```bash
npx depcruise src --include-only "^src" --output-type dot | dot -T svg > dependency-graph.svg
```

`dot`コマンドを使うにはGraphvizのインストールが必要である。

```bash
# macOS
brew install graphviz

# Ubuntu/Debian
sudo apt-get install graphviz

# Windows (chocolatey)
choco install graphviz
```

### HTMLレポートの生成

ブラウザで閲覧可能なインタラクティブなHTMLレポートも生成できる。

```bash
npx depcruise src --include-only "^src" --output-type err-html -f dependency-report.html
```

このレポートには、ルール違反のハイライト、モジュールごとの依存数、循環依存の検出結果などが含まれる。

### フォルダレベルの集約表示

大規模プロジェクトではモジュール単位のグラフが複雑になりすぎる。フォルダレベルで集約することで、アーキテクチャの全体像を把握できる。

```bash
npx depcruise src --include-only "^src" --output-type ddot | dot -T svg > folder-dependency-graph.svg
```

`ddot`（directory dot）出力タイプを指定すると、ディレクトリ単位で集約されたグラフが生成される。

### reporterOptionsによるカスタマイズ

`.dependency-cruiser.cjs`の`reporterOptions`でグラフの見た目を制御できる。

```javascript
module.exports = {
  // ...
  options: {
    reporterOptions: {
      dot: {
        collapsePattern: "node_modules/(?:@[^/]+/[^/]+|[^/]+)",
        theme: {
          graph: {
            rankdir: "LR",    // 左から右への配置（デフォルトはTB: 上から下）
            splines: "ortho"  // 直角の矢印線
          },
          node: {
            fontSize: 11
          },
          modules: [
            {
              criteria: { source: "\\.spec\\.ts$" },
              attributes: { fillcolor: "#ccccff" }  // テストファイルを青系に
            },
            {
              criteria: { source: "^src/domain" },
              attributes: { fillcolor: "#ccffcc" }  // ドメイン層を緑系に
            }
          ],
          dependencies: [
            {
              criteria: { resolved: "^src/infrastructure" },
              attributes: { color: "#ff6666" }  // インフラ層への依存を赤に
            }
          ]
        }
      }
    }
  }
};
```

この設定により、アーキテクチャ上の層を色分けした視覚的にわかりやすいグラフが生成される。

### npm scriptsへの登録

よく使うコマンドは`package.json`に登録しておくと便利である。

```json
{
  "scripts": {
    "depcruise": "depcruise src --config .dependency-cruiser.cjs",
    "depcruise:graph": "depcruise src --include-only '^src' --output-type dot | dot -T svg > docs/dependency-graph.svg",
    "depcruise:graph:folder": "depcruise src --include-only '^src' --output-type ddot | dot -T svg > docs/folder-graph.svg",
    "depcruise:html": "depcruise src --include-only '^src' --output-type err-html -f docs/dependency-report.html"
  }
}
```

## 組み込みルールの活用

dependency-cruiserには頻出パターンに対応した組み込みルールが用意されている。`--init`で生成される設定ファイルにはいくつかのルールがデフォルトで含まれているが、ここでは主要なルールとその設定方法を解説する。

### no-circular: 循環依存の禁止

最も基本的かつ重要なルールである。モジュール間の循環依存を検出する。

```javascript
{
  name: "no-circular",
  severity: "error",
  comment: "循環依存はモジュールの独立性を損ない、テスタビリティを低下させる",
  from: {},
  to: {
    circular: true
  }
}
```

循環依存が検出された場合の出力例は以下の通りである。

```text
  error no-circular: src/services/orderService.ts → 
      src/services/paymentService.ts → 
      src/services/orderService.ts
```

### no-orphans: 孤立モジュールの検出

どこからも参照されず、他のモジュールも参照しない「孤児」モジュールを検出する。

```javascript
{
  name: "no-orphans",
  severity: "warn",
  comment: "孤立モジュールは不要コードの可能性が高い",
  from: {
    orphan: true,
    pathNot: [
      "\\.d\\.ts$",           // 型定義ファイルは除外
      "(^|/)index\\.[jt]sx?$", // indexファイルは除外
      "\\.config\\.[jt]s$"     // 設定ファイルは除外
    ]
  },
  to: {}
}
```

### not-to-dev-dep: 本番コードからdevDependenciesへの依存禁止

`src/`配下のコードが`devDependencies`に依存していないかチェックする。

```javascript
{
  name: "not-to-dev-dep",
  severity: "error",
  comment: "本番コードはdevDependenciesに依存してはならない",
  from: {
    path: "^src",
    pathNot: "\\.spec\\.[jt]sx?$"  // テストファイルは除外
  },
  to: {
    dependencyTypes: ["npm-dev"]
  }
}
```

### not-to-deprecated: 非推奨パッケージへの依存検出

`package.json`で`deprecated`フラグが立っているパッケージへの依存を検出する。

```javascript
{
  name: "not-to-deprecated",
  severity: "warn",
  comment: "非推奨パッケージは代替へ移行すべきである",
  from: {},
  to: {
    dependencyTypes: ["deprecated"]
  }
}
```

### no-duplicate-dep-types: 重複依存タイプの検出

同一パッケージが`dependencies`と`devDependencies`の両方に記載されている場合を検出する。

```javascript
{
  name: "no-duplicate-dep-types",
  severity: "warn",
  comment: "同一パッケージが複数の依存タイプに重複している",
  from: {},
  to: {
    moreThanOneDependencyType: true
  }
}
```

### 組み込みルール一覧

主要な組み込みルールを整理する。

| ルール名 | 検出対象 | 推奨severity |
|:---|:---|:---|
| `no-circular` | 循環依存 | error |
| `no-orphans` | 孤立モジュール | warn |
| `not-to-dev-dep` | 本番コードからのdevDep参照 | error |
| `no-duplicate-dep-types` | 依存タイプの重複 | warn |
| `not-to-deprecated` | 非推奨パッケージ | warn |
| `not-to-unresolvable` | 解決不能なモジュール参照 | error |
| `no-non-package-json` | package.json外のnode_modules参照 | error |

:::message
📖 dependency-cruiserが検出する依存問題の多くは、パッケージマネージャの**依存解決アルゴリズム**に起因する。node_modulesがなぜ壊れるのか、その原理からnpm/pnpm/Yarnの設計思想の違いを理解したい方は **[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)** を参照してほしい。
:::

## カスタムルールの書き方

組み込みルールだけでは、プロジェクト固有のアーキテクチャ制約を表現しきれない。dependency-cruiserのカスタムルールは、`from`と`to`の条件をパターンマッチで記述する柔軟な仕組みを持っている。

### ルールの基本構造

```javascript
{
  name: "rule-name",           // ルールの一意な名前
  severity: "error",           // error | warn | info
  comment: "ルールの説明",      // 違反時に表示される説明
  from: {
    path: "^src/domain"        // 依存元のパス条件（正規表現）
  },
  to: {
    path: "^src/infrastructure" // 依存先のパス条件（正規表現）
  }
}
```

`from`に記述したパスから`to`に記述したパスへの依存が検出された場合、指定した`severity`で報告される。

### pathとpathNot

`path`は正規表現で依存元・依存先のパスをマッチさせる。`pathNot`で除外パターンを指定できる。

```javascript
{
  name: "no-test-to-production-internals",
  severity: "warn",
  comment: "テストはpublic APIのみを使用すべきである",
  from: {
    path: "\\.spec\\.[jt]sx?$"
  },
  to: {
    path: "^src/.+/internal",
    pathNot: "index\\.[jt]sx?$"  // index経由のアクセスは許可
  }
}
```

### dependencyTypesによるフィルタリング

依存の種類（npm, local, type-onlyなど）で条件を絞り込める。

```javascript
{
  name: "no-type-only-to-runtime",
  severity: "info",
  comment: "型のみのインポートが可能な場合はtype importを推奨",
  from: {
    path: "^src"
  },
  to: {
    dependencyTypes: ["local"],
    dependencyTypesNot: ["type-only"]
  }
}
```

主要なdependencyTypesは以下の通りである。

| dependencyType | 説明 |
|:---|:---|
| `local` | プロジェクト内モジュール |
| `npm` | dependencies |
| `npm-dev` | devDependencies |
| `npm-peer` | peerDependencies |
| `npm-optional` | optionalDependencies |
| `type-only` | TypeScriptのtype import |
| `deprecated` | 非推奨パッケージ |

### severityの使い分け

3段階のseverityを目的に応じて使い分ける。

```javascript
// error: CI/CDで即座にブロックする（アーキテクチャ違反）
{ severity: "error" }

// warn: 警告として表示するが、ビルドは止めない（改善推奨）
{ severity: "warn" }

// info: 情報として表示するのみ（現状把握）
{ severity: "info" }
```

導入初期は`warn`で始め、チーム内で合意が取れたら`error`に昇格させるアプローチが実用的である。

### レイヤーアーキテクチャの依存方向を強制する

ここからが本記事の核心部分である。典型的なレイヤードアーキテクチャの依存方向を強制するルールセットを構築する。

以下のディレクトリ構造を前提とする。

```text
src/
├── presentation/    # コントローラー、ビュー
├── application/     # ユースケース、サービス
├── domain/          # エンティティ、値オブジェクト、リポジトリインターフェース
└── infrastructure/  # DB接続、外部API、リポジトリ実装
```

依存の方向は `presentation → application → domain ← infrastructure` である。domainは最も内側の層であり、外側に依存してはならない。

```javascript
// .dependency-cruiser.cjs
module.exports = {
  forbidden: [
    // domain → 外部レイヤー禁止
    {
      name: "domain-not-to-application",
      severity: "error",
      comment: "domain層はapplication層に依存してはならない",
      from: { path: "^src/domain" },
      to: { path: "^src/application" }
    },
    {
      name: "domain-not-to-presentation",
      severity: "error",
      comment: "domain層はpresentation層に依存してはならない",
      from: { path: "^src/domain" },
      to: { path: "^src/presentation" }
    },
    {
      name: "domain-not-to-infrastructure",
      severity: "error",
      comment: "domain層はinfrastructure層に依存してはならない",
      from: { path: "^src/domain" },
      to: { path: "^src/infrastructure" }
    },
    // application → presentation禁止
    {
      name: "application-not-to-presentation",
      severity: "error",
      comment: "application層はpresentation層に依存してはならない",
      from: { path: "^src/application" },
      to: { path: "^src/presentation" }
    },
    // application → infrastructure禁止（DIP: インターフェースはdomainに置く）
    {
      name: "application-not-to-infrastructure",
      severity: "error",
      comment: "application層はinfrastructure層に直接依存してはならない（DIPを適用する）",
      from: { path: "^src/application" },
      to: { path: "^src/infrastructure" }
    },
    // infrastructure → presentation禁止
    {
      name: "infrastructure-not-to-presentation",
      severity: "error",
      comment: "infrastructure層はpresentation層に依存してはならない",
      from: { path: "^src/infrastructure" },
      to: { path: "^src/presentation" }
    },
    // 循環依存禁止
    {
      name: "no-circular",
      severity: "error",
      comment: "循環依存禁止",
      from: {},
      to: { circular: true }
    }
  ],
  options: {
    doNotFollow: { path: "node_modules" },
    tsPreCompilationDeps: true,
    tsConfig: { fileName: "tsconfig.json" }
  }
};
```

このルールセットにより、以下の依存方向のみが許可される。

```text
許可される依存方向:
  presentation  → application  ✅
  presentation  → domain       ✅
  application   → domain       ✅
  infrastructure → domain      ✅
  infrastructure → application ✅

禁止される依存方向:
  domain        → application     ❌
  domain        → presentation    ❌
  domain        → infrastructure  ❌
  application   → presentation    ❌
  application   → infrastructure  ❌
  infrastructure → presentation   ❌
```

## 実践パターン

### Clean Architectureの依存方向強制

Clean Architectureの「依存性は内側にのみ向く」というルールをdependency-cruiserで強制する。

```text
src/
├── entities/         # Enterprise Business Rules
├── usecases/         # Application Business Rules
├── adapters/         # Interface Adapters
│   ├── controllers/
│   ├── gateways/
│   └── presenters/
└── frameworks/       # Frameworks & Drivers
    ├── web/
    ├── db/
    └── external/
```

```javascript
const layers = [
  { name: "entities",   path: "^src/entities" },
  { name: "usecases",   path: "^src/usecases" },
  { name: "adapters",   path: "^src/adapters" },
  { name: "frameworks", path: "^src/frameworks" }
];

// 各層は自分より外側の層に依存してはならない
const forbidden = [
  // entities（最内層）は他のどの層にも依存しない
  ...["usecases", "adapters", "frameworks"].map(outer => ({
    name: `entities-not-to-${outer}`,
    severity: "error",
    comment: `entities層は${outer}層に依存してはならない`,
    from: { path: "^src/entities" },
    to: { path: `^src/${outer}` }
  })),
  // usecasesはadapters, frameworksに依存しない
  ...["adapters", "frameworks"].map(outer => ({
    name: `usecases-not-to-${outer}`,
    severity: "error",
    comment: `usecases層は${outer}層に依存してはならない`,
    from: { path: "^src/usecases" },
    to: { path: `^src/${outer}` }
  })),
  // adaptersはframeworksに依存しない
  {
    name: "adapters-not-to-frameworks",
    severity: "error",
    comment: "adapters層はframeworks層に依存してはならない",
    from: { path: "^src/adapters" },
    to: { path: "^src/frameworks" }
  }
];

module.exports = {
  forbidden,
  options: {
    doNotFollow: { path: "node_modules" },
    tsPreCompilationDeps: true,
    tsConfig: { fileName: "tsconfig.json" }
  }
};
```

このように、JavaScriptの配列操作でルールを動的に生成できるのが`.cjs`形式の利点である。

### Feature-basedアーキテクチャのルール

機能（feature）ごとにディレクトリを切る構成では、feature間の直接的な依存を禁止し、共有モジュールを経由させるルールが有効である。

```text
src/
├── features/
│   ├── auth/
│   ├── billing/
│   ├── notification/
│   └── user/
└── shared/
    ├── lib/
    ├── types/
    └── utils/
```

```javascript
{
  name: "no-cross-feature-import",
  severity: "error",
  comment: "feature間の直接依存は禁止。shared/を経由すること",
  from: {
    path: "^src/features/([^/]+)"
  },
  to: {
    path: "^src/features/([^/]+)",
    pathNot: "^src/features/$1"  // 自分自身のfeature内は許可
  }
}
```

`$1`はfromのキャプチャグループを参照する。これにより、`auth/`から`billing/`への直接依存は禁止されるが、`auth/`内のモジュール同士の依存は許可される。

### shared/からfeatureへの逆依存禁止

```javascript
{
  name: "shared-not-to-features",
  severity: "error",
  comment: "shared/はfeature固有のコードに依存してはならない",
  from: { path: "^src/shared" },
  to: { path: "^src/features" }
}
```

### 特定のnpmパッケージの使用を制限する

セキュリティやライセンスの観点から、特定パッケージの使用を制限する場合にも使える。

```javascript
{
  name: "no-moment-js",
  severity: "error",
  comment: "moment.jsは非推奨。date-fnsまたはDay.jsを使用すること",
  from: {},
  to: {
    path: "moment",
    dependencyTypes: ["npm"]
  }
}
```

### 特定ディレクトリ配下でのみ外部APIクライアントを許可する

```javascript
{
  name: "external-api-only-in-infrastructure",
  severity: "error",
  comment: "外部APIクライアントはinfrastructure/external/でのみ使用可能",
  from: {
    path: "^src",
    pathNot: "^src/infrastructure/external"
  },
  to: {
    path: "^node_modules/(axios|node-fetch|got)"
  }
}
```

## CI/CDでの自動チェック

dependency-cruiserのルールは、手動で確認するだけでは運用が形骸化する。CI/CDパイプラインに組み込むことで、ルール違反を確実にブロックする。

### GitHub Actionsでの統合

```yaml
# .github/workflows/dependency-check.yml
name: Dependency Architecture Check

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

jobs:
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci

      - name: Run dependency-cruiser
        run: npx depcruise src --config .dependency-cruiser.cjs

      - name: Generate dependency graph
        if: github.event_name == 'pull_request'
        run: |
          sudo apt-get install -y graphviz
          npx depcruise src --include-only "^src" --output-type ddot | dot -T svg > dependency-graph.svg

      - name: Upload dependency graph
        if: github.event_name == 'pull_request'
        uses: actions/upload-artifact@v4
        with:
          name: dependency-graph
          path: dependency-graph.svg
```

この設定により、PRが作成されるたびに依存関係のチェックが自動実行される。違反がある場合、`depcruise`コマンドは非ゼロの終了コードを返すため、CIジョブが失敗する。

### PRに違反レポートをコメントとして投稿する

violation件数をPRコメントに自動投稿するステップを追加する。

```yaml
      - name: Generate violation report
        if: github.event_name == 'pull_request'
        id: violation-report
        run: |
          REPORT=$(npx depcruise src --config .dependency-cruiser.cjs --output-type err 2>&1 || true)
          if [ -n "$REPORT" ]; then
            echo "has_violations=true" >> "$GITHUB_OUTPUT"
            {
              echo "report<<DEPCRUISE_EOF"
              echo "$REPORT"
              echo "DEPCRUISE_EOF"
            } >> "$GITHUB_OUTPUT"
          else
            echo "has_violations=false" >> "$GITHUB_OUTPUT"
          fi

      - name: Comment on PR
        if: steps.violation-report.outputs.has_violations == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const report = `${{ steps.violation-report.outputs.report }}`;
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## 🕸️ Dependency Architecture Violations\n\n\`\`\`\n${report}\n\`\`\`\n\n依存関係のルール違反が検出された。修正してから再度PRを提出すること。`
            });
```

### pre-commitフックでの早期検出

CIまで待たずに、コミット前にローカルで検出する。[husky](https://github.com/typicode/husky)とlint-stagedを使う方法と、huskyのpre-commitフックで直接実行する方法がある。

```bash
# huskyのインストールと初期化
npm install --save-dev husky
npx husky init
```

`.husky/pre-commit`に以下を記述する。

```bash
#!/usr/bin/env sh
npx depcruise src --config .dependency-cruiser.cjs
```

これにより、`git commit`実行時にdependency-cruiserが走り、ルール違反があればコミットがブロックされる。

### 差分チェックによる高速化

大規模プロジェクトでは全ファイルスキャンに時間がかかる。変更されたファイルのみをチェックすることで高速化できる。

```bash
# git diffで変更ファイルを取得し、srcディレクトリ内のものだけをチェック
CHANGED_FILES=$(git diff --name-only HEAD~1 -- 'src/**/*.ts' 'src/**/*.tsx')

if [ -n "$CHANGED_FILES" ]; then
  npx depcruise $CHANGED_FILES --config .dependency-cruiser.cjs
fi
```

GitHub Actionsでの差分チェック設定は以下の通りである。

```yaml
      - name: Get changed files
        id: changed
        run: |
          FILES=$(git diff --name-only origin/${{ github.base_ref }}...HEAD -- 'src/**/*.ts' 'src/**/*.tsx' | tr '\n' ' ')
          echo "files=$FILES" >> "$GITHUB_OUTPUT"

      - name: Run dependency-cruiser on changed files
        if: steps.changed.outputs.files != ''
        run: npx depcruise ${{ steps.changed.outputs.files }} --config .dependency-cruiser.cjs
```

ただし、差分チェックは変更ファイル起点の依存のみを検査するため、削除や移動による影響の検出漏れが起きうる。定期的なフルスキャン（例: mainブランチへのpush時）と組み合わせるのが望ましい。

## monorepoでの活用

monorepo構成では、パッケージ間の依存関係の管理が単一リポジトリ以上に重要になる。dependency-cruiserはmonorepoにも対応している。

### 基本的なmonorepo構成

```text
monorepo/
├── packages/
│   ├── core/          # 共有コアロジック
│   ├── api/           # APIサーバー
│   ├── web/           # Webフロントエンド
│   └── shared-types/  # 共有型定義
├── package.json
└── .dependency-cruiser.cjs
```

### ワークスペース間の依存ルール

```javascript
module.exports = {
  forbidden: [
    // core は他のパッケージに依存してはならない（最下層）
    {
      name: "core-independence",
      severity: "error",
      comment: "core パッケージは他のワークスペースパッケージに依存してはならない",
      from: { path: "^packages/core" },
      to: { path: "^packages/(?!core)" }
    },
    // shared-types は core 以外に依存してはならない
    {
      name: "shared-types-only-depends-on-core",
      severity: "error",
      comment: "shared-types は core のみに依存可能",
      from: { path: "^packages/shared-types" },
      to: {
        path: "^packages/(?!shared-types|core)"
      }
    },
    // web から api への直接依存禁止（shared-types経由とする）
    {
      name: "no-web-to-api-direct",
      severity: "error",
      comment: "web から api への直接依存禁止。型は shared-types 経由とする",
      from: { path: "^packages/web" },
      to: { path: "^packages/api" }
    },
    // api から web への依存禁止
    {
      name: "no-api-to-web",
      severity: "error",
      comment: "api から web への依存は禁止",
      from: { path: "^packages/api" },
      to: { path: "^packages/web" }
    }
  ],
  options: {
    doNotFollow: { path: "node_modules" },
    tsPreCompilationDeps: true,
    // monorepoのルートtsconfig
    tsConfig: { fileName: "tsconfig.json" },
    enhancedResolveOptions: {
      exportsFields: ["exports"],
      conditionNames: ["import", "require", "node", "default"],
      // ワークスペースのシンボリックリンク解決
      symlinks: true
    }
  }
};
```

### パッケージ境界の強制

各パッケージはpublic API（`index.ts`や`package.json`の`exports`フィールド）のみを公開すべきであり、内部モジュールへの直接参照は禁止する。

```javascript
{
  name: "respect-package-boundaries",
  severity: "error",
  comment: "他パッケージの内部モジュールへの直接参照は禁止",
  from: {
    path: "^packages/([^/]+)"
  },
  to: {
    path: "^packages/(?!$1)[^/]+/src",  // 他パッケージのsrc/配下への直接参照
    pathNot: "^packages/(?!$1)[^/]+/src/index\\.[jt]sx?$"  // index経由は許可
  }
}
```

### monorepo全体の依存グラフ生成

```bash
# ルートから全パッケージの依存関係を可視化
npx depcruise packages --include-only "^packages" --output-type ddot | dot -T svg > docs/monorepo-deps.svg
```

パッケージ間の依存方向がフォルダレベルで一目瞭然となる。

### 各パッケージ個別のチェック

各パッケージに独自のルールファイルを置くことも可能である。

```text
packages/
├── api/
│   ├── .dependency-cruiser.cjs  # api固有のルール
│   └── src/
├── web/
│   ├── .dependency-cruiser.cjs  # web固有のルール
│   └── src/
```

```bash
# 各パッケージ内で個別実行
npx depcruise packages/api/src --config packages/api/.dependency-cruiser.cjs
npx depcruise packages/web/src --config packages/web/.dependency-cruiser.cjs
```

`package.json`のscriptsで一括実行する設定は以下の通りである。

```json
{
  "scripts": {
    "depcruise:all": "depcruise packages --config .dependency-cruiser.cjs",
    "depcruise:api": "depcruise packages/api/src --config packages/api/.dependency-cruiser.cjs",
    "depcruise:web": "depcruise packages/web/src --config packages/web/.dependency-cruiser.cjs"
  }
}
```

## まとめ

### 導入ステップ

dependency-cruiserの導入は段階的に進めるのが効果的である。以下のステップを推奨する。

| ステップ | 内容 | 期間目安 |
|:---|:---|:---|
| 1. インストール | `npm i -D dependency-cruiser && npx depcruise --init` | 10分 |
| 2. 現状把握 | 依存グラフ生成、既存の循環依存を洗い出す | 1日 |
| 3. 基本ルール導入 | `no-circular`を`warn`で追加、CI連携 | 1日 |
| 4. 既存違反の解消 | 循環依存の解消、不要モジュールの削除 | 1-2週間 |
| 5. warn → error昇格 | チームで合意後、`severity: "error"`に変更 | - |
| 6. アーキテクチャルール追加 | レイヤー依存方向の強制ルールを追加 | 1-2日 |
| 7. 継続運用 | pre-commitフック、定期的なグラフ更新 | 継続 |

### ルール設計のベストプラクティス

1. **小さく始める** — まずは`no-circular`と`not-to-dev-dep`だけで十分な効果がある
2. **warnから始めてerrorに昇格** — 既存コードに大量の違反がある場合、いきなりerrorにすると開発が止まる
3. **設定ファイルは.cjs形式** — JavaScriptの表現力を活かしてルールを動的に生成できる
4. **グラフを定期的に生成** — 数値だけでなく視覚的にアーキテクチャの健全性を確認する
5. **チームで合意する** — ルールはコードベースの憲法である。一人で決めず、チームの合意を取る
6. **例外は明示的に** — やむを得ない例外は`pathNot`で除外し、コメントに理由を記録する

dependency-cruiserは「アーキテクチャの意図をコードとして表現する」ためのツールである。人間の注意力に頼らず、機械的にルールを強制することで、プロジェクトの長期的な保守性を守ることができる。

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説している。
:::
