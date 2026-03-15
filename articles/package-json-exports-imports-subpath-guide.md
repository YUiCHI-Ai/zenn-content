---
title: "package.json exports完全ガイド：subpath exports / imports / conditional exportsを実務で使いこなす"
emoji: "📤"
type: "tech"
topics: ["nodejs", "npm", "packagejson", "esm", "typescript"]
published: true
---

## はじめに

Node.jsのパッケージを公開・利用する上で、`package.json`の`exports`フィールドは今や避けて通れない設定項目である。

従来の`main`フィールドだけでは、以下のような問題に対処できなかった。

- **ESM/CJS混在問題**: `import`と`require`で異なるエントリーポイントを返せない
- **内部ファイルへの意図しないアクセス**: `require('my-lib/src/internal/helper')`のような、公開APIではないファイルへのアクセスを防げない
- **サブパスの明示的な定義**: パッケージが複数のエントリーポイントを持つ場合に、どのパスが有効かを宣言的に示す手段がない

`exports`フィールドはNode.js 12.7.0で導入され、v12.22.0以降で安定版（フラグなし）となった。2026年現在、主要なバンドラー・ツールが対応しており、新規パッケージでは`exports`の設定が事実上の標準となっている。

本記事では、`exports`・`imports`・subpath patternsの**具体的な設定方法**にフォーカスし、実務でそのまま使えるパターンを網羅する。モジュールシステムがなぜ複雑になったのか、という設計思想・歴史的経緯についてはここでは扱わない。

## exportsの基本

### mainフィールドとの違い

従来の`main`フィールドは、パッケージのエントリーポイントを1つだけ指定するものだった。

```json
{
  "name": "my-lib",
  "main": "./dist/index.js"
}
```

この場合、`require('my-lib')`は`./dist/index.js`を返す。しかし、これだけでは以下が実現できない。

| 機能 | `main` | `exports` |
|:---|:---:|:---:|
| 単一エントリーポイント | o | o |
| 複数エントリーポイント | x | o |
| CJS/ESM条件分岐 | x | o |
| 内部ファイルのアクセス制限 | x | o |
| サブパスのワイルドカード | x | o |
| TypeScript types条件 | x | o |

### 最小限のexports設定

`exports`の最もシンプルな形は、メインエントリーポイントを`"."`で定義する形式である。

```json
{
  "name": "my-lib",
  "exports": "./dist/index.js"
}
```

これは以下の省略形である。

```json
{
  "name": "my-lib",
  "exports": {
    ".": "./dist/index.js"
  }
}
```

`"."`はパッケージ名そのもの（`import something from 'my-lib'`）に対応する。

### exportsの「封鎖」効果

`exports`を定義した瞬間、**明示的に定義されたパス以外はすべてアクセス不可になる**。これが`main`との最大の違いである。

```json
{
  "name": "my-lib",
  "exports": {
    ".": "./dist/index.js"
  }
}
```

```typescript
// OK: exportsで定義されたパス
import { something } from 'my-lib';

// ERR_PACKAGE_PATH_NOT_EXPORTED: 定義されていないパス
import { helper } from 'my-lib/dist/utils/helper';
```

これにより、パッケージの公開APIを厳密に制御できる。内部の実装詳細を利用者に触らせたくない場合に非常に有効である。

### mainとexportsの共存

後方互換性のために、`main`と`exports`を共存させることが可能である。`exports`をサポートする環境では`exports`が優先され、古い環境では`main`にフォールバックする。

