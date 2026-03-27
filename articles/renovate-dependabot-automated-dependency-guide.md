---
title: "Renovate / Dependabot 実践ガイド：自動依存関係更新でセキュリティと開発速度を両立する"
emoji: "🔄"
type: "tech"
topics: ["renovate", "dependabot", "npm", "nodejs", "cicd"]
published: true
---

## はじめに — なぜ自動依存関係更新が必要か

Node.jsプロジェクトの依存関係は、放置すると急速に腐敗する。`npm audit`を半年ぶりに実行して、数十件のvulnerabilityに絶望した経験は多くのエンジニアにあるだろう。

手動での依存関係更新には、以下の構造的な限界がある。

| 問題 | 影響 |
|------|------|
| 更新頻度の不足 | 脆弱性が数週間〜数ヶ月放置される |
| 変更差分の肥大化 | まとめて上げるほどbreaking changeのリスクが増大する |
| 認知負荷 | 「どのパッケージをいつ上げるか」の判断が属人化する |
| セキュリティSLAの未達 | critical脆弱性の対応に明確な期限がない |

自動依存関係更新ツールは、これらの問題を仕組みで解決する。本記事では、GitHub標準の**Dependabot**と、より高機能な**Renovate**の2つを取り上げ、実践的な設定パターンを網羅的に解説する。

なお、本記事は**HOW（どう設定し運用するか）**にフォーカスしている。「なぜ依存解決は複雑なのか」「npm / pnpm / Yarnの設計思想の違い」といったWHYの部分に興味がある方は、末尾で紹介している書籍を参照してほしい。

---

## Dependabot入門 — GitHub標準の自動更新

### Dependabotとは

DependabotはGitHubに組み込まれた依存関係の自動更新サービスである。リポジトリに設定ファイルを1つ追加するだけで、以下の2つの機能が有効になる。

- **バージョン更新（Version Updates）**: 定期的に依存パッケージの新バージョンを検出し、PRを自動生成する
- **セキュリティアラート（Security Updates）**: GitHub Advisory Databaseに基づき、脆弱性のあるパッケージのPRを即座に生成する

### 基本設定

`.github/dependabot.yml`をリポジトリのルートに配置する。

```yaml
# .github/dependabot.yml
version: 2
updates:
  # npm（package.json）の依存関係を管理
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Tokyo"
    open-pull-requests-limit: 10
    reviewers:
      - "your-team/frontend"
    labels:
      - "dependencies"
      - "automated"
```

### 主要な設定項目

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"      # daily / weekly / monthly
    
    # PRの上限数（デフォルト: 5）
    open-pull-requests-limit: 15
    
    # バージョン戦略
    versioning-strategy: "increase"  # increase / increase-if-necessary / 
                                      # lockfile-only / widen / auto
    
    # 特定パッケージの無視
    ignore:
      - dependency-name: "express"
        versions: ["5.x"]        # Express 5.xへの更新を無視
      - dependency-name: "aws-sdk"
        update-types: ["version-update:semver-major"]  # メジャー更新のみ無視
    
    # コミットメッセージのカスタマイズ
    commit-message:
      prefix: "deps"
      prefix-development: "dev-deps"
      include: "scope"
```

### グルーピング機能（2023年8月GA）

Dependabotにもグルーピング機能が追加されている。関連パッケージをまとめて1つのPRにできる。

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      # ESLint関連を1つのPRにまとめる
      eslint:
        patterns:
          - "eslint*"
          - "@typescript-eslint/*"
        update-types:
          - "minor"
          - "patch"
      
      # AWS SDK v3をまとめる
      aws-sdk:
        patterns:
          - "@aws-sdk/*"
      
      # テスト関連をまとめる
      testing:
        patterns:
          - "jest*"
          - "@jest/*"
          - "ts-jest"
          - "@types/jest"
      
      # その他のminor/patchをまとめる
      minor-and-patch:
        patterns:
          - "*"
        update-types:
          - "minor"
          - "patch"
```

### セキュリティアラートの設定

