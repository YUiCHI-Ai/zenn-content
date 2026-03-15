---
title: "環境構築と基本設定 ── 最速で使い始める3つの設定"
free: true
---

## テクニック1: CLAUDE.mdでプロジェクトの文脈を注入する

Claude Codeはプロジェクトルートの`CLAUDE.md`を自動的に読み込みます。ここにプロジェクト固有のルールを書いておくと、毎回の説明が不要になります。

```markdown
# CLAUDE.md の例

## 技術スタック
- Next.js 14 (App Router)
- TypeScript (strict mode)
- Prisma + PostgreSQL
- Tailwind CSS

## コーディング規約
- 関数コンポーネントのみ使用（classコンポーネント禁止）
- テストは Vitest で書く（Jest不使用）
- APIレスポンスは zod でバリデーション
- エラーは Result型パターンで処理（throw禁止）

## ディレクトリ構造
- src/app/ : ページコンポーネント
- src/components/ : 共通UIコンポーネント
- src/lib/ : ビジネスロジック
- src/db/ : Prismaスキーマとマイグレーション
```

**効果**: この設定だけで「Jestでテスト書いて」と言ってもVitestで書いてくれるようになります。プロジェクト固有の制約を毎回伝える手間がゼロになります。

## テクニック2: 許可モードの使い分け

Claude Codeには3つの許可モードがあります。用途に応じて使い分けることで、安全性と効率性を両立できます。

| モード | 動作 | 適する場面 |
|-------|------|-----------|
| `default` | ファイル編集・bash実行に毎回確認を求める | 初回利用時、本番環境の作業 |
| `autoEdit` | ファイル編集は自動許可、bash実行は確認 | 通常の開発作業（推奨） |
| `dontAsk` | 全操作を自動許可 | 信頼できるタスク、CI/CD連携 |

```bash
# 推奨: autoEditモードで起動
claude --allowedTools "Edit,Write,Read,Glob,Grep" "テスト追加して"

# 完全自動化（CI/CDなど）
claude -p --dontAsk "lint修正して全テスト通して"
```

**実践的な判断基準**: 開発中は`autoEdit`、CIパイプラインでは`dontAsk`の`-p`モードが最も効率的です。`default`は最初の1週間だけ使い、Claude Codeの動作を把握したら切り替えましょう。

## テクニック3: .claude/settings.jsonでチーム共有設定を作る

プロジェクトルートに`.claude/settings.json`を置くと、チームメンバー全員が同じ設定でClaude Codeを使えます。

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Edit",
      "Write",
      "Bash(npm run test*)",
      "Bash(npm run lint*)",
      "Bash(git status)",
      "Bash(git diff*)",
      "Bash(git log*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force*)",
      "Bash(git reset --hard*)"
    ]
  }
}
```

**ポイント**: `allow`にはよく使うコマンドをパターンで許可し、`deny`には破壊的操作を明示的にブロックします。これにより`autoEdit`モードでも安全に運用できます。

:::message
`deny`に指定したコマンドは、`dontAsk`モードでも実行がブロックされます。本番環境で事故を防ぐ安全弁として必ず設定しましょう。
:::