```json
{
  "name": "my-lib",
  "main": "./dist/index.cjs",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

## Subpath exports

### 基本的なサブパスの定義

パッケージが複数のエントリーポイントを提供する場合、サブパスを定義する。

```json
{
  "name": "my-lib",
  "exports": {
    ".": "./dist/index.js",
    "./utils": "./dist/utils/index.js",
    "./constants": "./dist/constants.js",
    "./types": "./dist/types.js"
  }
}
```

利用側のコードは以下のようになる。

```typescript
import { something } from 'my-lib';
import { formatDate } from 'my-lib/utils';
import { MAX_RETRY } from 'my-lib/constants';
import type { Config } from 'my-lib/types';
```

### ワイルドカード（*）パターン

多数のサブパスを1つずつ列挙する代わりに、`*`（アスタリスク）を使ったパターンマッチが利用できる。

```json
{
  "name": "my-lib",
  "exports": {
    ".": "./dist/index.js",
    "./components/*": "./dist/components/*/index.js",
    "./utils/*": "./dist/utils/*.js"
  }
}
```

この場合の対応関係は以下の通りである。

| importパス | 解決先ファイル |
|:---|:---|
| `my-lib/components/Button` | `./dist/components/Button/index.js` |
| `my-lib/components/Modal` | `./dist/components/Modal/index.js` |
| `my-lib/utils/format` | `./dist/utils/format.js` |
| `my-lib/utils/validate` | `./dist/utils/validate.js` |

`*`は1段階のみマッチする（ディレクトリ区切りを含まない）。つまり、`my-lib/components/form/Input`は上記のパターンにはマッチしない点に注意が必要である。

### 拡張子を含むサブパス

ファイル拡張子を明示的にサブパスに含めることも可能である。

```json
{
  "name": "my-lib",
  "exports": {
    ".": "./dist/index.js",
    "./styles/*.css": "./dist/styles/*.css",
    "./locales/*.json": "./dist/locales/*.json"
  }
}
```

```typescript
import 'my-lib/styles/reset.css';

// JSON importの例
import ja from 'my-lib/locales/ja.json' with { type: 'json' };
```

### ネストしたサブパスの定義

ディレクトリ階層が深い場合は、明示的にパスを列挙する方法が安全である。

```json
{
  "name": "my-ui",
  "exports": {
    ".": "./dist/index.js",
    "./components/Button": "./dist/components/Button/index.js",
    "./components/Form/Input": "./dist/components/Form/Input/index.js",
    "./components/Form/Select": "./dist/components/Form/Select/index.js",
    "./hooks/useForm": "./dist/hooks/useForm.js"
  }
}
```

多数になる場合はワイルドカードと組み合わせるとよい。

```json
{
  "name": "my-ui",
  "exports": {
    ".": "./dist/index.js",
    "./components/*": "./dist/components/*/index.js",
    "./components/Form/*": "./dist/components/Form/*/index.js",
    "./hooks/*": "./dist/hooks/*.js"
  }
}
```

## Conditional exports

### 基本構文

Conditional exportsは、実行環境の条件に応じて異なるファイルを返す仕組みである。

```json
{
  "name": "my-lib",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "default": "./dist/index.mjs"
    }
  }
}
```

### 主要な条件キー

Node.jsとエコシステムで認識される主要な条件キーを以下にまとめる。

| 条件キー | 評価される場面 | 典型的な用途 |
|:---|:---|:---|
| `import` | ESM（`import`文）で読み込まれた場合 | `.mjs`や`type: "module"`のファイル |
| `require` | CJS（`require()`）で読み込まれた場合 | `.cjs`ファイル |
| `types` | TypeScriptの型解決 | `.d.ts`/`.d.mts`/`.d.cts`ファイル |
| `default` | 他のどの条件にもマッチしなかった場合 | フォールバック |
| `node` | Node.js環境 | Node固有の実装 |
| `browser` | ブラウザ環境（バンドラーが解釈） | DOM依存の実装 |
| `development` | 開発環境 | デバッグ用コード |
| `production` | 本番環境 | 最適化済みコード |

### 条件の評価順序

条件は**上から順に**評価され、最初にマッチした条件のパスが使われる。これは極めて重要なルールである。

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "default": "./dist/index.mjs"
    }
  }
}
```

上記の場合、TypeScriptコンパイラは`types`を最初に見つけて`.d.ts`を使用する。Node.jsのESMローダーは`types`を無視し、`import`にマッチして`.mjs`を返す。

**`default`は必ず最後に置く。** `default`を先頭に置くと、すべての環境でそのファイルが使われてしまう。

### 条件のネスト

条件はネストすることが可能である。「ESMかつNode.js環境」のような複合条件を表現できる。

```json
{
  "exports": {
    ".": {
      "node": {
        "import": "./dist/node-esm.mjs",
        "require": "./dist/node-cjs.cjs"
      },
      "browser": {
        "import": "./dist/browser-esm.mjs",
        "default": "./dist/browser.mjs"
      },
      "default": "./dist/index.mjs"
    }
  }
}
```

### Dual Package向けの推奨構成