セキュリティアップデートはデフォルトで有効だが、明示的に設定することもできる。GitHub リポジトリの Settings > Code security で以下を有効にする。

- **Dependabot alerts**: 脆弱性の検知と通知
- **Dependabot security updates**: 脆弱性修正PRの自動生成

---

## Renovate入門 — 高機能な依存関係管理

### Renovateとは

RenovateはMend（旧WhiteSource）が開発するオープンソースの依存関係更新ツールである。Dependabotと比較して、以下の点で優れている。

- 200以上のパッケージマネージャ / データソースに対応
- 高度なグルーピング・スケジューリング
- automerge機能の柔軟な制御
- セルフホスト可能

### Mend Renovate Appの導入手順

最も手軽な導入方法は、GitHub Appとしてインストールする方法である。

**手順1**: [Mend Renovate App](https://github.com/apps/renovate) にアクセスし、「Install」をクリック

**手順2**: インストール対象のリポジトリを選択（全リポジトリ or 個別選択）

**手順3**: インストール完了後、Renovateが自動的に「Configure Renovate」というオンボーディングPRを生成する

**手順4**: オンボーディングPRの内容を確認してマージすると、`renovate.json`がリポジトリに追加される

### 基本設定

```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended"
  ]
}
```

`config:recommended`は以下のプリセットを含む合理的なデフォルト設定である。

```text
config:recommended に含まれるプリセット:
- :dependencyDashboard        → Issue でダッシュボード管理
- :semanticPrefixFixDepsChoreOthers → Conventional Commits対応
- :ignoreModulesAndTests      → node_modules, test fixtures を除外
- :prHourlyLimit2             → 1時間あたりPR上限2
- :prConcurrentLimit10        → 同時PR上限10
- group:monorepos             → 既知のmonorepoを自動グルーピング
- group:recommended           → 推奨グルーピング
- replacements:all            → 既知のパッケージ置換ルール
- workarounds:all             → 既知の問題への対応
```

### よく使う設定オプション

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  
  "timezone": "Asia/Tokyo",
  "schedule": ["before 9am on monday"],
  
  "labels": ["dependencies", "renovate"],
  "reviewers": ["team:platform"],
  
  "prHourlyLimit": 3,
  "prConcurrentLimit": 15,
  
  "packageRules": [
    {
      "description": "patch/minorは自動マージ",
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true
    },
    {
      "description": "メジャーアップデートにラベルを付与",
      "matchUpdateTypes": ["major"],
      "labels": ["dependencies", "breaking-change"],
      "automerge": false
    }
  ]
}
```

---

## Dependabot vs Renovate 比較

### 機能比較表

| 機能 | Dependabot | Renovate |
|------|-----------|----------|
| **提供形態** | GitHub組み込み | GitHub App / セルフホスト |
| **対応プラットフォーム** | GitHub のみ | GitHub / GitLab / Bitbucket / Gitea 等 |
| **対応エコシステム数** | 約20 | 200以上 |
| **グルーピング** | あり（groups設定） | あり（より柔軟） |
| **automerge** | GitHub native automerge経由 | 組み込みサポート |
| **スケジュール制御** | daily / weekly / monthly | cron的な柔軟な指定 |
| **monorepo対応** | directory個別指定 | 自動検出 + グルーピング |
| **Dependency Dashboard** | なし | あり（Issue形式） |
| **Conventional Commits** | prefix設定 | プリセットで自動対応 |
| **ロックファイル更新** | あり | あり |
| **セキュリティアラート統合** | GitHub Advisory Database | 複数ソース対応 |
| **設定ファイル** | `.github/dependabot.yml` | `renovate.json` 等 |
| **設定の共有** | なし | プリセット機能あり |
| **PR本文の情報量** | 基本的 | Release Notes / Changelog 詳細表示 |

### どちらを選ぶべきか

**Dependabotが適するケース:**
- GitHubのみで完結するプロジェクト
- 設定をシンプルに保ちたい
- セキュリティアラートとの統合を重視する
- 追加のGitHub App導入が制限されている環境

**Renovateが適するケース:**
- monorepoで複雑なグルーピングが必要
- automergeを積極的に活用したい
- GitHub以外のプラットフォームでも統一したい
- 組織全体で設定をプリセットとして共有したい
- Dependency Dashboardで更新状況を一元管理したい

### 併用について

DependabotとRenovateの併用は推奨しない。同じ依存関係に対して重複したPRが生成され、混乱の原因になる。どちらか一方を選択し、統一すべきである。

セキュリティアラートのみDependabot（GitHub標準の通知機能として）、バージョン更新はRenovateという組み合わせは成立する。その場合、Dependabotの`version updates`は無効にし、`security updates`のみ利用する。

:::message
📖 この記事ではRenovate/Dependabotの**設定方法（HOW）**を解説した。npm・pnpm・Yarnがなぜ異なる依存解決の仕組みを持つのか、その**設計思想の違い（WHY）**を知りたい方は **[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)** を参照してほしい。
:::

---

## 実践的な設定パターン

### パターン1: monorepo対応（Renovate）

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "ignorePaths": ["**/fixtures/**", "**/examples/**"],
  "packageRules": [
    {
      "description": "apps/web 固有の設定",
      "matchFileNames": ["apps/web/**"],
      "labels": ["dependencies", "web"],
      "reviewers": ["team:web-frontend"]
    },
    {
      "description": "apps/api 固有の設定",
      "matchFileNames": ["apps/api/**"],
      "labels": ["dependencies", "api"],
      "reviewers": ["team:backend"]
    },
    {
      "description": "packages/ は全チームレビュー",
      "matchFileNames": ["packages/**"],
      "labels": ["dependencies", "shared"],
      "reviewers": ["team:platform"]
    }
  ]
}
```

