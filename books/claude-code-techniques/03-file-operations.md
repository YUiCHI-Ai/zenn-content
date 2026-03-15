---
title: "ファイル操作の効率化 ── 一括処理で手作業を消す"
---

## テクニック4: 複数ファイルの一括リネーム・リファクタ

ファイル名やexportの命名規則を一括で変更できます。Claude Codeはimport文の参照先まで自動修正します。

```bash
claude "src/components/ 以下の全ファイルを kebab-case から PascalCase にリネームして。
import文も全て修正して。"
```

**手動との比較**:
- 手動: ファイル名変更 → 全import文をgrep → 1つずつ修正（30分〜）
- Claude Code: 1コマンド（30秒〜2分）

## テクニック5: プロジェクト横断のコード検索＆置換

正規表現では対応しきれない「意味を理解した置換」ができます。

```bash
claude "プロジェクト全体で console.log を使っている箇所を探して、
開発用のデバッグログは logger.debug() に、
エラーハンドリング内のものは logger.error() に置き換えて。
テストファイルの console.log はそのまま残して。"
```

**ポイント**: 単純な文字列置換ではなく、**文脈を判断して置換先を変える**ことがClaude Codeの強みです。正規表現の`sed`では実現できない操作です。

## テクニック6: 設定ファイルの一括生成

新しいプロジェクトのボイラープレート生成を1コマンドで完了します。

```bash
claude "以下の条件でNext.js 14プロジェクトの設定ファイルを生成して:
- TypeScript strict
- ESLint flat config
- Prettier連携
- Vitest設定
- GitHub Actions CI（lint + test + build）
- .env.example

package.jsonのscriptsも適切に設定して。"
```

生成されるファイル:
```
tsconfig.json
eslint.config.mjs
.prettierrc
vitest.config.ts
.github/workflows/ci.yml
.env.example
package.json (scripts追加)
```

**実践的な使い方**: プロジェクトテンプレートをCLAUDE.mdに定義しておくと、新規プロジェクト作成時に毎回同じ品質の設定が生成されます。

## テクニック7: ドキュメントの自動生成

コードからAPIドキュメントやREADMEを自動生成します。

```bash
# APIエンドポイントからOpenAPI仕様書を生成
claude "src/app/api/ 以下の全ルートを読んで、
OpenAPI 3.0形式のAPI仕様書をdocs/api.yamlに生成して。
各エンドポイントのリクエスト/レスポンスの型はコードから推論して。"

# README.mdの自動更新
claude "package.jsonとsrc/の構造を読んで、README.mdの
セットアップ手順・利用可能なnpmスクリプト・ディレクトリ構造の
セクションを最新の状態に更新して。"
```

**運用のコツ**: `git commit`前のフックにClaude Codeを入れて、変更があったファイルに関連するドキュメントを自動更新するワークフローが構築できます（9章で詳述）。