CJSとESMの両方に対応するパッケージの`exports`設定は以下の形式が推奨される。

```json
{
  "name": "my-lib",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  }
}
```

`"type": "module"`を設定している場合、`.js`はESMとして扱われ、`.cjs`がCJSとなる。逆に`"type": "module"`がない場合は`.js`がCJSで`.mjs`がESMとなる。

| `type`フィールド | `.js`の扱い | ESM拡張子 | CJS拡張子 |
|:---|:---|:---|:---|
| `"module"` | ESM | `.js` | `.cjs` |
| なし（デフォルト） | CJS | `.mjs` | `.js` |

## TypeScriptとの統合

### typesCondition

TypeScript 4.7以降、`exports`フィールドの`types`条件を解決できるようになった。ただし、`tsconfig.json`の`moduleResolution`設定が適切である必要がある。

```json
// tsconfig.json
{
  "compilerOptions": {
    "module": "Node16",
    "moduleResolution": "Node16"
  }
}
```

もしくは、より新しい設定として以下を使う。

```json
// tsconfig.json
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext"
  }
}
```

`"moduleResolution": "Bundler"`（TypeScript 5.0以降）もexportsの`types`条件を解決する。

```json
// tsconfig.json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler"
  }
}
```

| moduleResolution | exports対応 | 主な用途 |
|:---|:---:|:---|
| `node` (Node10) | x | レガシー（非推奨） |
| `Node16` / `NodeNext` | o | Node.js向けパッケージ |
| `Bundler` | o | バンドラー利用のアプリ |

**`"moduleResolution": "node"`（Node10）では`exports`のtypes条件が解決されない。** これは非常に多くの問題の原因となっている。

### CJS/ESM別の型定義ファイル

Dual Packageで型定義もCJS/ESM別に提供する場合、拡張子を使い分ける。

```json
{
  "exports": {
    ".": {
      "import": {
        "types": "./dist/index.d.mts",
        "default": "./dist/index.mjs"
      },
      "require": {
        "types": "./dist/index.d.cts",
        "default": "./dist/index.cjs"
      }
    }
  }
}
```

型定義ファイルの拡張子の対応関係は以下の通りである。

| ソース拡張子 | 型定義拡張子 |
|:---|:---|
| `.js` | `.d.ts` |
| `.mjs` | `.d.mts` |
| `.cjs` | `.d.cts` |

### tsconfig paths との関係

monorepoなどで`tsconfig.json`の`paths`を使っている場合、`exports`と`paths`は異なるレイヤーで機能する。

- **`paths`**: TypeScriptコンパイラのみが参照する。ビルド時のモジュール解決を制御する
- **`exports`**: Node.jsランタイムおよびバンドラーが参照する。実行時のモジュール解決を制御する

```json
// tsconfig.json（開発時のエイリアス）
{
  "compilerOptions": {
    "paths": {
      "@my-lib/*": ["./packages/my-lib/src/*"]
    }
  }
}
```

```json
// packages/my-lib/package.json（公開時のエクスポート）
{
  "exports": {
    ".": "./dist/index.js",
    "./*": "./dist/*.js"
  }
}
```

### typesVersions（レガシー環境向け）

`"moduleResolution": "node"`（Node10）を使う利用者がいる場合、`typesVersions`によるフォールバックを設定できる。

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./utils": {
      "types": "./dist/utils.d.ts",
      "default": "./dist/utils.js"
    }
  },
  "typesVersions": {
    "*": {
      "utils": ["./dist/utils.d.ts"]
    }
  }
}
```

`typesVersions`は`exports`の`types`条件が解決されない環境でのみフォールバックとして動作する。新規プロジェクトでは不要になりつつあるが、広く利用されるライブラリでは設定しておくと安全である。

## importsフィールド

### 基本構文

`imports`フィールドは、パッケージ**内部**で使用するモジュールエイリアスを定義する機能である。`exports`が外部向けの公開APIを制御するのに対し、`imports`は内部のモジュール解決を制御する。

```json
{
  "name": "my-app",
  "imports": {
    "#utils": "./src/utils/index.js",
    "#config": "./src/config.js",
    "#db": "./src/database/client.js"
  }
}
```

`imports`のキーは必ず`#`で始まる必要がある。これにより、npmパッケージ名との衝突を避ける。