### パターン2: monorepo対応（Dependabot）

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/apps/web"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "web"
  
  - package-ecosystem: "npm"
    directory: "/apps/api"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "api"
  
  - package-ecosystem: "npm"
    directory: "/packages/shared"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "shared"
```

### パターン3: グルーピング戦略（Renovate）

```json
{
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "description": "TypeScript関連を1つのPRにまとめる",
      "matchPackageNames": ["typescript", "@typescript-eslint/**"],
      "groupName": "TypeScript",
      "groupSlug": "typescript"
    },
    {
      "description": "React関連をまとめる",
      "matchPackageNames": ["react", "react-dom", "@types/react", "@types/react-dom"],
      "groupName": "React",
      "groupSlug": "react"
    },
    {
      "description": "Next.js関連をまとめる",
      "matchPackageNames": ["next", "@next/**"],
      "groupName": "Next.js",
      "groupSlug": "nextjs"
    },
    {
      "description": "Linter / Formatter をまとめる",
      "matchPackageNames": ["eslint", "eslint-*", "@eslint/**", "prettier", "@prettier/**"],
      "groupName": "Linters and Formatters",
      "groupSlug": "linters-formatters"
    },
    {
      "description": "テスト関連をまとめる",
      "matchPackageNames": ["jest", "jest-*", "@jest/**", "vitest", "@vitest/**", "ts-jest", "@types/jest", "@testing-library/**"],
      "groupName": "Testing",
      "groupSlug": "testing"
    },
    {
      "description": "AWS SDK v3をまとめる",
      "matchPackageNames": ["@aws-sdk/**"],
      "groupName": "AWS SDK v3",
      "groupSlug": "aws-sdk-v3"
    },
    {
      "description": "残りのpatch更新をまとめる",
      "matchUpdateTypes": ["patch"],
      "excludePackageNames": [
        "@typescript-eslint/**", "react", "@types/react",
        "next", "@next/**", "eslint", "eslint-*", "prettier",
        "jest", "jest-*", "@jest/**", "vitest", "@vitest/**",
        "@testing-library/**", "@aws-sdk/**"
      ],
      "groupName": "Non-major (patch)",
      "groupSlug": "non-major-patch"
    }
  ]
}
```

### パターン4: automergeの段階的設定（Renovate）

```json
{
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "description": "devDependenciesのpatch/minorは即automerge",
      "matchDepTypes": ["devDependencies"],
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true,
      "automergeType": "branch",
      "automergeSchedule": ["before 6am on monday"]
    },
    {
      "description": "productionのpatchのみautomerge（CIパス必須）",
      "matchDepTypes": ["dependencies"],
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "pr",
      "platformAutomerge": true
    },
    {
      "description": "productionのminorは手動レビュー",
      "matchDepTypes": ["dependencies"],
      "matchUpdateTypes": ["minor"],
      "automerge": false,
      "reviewers": ["team:platform"]
    },
    {
      "description": "メジャーアップデートは常に手動レビュー",
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "labels": ["breaking-change"],
      "reviewers": ["team:platform", "team:lead"]
    }
  ]
}
```

`automergeType`の違いは以下の通りである。

| automergeType | 動作 | 適用場面 |
|---------------|------|----------|
| `"branch"` | PRを作成せずブランチを直接マージ | ノイズを最小化したい場合 |
| `"pr"` | PRを作成し、CI通過後にマージ | 履歴を残したい場合 |

### パターン5: メジャーアップデートの扱い

メジャーアップデートは別PRとして分離し、明確なレビューフローに載せるのが鉄則である。

```json
{
  "packageRules": [
    {
      "description": "メジャーアップデートは個別PRで管理",
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "separateMajorMinor": true,
      "labels": ["dependencies", "major", "needs-migration-check"],
      "prBodyNotes": [
        "## :warning: メジャーアップデート",
        "Breaking changesの確認が必要。以下を確認すること:",
        "- [ ] CHANGELOG / Migration Guide の確認",
        "- [ ] 型エラーの有無",
        "- [ ] E2Eテストの通過",
        "- [ ] パフォーマンスへの影響確認"
      ]
    }
  ]
}
```

---

## CI/CDとの統合

### GitHub Actionsでのテスト自動実行

自動更新PRに対してCIを実行する設定例を示す。

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [20, 22]
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm test
      - run: npm run build

  # 依存関係更新PRに特化した追加チェック
  dependency-review:
    if: github.event.pull_request.user.login == 'renovate[bot]' || 
        github.event.pull_request.user.login == 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Dependency Review
        uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high
          deny-licenses: GPL-3.0, AGPL-3.0
```