```typescript
// src/handlers/user.js
import { formatDate } from '#utils';
import { DATABASE_URL } from '#config';
import { query } from '#db';
```

### ワイルドカードパターン

`exports`と同様に`*`パターンが使える。

```json
{
  "imports": {
    "#components/*": "./src/components/*/index.js",
    "#utils/*": "./src/utils/*.js",
    "#lib/*": "./src/lib/*.js"
  }
}
```

```typescript
import { Button } from '#components/Button';
import { formatCurrency } from '#utils/format';
```

### Conditional imports

`imports`にもconditional exportsと同じ条件分岐の仕組みが使える。これがテスト時のモック差し替えに特に有用である。

```json
{
  "imports": {
    "#db": {
      "development": "./src/database/mock-client.js",
      "default": "./src/database/client.js"
    },
    "#logger": {
      "development": "./src/logger/console.js",
      "production": "./src/logger/structured.js",
      "default": "./src/logger/console.js"
    }
  }
}
```

### テスト用モック差し替え

`imports`の最も実用的なユースケースの1つが、テスト時のモジュール差し替えである。

```json
{
  "imports": {
    "#payment-gateway": {
      "production": "./src/payment/stripe.js",
      "default": "./src/payment/stripe.js"
    }
  }
}
```

テスト時に`--conditions`フラグでカスタム条件を指定することで、モジュールを差し替えられる。

```bash
# 通常実行
node src/server.js

# テスト用条件を指定
node --conditions=test src/test-runner.js
```

```json
{
  "imports": {
    "#payment-gateway": {
      "test": "./src/payment/mock-gateway.js",
      "default": "./src/payment/stripe.js"
    }
  }
}
```

### 外部パッケージへのマッピング

`imports`のマッピング先はローカルファイルだけでなく、外部パッケージも指定できる。

```json
{
  "imports": {
    "#http": {
      "node": "node:http",
      "default": "./src/polyfills/http.js"
    },
    "#crypto": {
      "node": "node:crypto",
      "default": "./src/polyfills/crypto.js"
    }
  }
}
```

これにより、Node.js固有のビルトインモジュールをブラウザ環境ではポリフィルに切り替えるといった制御が可能になる。

## 実践パターン集

### パターン1: Dual Package（CJS + ESM）公開

最も一般的なパターンである。TypeScriptで開発し、CJSとESMの両方を公開する構成を示す。

```json
{
  "name": "my-utility-lib",
  "version": "2.1.0",
  "type": "module",
  "main": "./dist/cjs/index.cjs",
  "module": "./dist/esm/index.js",
  "types": "./dist/types/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/types/index.d.ts",
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.cjs",
      "default": "./dist/esm/index.js"
    },
    "./package.json": "./package.json"
  },
  "files": [
    "dist"
  ]
}
```

ディレクトリ構成:

```text
my-utility-lib/
├── dist/
│   ├── esm/
│   │   └── index.js
│   ├── cjs/
│   │   └── index.cjs
│   └── types/
│       └── index.d.ts
├── src/
│   └── index.ts
├── package.json
└── tsconfig.json
```

ビルド設定（tsup）の例:

```typescript
// tsup.config.ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['cjs', 'esm'],
  dts: true,
  outDir: 'dist',
  clean: true,
  splitting: false,
});
```

### パターン2: Reactコンポーネントライブラリ

Reactコンポーネントをサブパスで個別にインポート可能にするパターンである。tree-shakingを最大限に活かせる。

```json
{
  "name": "@myorg/ui",
  "version": "3.0.0",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    },
    "./Button": {
      "types": "./dist/components/Button/index.d.ts",
      "import": "./dist/components/Button/index.js",
      "require": "./dist/components/Button/index.cjs"
    },
    "./Modal": {
      "types": "./dist/components/Modal/index.d.ts",
      "import": "./dist/components/Modal/index.js",
      "require": "./dist/components/Modal/index.cjs"
    },
    "./hooks/*": {
      "types": "./dist/hooks/*.d.ts",
      "import": "./dist/hooks/*.js",
      "require": "./dist/hooks/*.cjs"
    },
    "./styles/*.css": "./dist/styles/*.css",
    "./package.json": "./package.json"
  },
  "peerDependencies": {
    "react": "^18.0.0 || ^19.0.0",
    "react-dom": "^18.0.0 || ^19.0.0"
  }
}
```

利用側:

```typescript
// バンドル全体（tree-shakingに依存）
import { Button, Modal } from '@myorg/ui';

// 個別インポート（確実に不要なコードを除外）
import { Button } from '@myorg/ui/Button';
import { useDialog } from '@myorg/ui/hooks/useDialog';
import '@myorg/ui/styles/reset.css';
```

### パターン3: CLIツール

CLIツールでは`bin`フィールドと`exports`を併用する。プログラマティックAPIとCLIの両方を提供するパターンが多い。

```json
{
  "name": "my-cli-tool",
  "version": "1.5.0",
  "type": "module",
  "bin": {
    "my-cli": "./dist/cli.js"
  },
  "exports": {
    ".": {
      "types": "./dist/api/index.d.ts",
      "import": "./dist/api/index.js",
      "require": "./dist/api/index.cjs"
    },
    "./config": {
      "types": "./dist/config/index.d.ts",
      "import": "./dist/config/index.js",
      "require": "./dist/config/index.cjs"
    }
  }
}
```

```typescript
// CLIとして実行
// $ npx my-cli-tool --input file.txt

// プログラマティックAPIとして利用
import { processFile } from 'my-cli-tool';
import { defineConfig } from 'my-cli-tool/config';
```

### パターン4: monorepoの内部パッケージ

monorepoの内部パッケージでは、ビルド前のTypeScriptソースを直接参照する構成が便利である。

```json
{
  "name": "@myorg/shared-utils",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "default": "./src/index.ts"
    },
    "./date": {
      "types": "./src/date/index.ts",
      "default": "./src/date/index.ts"
    },
    "./validation": {
      "types": "./src/validation/index.ts",
      "default": "./src/validation/index.ts"
    }
  }
}
```

この構成はバンドラー（turbopack、tsupなど）がビルド時にトランスパイルすることを前提としている。`"private": true`を設定して、npmに公開しないことを明示する。

monorepoのワークスペース設定（pnpm-workspace.yamlの例）:

```yaml
packages:
  - 'packages/*'
  - 'apps/*'
```

利用側の`package.json`:

```json
{
  "name": "@myorg/web-app",
  "dependencies": {
    "@myorg/shared-utils": "workspace:*"
  }
}
```

```typescript
// apps/web/src/handler.ts
import { formatDate } from '@myorg/shared-utils/date';
import { validateEmail } from '@myorg/shared-utils/validation';
```

### パターン5: Server/Client条件分岐（フルスタックライブラリ）

Next.jsなどのフルスタックフレームワーク向けに、サーバー/クライアントで異なるコードを提供するパターンである。

```json
{
  "name": "my-auth-lib",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "react-server": "./dist/server.js",
      "import": "./dist/client.js",
      "default": "./dist/client.js"
    },
    "./server": {
      "types": "./dist/server.d.ts",
      "import": "./dist/server.js"
    },
    "./client": {
      "types": "./dist/client.d.ts",
      "import": "./dist/client.js"
    }
  }
}
```

`react-server`条件はNext.js App Routerのサーバーコンポーネントで認識される。

## バンドラー・ツールとの互換性

### 対応状況一覧（2026年3月時点）

| ツール | exports対応 | imports対応 | conditions対応 | 備考 |
|:---|:---:|:---:|:---:|:---|
| Node.js (>=12.22) | o | o | o | ネイティブサポート |
| webpack (>=5) | o | o | o | `resolve.conditionNames`で追加条件指定可 |
| Vite (>=3) | o | o | o | `resolve.conditions`で追加条件指定可 |
| esbuild (>=0.15) | o | o | o | `--conditions`フラグ対応 |
| Rollup (>=3) | o | o | o | `@rollup/plugin-node-resolve`経由 |
| Jest (>=28) | o | 部分的 | 部分的 | `customExportConditions`設定が必要 |
| Vitest | o | o | o | Viteベースのため完全対応 |
| tsx | o | o | o | esbuildベース |
| ts-node | 部分的 | 部分的 | 部分的 | `--esm`フラグ + `experimentalSpecifierResolution`が必要な場合あり |

### webpack 5の設定

```javascript
// webpack.config.js
module.exports = {
  resolve: {
    // デフォルトでexportsを解決する
    // カスタム条件を追加する場合:
    conditionNames: ['import', 'require', 'node', 'default'],
  },
};
```