### automerge条件の設定（GitHub）

GitHub側のBranch Protection Rulesと組み合わせることで、安全なautomergeを実現する。

```text
Branch Protection Rules (main):
  ├─ Require a pull request before merging
  │   └─ Required number of approvals: 0 (botのautomergeを許可する場合)
  ├─ Require status checks to pass before merging
  │   ├─ test (20)
  │   ├─ test (22)
  │   └─ dependency-review
  ├─ Require branches to be up to date before merging
  └─ Allow auto-merge
```

Renovateの`platformAutomerge: true`を設定すると、上記のstatus checksがすべてパスした時点でGitHubのauto-merge機能が発動する。

### Dependabotのautomerge（GitHub Actions経由）

Dependabotにはautomerge機能が組み込まれていないため、GitHub Actionsで実現する。

```yaml
# .github/workflows/dependabot-automerge.yml
name: Dependabot Auto-merge
on: pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  dependabot-automerge:
    runs-on: ubuntu-latest
    if: github.actor == 'dependabot[bot]'
    steps:
      - name: Dependabot metadata
        id: metadata
        uses: dependabot/fetch-metadata@v2
        with:
          github-token: "${{ secrets.GITHUB_TOKEN }}"
      
      - name: Auto-merge patch and minor updates
        if: >-
          steps.metadata.outputs.update-type == 'version-update:semver-patch' ||
          steps.metadata.outputs.update-type == 'version-update:semver-minor'
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Conventional Commits対応

チームでConventional Commitsを採用している場合、自動更新のコミットメッセージもそれに準拠させる。

**Renovate:**

```json
{
  "extends": [
    "config:recommended",
    ":semanticCommits"
  ],
  "semanticCommitType": "chore",
  "semanticCommitScope": "deps"
}
```

生成されるコミットメッセージ例:
```text
chore(deps): update dependency express to v4.21.2
chore(deps): update eslint packages (major)
fix(deps): update dependency axios to v1.7.9 [SECURITY]
```

**Dependabot:**

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    commit-message:
      prefix: "chore"
      prefix-development: "chore"
      include: "scope"
```