### Viteの設定

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  resolve: {
    // カスタム条件を追加する場合
    conditions: ['development'],
  },
});
```

### esbuildの設定

```bash
# CLIで条件を指定
esbuild src/index.ts --bundle --conditions=production

# development条件を使う場合
esbuild src/index.ts --bundle --conditions=development
```

```typescript
// esbuild APIの場合
import * as esbuild from 'esbuild';

await esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  conditions: ['production'],
  outdir: 'dist',
});
```

### Jest の設定

Jestは`exports`の解決にやや追加設定が必要な場合がある。

```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'node',
  // exportsのconditionsを制御
  testEnvironmentOptions: {
    customExportConditions: ['node', 'node-addons'],
  },
  // もしくはmoduleNameMapperで直接マッピング
  moduleNameMapper: {
    '^#(.*)$': '<rootDir>/src/$1',
  },
};
```

### Vitest の設定

VitestはViteベースであるため、exportsは追加設定なしで動作する。

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    // importsフィールドの#エイリアスも自動解決される
    // カスタム条件が必要な場合:
    server: {
      deps: {
        external: [/node_modules/],
      },
    },
  },
  resolve: {
    conditions: ['development'],
  },
});
```

## トラブルシューティング

### ERR_PACKAGE_PATH_NOT_EXPORTED

最も頻繁に遭遇するエラーである。

```text
Error [ERR_PACKAGE_PATH_NOT_EXPORTED]: Package subpath './internal/helper'
is not defined by "exports" in /path/to/node_modules/some-lib/package.json
```

**原因**: `exports`で定義されていないサブパスにアクセスしようとしている。

**解決策**:

1. パッケージが`exports`で公開しているパスを確認する
2. 正しいインポートパスに修正する
3. どうしても内部ファイルにアクセスする必要がある場合、パッケージのメンテナーにissueを立てる

```bash
# パッケージのexportsを確認する
node -e "console.log(JSON.stringify(require('./node_modules/some-lib/package.json').exports, null, 2))"
```

### ERR_PACKAGE_IMPORT_NOT_DEFINED

```text
Error [ERR_PACKAGE_IMPORT_NOT_DEFINED]: Package import specifier "#utils"
is not defined in package /path/to/project/package.json
```

**原因**: `imports`フィールドで`#utils`が定義されていない、またはパスが間違っている。

**解決策**:

```json
// package.jsonにimportsを追加/修正
{
  "imports": {
    "#utils": "./src/utils/index.js"
  }
}
```

### TypeScriptで型が解決されない

```text
Cannot find module 'my-lib/utils' or its corresponding type declarations.
```

**原因**: 多くの場合、`tsconfig.json`の`moduleResolution`が`node`（Node10）になっている。

**解決策**:

```json
// tsconfig.json
{
  "compilerOptions": {
    // "moduleResolution": "node" ← これが原因
    "moduleResolution": "NodeNext"  // ← これに変更
  }
}
```

もしくは、`moduleResolution`を変更できない場合は`typesVersions`をフォールバックとして設定する。

### Dual Package Hazard

CJSとESMの両方を公開する場合、同じパッケージがCJSとESMで二重にロードされる「Dual Package Hazard」が発生し得る。

```text
# シングルトンのはずが2つのインスタンスが存在する
import config from 'my-config'; // ESM版のインスタンス
const config2 = require('my-config'); // CJS版の別インスタンス
// config !== config2 → バグの原因
```

**解決策**: CJS側をESM側のラッパーにする。

```javascript
// dist/index.cjs
// CJS版はESM版を動的importしてre-exportする
module.exports = new Promise((resolve) => {
  import('./index.mjs').then((mod) => {
    resolve(mod);
  });
});
```

もしくは、ステートを持たないユーティリティ関数のみの場合はこの問題は発生しないため、気にしなくてよい。

:::message
📖 exportsの複雑さの根底にあるのは、CJSとESMという2つのモジュールシステムの**設計思想の衝突**である。この衝突がなぜ起きたのか、依存解決の原理から理解したい方は **[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)** で図解で解説している。
:::

### ワイルドカードパターンがマッチしない