生成されるコミットメッセージ例:
```text
chore(deps): bump express from 4.21.1 to 4.21.2
chore(deps-dev): bump eslint from 9.18.0 to 9.19.0
```

---

## セキュリティ対応の自動化

### 脆弱性PR自動生成

セキュリティ修正は通常の更新と異なり、可能な限り迅速に適用する必要がある。

**Renovateでの脆弱性対応設定:**

```json
{
  "extends": ["config:recommended"],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"],
    "automerge": true,
    "platformAutomerge": true,
    "schedule": ["at any time"],
    "prPriority": 10,
    "reviewers": ["team:security"]
  }
}
```

**Dependabotでの設定:**

Dependabotのセキュリティアップデートはリポジトリ設定で有効化する。`dependabot.yml`内では直接制御できないが、セキュリティPRはバージョン更新PRとは独立して生成される。

### SLA設定の実装

脆弱性の深刻度に応じたSLA（対応期限）を設定する例を示す。GitHub Actionsで監視する。

```yaml
# .github/workflows/security-sla.yml
name: Security SLA Check
on:
  schedule:
    - cron: '0 9 * * *'  # 毎日 09:00 JST
  workflow_dispatch:

jobs:
  check-security-prs:
    runs-on: ubuntu-latest
    steps:
      - name: Check overdue security PRs
        uses: actions/github-script@v7
        with:
          script: |
            const SLA_HOURS = {
              critical: 24,
              high: 168,      // 1 week
              moderate: 720,  // 30 days
              low: 2160       // 90 days
            };
            
            const pulls = await github.rest.pulls.list({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'open',
              labels: 'security'
            });
            
            const now = new Date();
            const overdue = [];
            
            for (const pr of pulls.data) {
              const created = new Date(pr.created_at);
              const ageHours = (now - created) / (1000 * 60 * 60);
              
              // ラベルから深刻度を取得
              const severityLabel = pr.labels.find(l => 
                ['critical', 'high', 'moderate', 'low'].includes(l.name)
              );
              
              const severity = severityLabel ? severityLabel.name : 'high';
              const slaHours = SLA_HOURS[severity];
              
              if (ageHours > slaHours) {
                overdue.push({
                  number: pr.number,
                  title: pr.title,
                  severity,
                  ageHours: Math.round(ageHours),
                  slaHours
                });
              }
            }
            
            if (overdue.length > 0) {
              const message = overdue.map(pr => 
                `- #${pr.number}: ${pr.title} (${pr.severity}, ${pr.ageHours}h / SLA ${pr.slaHours}h)`
              ).join('\n');
              
              core.setFailed(`SLA超過のセキュリティPR:\n${message}`);
            }
```

### セキュリティ対応フロー

```text
脆弱性検出
  │
  ├─ critical / high
  │   ├─ Renovate: schedule: "at any time" + automerge
  │   ├─ Dependabot: security update PR自動生成
  │   └─ Slack通知 → 即時対応
  │
  ├─ moderate
  │   ├─ 通常のスケジュールで更新
  │   └─ 30日以内に対応
  │
  └─ low
      ├─ 通常のスケジュールで更新
      └─ 90日以内に対応
```

Slack通知の設定例（GitHub Actions）:

```yaml
      - name: Notify Slack on critical security PR
        if: contains(github.event.pull_request.labels.*.name, 'security')
        uses: slackapi/slack-github-action@v2
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
          webhook-type: incoming-webhook
          payload: |
            {
              "text": ":rotating_light: セキュリティ更新PR: ${{ github.event.pull_request.title }}\n${{ github.event.pull_request.html_url }}"
            }