```typescript
// 期待: my-lib/components/nested/Deep にマッチしてほしい
import { Deep } from 'my-lib/components/nested/Deep';
// ERR_PACKAGE_PATH_NOT_EXPORTED
```

**原因**: `*`は1段階のパスセグメントのみにマッチする。`/`を跨がない。

```json
{
  "exports": {
    "./components/*": "./dist/components/*/index.js"
  }
}
```

この定義では`my-lib/components/Button`はマッチするが、`my-lib/components/nested/Deep`はマッチしない。

**解決策**: ネストしたパスを明示的に定義する。

```json
{
  "exports": {
    "./components/*": "./dist/components/*/index.js",
    "./components/nested/*": "./dist/components/nested/*/index.js"
  }
}
```

### package.json自体へのアクセス

一部のツールがパッケージの`package.json`を直接読み込む場合がある。`exports`でこれを許可する必要がある。

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./package.json": "./package.json"
  }
}
```

これを忘れると、`require('my-lib/package.json')`が`ERR_PACKAGE_PATH_NOT_EXPORTED`になる。

### 条件の優先順位を間違える

```json
{
  "exports": {
    ".": {
      "default": "./dist/index.js",
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs"
    }
  }
}
```

上記は間違いである。`default`が先頭にあるため、すべての環境で`./dist/index.js`が返される。`types`や`import`は無視される。

**正しい順序**:

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "default": "./dist/index.js"
    }
  }
}
```

ルール: **具体的な条件を先に、`default`は最後に。**

## まとめ

### exports設定のベストプラクティスチェックリスト

設定を行う際に以下のチェックリストを確認してほしい。

**基本設定:**

- [ ] `exports`フィールドが定義されている
- [ ] `"."`エントリーポイントが存在する
- [ ] `main`フィールドをフォールバックとして併記している（広い互換性が必要な場合）
- [ ] `"./package.json": "./package.json"`を含めている

**Conditional exports:**

- [ ] `types`条件を最初に配置している
- [ ] `default`条件を最後に配置している
- [ ] CJS/ESM両対応の場合、`import`と`require`の両方を定義している
- [ ] 各条件のファイルパスが実際に存在するファイルを指している

**TypeScript:**

- [ ] `types`条件がすべてのエントリーポイントに設定されている
- [ ] `.d.ts`/`.d.mts`/`.d.cts`の拡張子が対応する`.js`/`.mjs`/`.cjs`と一致している
- [ ] 利用者向けに必要な`moduleResolution`を README に記載している
- [ ] レガシー環境向けに`typesVersions`を設定している（必要な場合のみ）

**Subpath exports:**

- [ ] ワイルドカード`*`が1段階のパスのみにマッチすることを理解した上で使用している
- [ ] 公開すべきサブパスがすべて列挙されている
- [ ] 内部実装ファイルが`exports`から除外されている

**テスト・検証:**

- [ ] `npm pack`でパッケージの内容物を確認している
- [ ] CJS（`require`）とESM（`import`）の両方で動作確認している
- [ ] 主要なバンドラー（webpack / Vite）で解決されることを確認している

**検証コマンド:**

```bash
# パッケージの内容物を確認
npm pack --dry-run

# CJSでの動作確認
node -e "const lib = require('my-lib'); console.log(lib);"

# ESMでの動作確認
node --input-type=module -e "import lib from 'my-lib'; console.log(lib);"

# exportsの解決先を確認
node -e "const {createRequire} = require('module'); const r = createRequire(__filename); console.log(r.resolve('my-lib'));"
```

`exports`フィールドは設定項目が多く、最初は複雑に感じるかもしれない。しかし、ここで解説したパターンを組み合わせれば、ほぼすべてのユースケースをカバーできる。まずは最小限の`"."`エントリーポイントから始め、必要に応じてサブパスや条件分岐を追加していくのが最も確実なアプローチである。

:::message
📖 パッケージマネージャの**仕組み**をさらに深く理解したい方へ
**[『なぜnode_modulesは壊れるのか？』](https://zenn.dev/yuichi_ai/books/package-manager-from-scratch)**では、依存解決アルゴリズムの原理から3つのパッケージマネージャの設計思想の違いを図解で解説している。
:::

:::message
この記事はAI（Claude）による生成を含む。技術的な正確性については可能な限り検証しているが、最新情報は公式ドキュメントを確認してほしい。
:::