```

---

## pnpm / yarn / Bun環境での設定

### pnpm

pnpmはRenovate・Dependabotの両方でサポートされているが、いくつか注意点がある。

**Renovateでのpnpm設定:**

```json
{
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "description": "pnpmのengine制約を尊重",
      "matchManagers": ["npm"],
      "rangeStrategy": "auto"
    }
  ]
}
```

Renovateはリポジトリ内の`pnpm-lock.yaml`を自動検出し、pnpmを使用してロックファイルを更新する。`packageManager`フィールド（`package.json`内）がある場合は、そのバージョンのpnpmが使われる。

```json
// package.json
{
  "packageManager": "pnpm@9.15.4"
}
```

**Dependabotでのpnpm:**

Dependabotもpnpmをサポートしている。`package-ecosystem: "npm"`を指定すれば、`pnpm-lock.yaml`を自動検出する。

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
```

**pnpm workspace対応（Renovate）:**

```json
{
  "extends": ["config:recommended"],
  "ignorePaths": ["**/node_modules/**"],
  "packageRules": [
    {
      "description": "workspace内部パッケージの更新を無視",
      "matchSourceUrls": [],
      "matchCurrentVersion": "workspace:*",
      "enabled": false
    }
  ]
}
```

pnpmのworkspace:プロトコルはRenovateが自動認識する。内部パッケージ間の参照（`"workspace:^"`等）は更新対象から除外される。

### Yarn (v3+ / Berry)

Yarn Berry（v3以降）は`.yarnrc.yml`で設定されるPlug'n'Play（PnP）モードに注意が必要である。

**Renovateでの設定:**

```json
{
  "extends": ["config:recommended"],
  "constraints": {
    "yarn": "4.x"
  }
}
```

Renovateは`.yarnrc.yml`の`yarnPath`を参照し、適切なYarnバージョンでロックファイルを更新する。

**注意点:**
- Yarn PnPモードの場合、`.pnp.cjs`や`.yarn/cache/`内のzipファイルもコミット対象になることがある
- `nodeLinker: node-modules`の場合は通常のnpmと同様の挙動になる
- Yarn Berryの`yarn.lock`はv1と形式が異なるため、古いツールでは解析に失敗することがある

**Dependabotでの注意:**

DependabotはYarn Berry（v2以降）のサポートに制限がある。特にPnPモードとの組み合わせで問題が発生する場合がある。Yarn Berryを使用している場合はRenovateを推奨する。

### Bun

BunはRenovateでの公式サポートが進んでいる。

**Renovateの対応状況:**

```json
{
  "extends": ["config:recommended"],
  "bun": {
    "enabled": true
  }
}
```

Renovateは`bun.lock`（Bun 1.2以降のデフォルトであるテキスト形式のロックファイル）および旧形式の`bun.lockb`（バイナリ形式）を検出し、Bunを使ってロックファイルを再生成する。ただし、Bunのバージョンによっては互換性の問題が発生する場合がある。

**Dependabotの対応状況:**

Dependabotは2026年3月時点でBunのサポートが限定的である。Bunプロジェクトの場合はRenovateの使用を推奨する。

### パッケージマネージャ別対応状況まとめ

| パッケージマネージャ | Renovate | Dependabot | 備考 |
|----------------------|----------|-----------|------|
| npm | 完全対応 | 完全対応 | 最も安定 |
| pnpm | 完全対応 | 対応 | workspace:プロトコルに注意 |
| Yarn Classic (v1) | 完全対応 | 完全対応 | |
| Yarn Berry (v3/v4) | 完全対応 | 制限あり | PnPモードはRenovate推奨 |
| Bun | 対応 | 限定的 | Renovate推奨 |

---

## トラブルシューティング

### 問題1: PRが生成されない

**原因と対処法:**

```text
チェックリスト:
  □ 設定ファイルのパスは正しいか
    - Dependabot: .github/dependabot.yml
    - Renovate: renovate.json / .renovaterc / .renovaterc.json
  □ YAMLの構文エラーはないか（インデントミス等）
  □ schedule設定の時間帯を過ぎているか
  □ open-pull-requests-limit / prConcurrentLimit に達していないか
  □ ignore / ignorePaths で除外していないか
  □ GitHub Appの権限は十分か（Renovate App）
```

**Renovateのデバッグ方法:**

Dependency Dashboard（自動生成されるIssue）を確認する。ダッシュボードには以下の情報が表示される。

- 検出された更新の一覧
- 各更新が処理されない理由
- 手動でPRを再作成するチェックボックス

ローカルでのデバッグ実行:

```bash
# Renovateをローカルで実行（dry-run）
npx renovate --platform=local --dry-run=full
```

**Dependabotのデバッグ:**

GitHub UIの Insights > Dependency graph > Dependabot タブで、直近の実行ログとエラーを確認できる。

### 問題2: ロックファイルの更新に失敗する

**よくある原因:**

```text
1. Node.jsバージョンの不一致
   → .nvmrc / .node-version / package.json の engines を確認
   → Renovate: constraints.node で明示指定

2. private registryへのアクセス失敗
   → Renovate: hostRules でトークンを設定
   → Dependabot: registries 設定を追加

3. postinstall scriptのエラー
   → Renovate: ignoreScripts: true を試す
```

**Renovateでprivate registryを設定する例:**

```json
{
  "extends": ["config:recommended"],
  "hostRules": [
    {
      "matchHost": "npm.pkg.github.com",
      "hostType": "npm",
      "token": "{{ secrets.GITHUB_TOKEN }}"
    },
    {
      "matchHost": "registry.npmjs.org",
      "hostType": "npm",
      "token": "{{ secrets.NPM_TOKEN }}"
    }
  ],
  "npmrc": "@myorg:registry=https://npm.pkg.github.com"
}
```

**Dependabotでprivate registryを設定する例:**

```yaml
version: 2
registries:
  github-npm:
    type: npm-registry
    url: https://npm.pkg.github.com
    token: ${{ secrets.GITHUB_TOKEN }}
    replaces-base: false
  
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    registries:
      - github-npm
```

### 問題3: automergeが動作しない

```text
チェックリスト:
  □ Branch Protection Rules で "Allow auto-merge" が有効か
  □ Required status checks がすべてパスしているか
  □ Renovate Appに十分な権限があるか
  □ "Require a pull request before merging" の設定は適切か
  □ platformAutomerge: true を設定しているか（Renovate）
  □ Dependabotの場合、automerge用のGitHub Actionsワークフローがあるか
```

### 問題4: 不要なPRが大量に生成される

**対処法（Renovate）:**

```json
{
  "prHourlyLimit": 2,
  "prConcurrentLimit": 10,
  "schedule": ["before 9am on monday"],
  "packageRules": [
    {
      "description": "テスト用パッケージの更新頻度を下げる",
      "matchPackageNames": ["@types/**"],
      "schedule": ["before 9am on the first day of the month"],
      "groupName": "@types packages"
    },
    {
      "description": "安定しているパッケージの更新を抑制",
      "matchPackageNames": ["lodash", "underscore"],
      "enabled": false
    }
  ]
}
```

**対処法（Dependabot）:**

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
    ignore:
      - dependency-name: "@types/*"
        update-types: ["version-update:semver-patch"]
    groups:
      all-minor-and-patch:
        patterns: ["*"]
        update-types: ["minor", "patch"]
```

### 問題5: マージ後にCIが壊れる

依存関係の更新PR単体ではCIが通っていたのに、マージ後に壊れるケースがある。

**原因:** ベースブランチとの差分が考慮されていない

**対処法:**

```text
1. Branch Protection で "Require branches to be up to date" を有効にする
   → ただしPR数が多い場合、マージキューが詰まる

2. Renovate の rebaseWhen 設定を調整する
   - "auto": Renovateが最適なタイミングでリベース（デフォルト）
   - "behind-base-branch": ベースブランチより古い場合にリベース
   - "conflicted": コンフリクト時のみリベース

3. GitHub Merge Queue を活用する（推奨）
   → 複数PRのマージ順序を制御し、統合テストを実行
```

```json
{
  "rebaseWhen": "behind-base-branch"
}
```

---

## まとめ — プロジェクト規模別の推奨構成

### 個人プロジェクト / 小規模（1-3名）

```text
推奨: Dependabot
理由: セットアップが最小限。GitHub標準で追加ツール不要。
```

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      all-minor-and-patch:
        patterns: ["*"]
        update-types: ["minor", "patch"]
    open-pull-requests-limit: 10
```

加えて、automerge用のGitHub Actionsワークフローを追加すれば、ほとんどの更新が自動処理される。

### 中規模チーム（4-15名）

```text
推奨: Renovate
理由: グルーピング・automerge・Dependency Dashboardが有用。
```

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
    ":semanticCommits",
    "schedule:weekends"
  ],
  "timezone": "Asia/Tokyo",
  "labels": ["dependencies"],
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "branch"
    },
    {
      "matchUpdateTypes": ["minor"],
      "matchDepTypes": ["devDependencies"],
      "automerge": true
    },
    {
      "matchUpdateTypes": ["major"],
      "labels": ["dependencies", "breaking-change"],
      "reviewers": ["team:leads"]
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "automerge": true,
    "schedule": ["at any time"],
    "labels": ["security"]
  }
}
```

### 大規模組織 / monorepo

```text
推奨: Renovate（セルフホスト）
理由: 実行頻度・API制限の制御、private registryとの統合、組織横断プリセット。
```

共有プリセットリポジトリを作成し、各リポジトリから参照する構成が有効である。

**共有プリセットリポジトリ（`myorg/renovate-config`）:**

```json
// default.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "timezone": "Asia/Tokyo",
  "schedule": ["before 9am on monday"],
  "labels": ["dependencies"],
  "semanticCommits": "enabled",
  "semanticCommitType": "chore",
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "branch"
    },
    {
      "matchUpdateTypes": ["minor"],
      "matchDepTypes": ["devDependencies"],
      "automerge": true
    },
    {
      "matchUpdateTypes": ["major"],
      "labels": ["dependencies", "major"],
      "automerge": false
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "automerge": true,
    "schedule": ["at any time"],
    "labels": ["security"],
    "prPriority": 10
  },
  "hostRules": [
    {
      "matchHost": "npm.pkg.github.com",
      "hostType": "npm",
      "token": "{{ secrets.GITHUB_TOKEN }}"
    }
  ]
}
```

**各リポジトリの設定（`renovate.json`）:**

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>myorg/renovate-config"
  ]
}
```

これにより、組織全体の依存関係管理ポリシーを1箇所で管理し、各リポジトリはオーバーライドが必要な部分のみ個別設定を追加する形になる。

### 構成選定フローチャート

```text
プロジェクトの特性を確認
  │
  ├─ GitHubのみ使用？
  │   ├─ Yes → monorepo？
  │   │         ├─ No  → 小規模なら Dependabot、中規模以上なら Renovate
  │   │         └─ Yes → Renovate
  │   └─ No  → Renovate（マルチプラットフォーム対応）
  │
  ├─ automergeを積極利用したい？
  │   ├─ Yes → Renovate
  │   └─ No  → どちらでも可
  │
  ├─ 組織横断で設定を統一したい？
  │   ├─ Yes → Renovate（共有プリセット）
  │   └─ No  → どちらでも可
  │
  └─ pnpm / Yarn Berry / Bun を使用？
      ├─ Yes → Renovate（より安定したサポート）
      └─ No  → どちらでも可
```

依存関係の自動更新は、設定して終わりではなく、チームの運用に合わせて継続的に調整していくものである。まずは最小限の設定で導入し、PR量やノイズを見ながらグルーピングやautomergeを段階的に有効化していくアプローチを推奨する。

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説している。
:::