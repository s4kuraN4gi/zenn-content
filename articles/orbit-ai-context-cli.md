---
title: "Claude Code / Cursor のAIコンテキスト生成を自動化するCLIツールを作った"
emoji: "🔍"
type: "tech"
topics: ["claude", "cursor", "ai", "cli", "typescript"]
published: false
---

## はじめに

Claude Code や Cursor を使って開発していると、「なんでこのAI、プロジェクトの構造わかってないんだろう」と思う瞬間がある。Next.js のプロジェクトなのに Express 前提のコードを提案してきたり、既にある共通関数を無視して同じ処理を一から書き始めたり。AIは賢いけど、目の前のプロジェクトについては何も知らない状態でスタートする。

そこで重要になるのが `CLAUDE.md` や `.cursorrules` といったコンテキストファイルだ。プロジェクトの技術スタック、ディレクトリ構成、コーディング規約などを書いておくと、AIがそれを参照してくれる。Claude Code なら `CLAUDE.md`、Cursor なら `.cursorrules`、GitHub Copilot なら `copilot-instructions.md` がある。

ただ、これを手書きで維持するのがとにかく面倒くさい。最初に気合を入れて書いても、ルーティングが増えたりDBスキーマが変わったりするたびに更新が必要になる。しかも実際にAIに必要な情報って「どのファイルが何をexportしているか」「どのモジュールが中心的に使われているか」みたいな、手作業で整理するには地味すぎる情報だったりする。

そこで、コードベースを自動解析してコンテキストファイルを生成するCLIツール「Orbit」を作った。`orbit scan -g` の一発でプロジェクトの全体像を構造化して出力してくれる。

## Orbit とは

Orbit は、コードベースの構造を解析してAIアシスタント向けのコンテキストファイルを生成するCLIツールだ。TypeScript で書かれていて、npm からインストールできる。

ポイントは、コードの全量ダンプではなく「構造化されたコンテキスト」を生成するところ。ソースコードを全部テキストにまとめてAIに渡すのではなく、技術スタック・ページ構成・APIルート・DBスキーマ・import graph・exportカタログといった「プロジェクトの地図」をコンパクトに生成する。AIにとって本当に必要なのは、コードの一字一句ではなく、プロジェクトの構造を俯瞰できる情報だ。

MIT ライセンスの完全無料・オープンソースで、認証やサインアップも不要。インストールして即使える。

## 使い方

### インストールと実行

```bash
npm i -g @orbit-cli/core
```

プロジェクトのルートで以下を実行するだけ。

```bash
orbit scan -g
```

これで `CLAUDE.md` がプロジェクトルートに生成される。npx でも動く。

```bash
npx @orbit-cli/core scan -g
```

### 生成されるファイルの中身

実際に Next.js + Drizzle ORM のSaaSプロジェクトに対して `orbit scan -g` を実行した例がこちら。

```markdown
# Project: example-nextjs-saas

## Tech Stack
React 19.0.0 / TypeScript / Tailwind CSS / Drizzle ORM

## Project Structure
- **Pages (4):** /dashboard /login /pricing /settings
- **API Routes (11):** GET, POST, PUT, PATCH
- **DB Tables (9):** users, sessions, teams, team_members, products,
  orders, order_items, subscriptions, audit_logs

## Key Files
- Largest: schema.ts (85 lines), billing-settings.tsx (34 lines),
  pricing-table.tsx (33 lines), data-table.tsx (32 lines)
- 38 source files, ~848 lines

## Exports
- **Components (22):** DashboardPage, RootLayout, LoginPage,
  PricingPage, SettingsPage, ProductList, RecentOrders, RevenueChart,
  StatsCards, Hero, PricingTable, BillingSettings, TeamSettings,
  Navbar, Sidebar, Badge, Button, Card, CardHeader, CardContent +2
- **Functions (27):** GET, POST, PUT, PATCH, useAuth, useLogout,
  useOrders, useOrderStats, useProducts, useCreateProduct, useTeam,
  useUpdateTeam, getSession +7
- **Types (14):** DB, CreateProductInput, CreateOrderInput, User,
  Team, Product, Order, Subscription, UserRole, OrderStatus, Plan,
  ApiResponse, PaginatedResponse, DashboardStats

## Import Graph
35 files, 81 local imports

Most imported modules:
- `@/lib/utils` (12 imports)
- `@/types` (10 imports)
- `@/lib/auth` (7 imports)
- `@/components/ui/button` (7 imports)
- `@/components/ui/card` (7 imports)
- `@/db` (6 imports)
- `@/db/schema` (6 imports)

## Environment Variables
DATABASE_URL, NEXT_PUBLIC_URL, STRIPE_SECRET_KEY, ...

## Scripts
- `dev`: `next dev`
- `build`: `next build`
- `start`: `next start`
- `lint`: `eslint .`
- `db:push`: `drizzle-kit push`
```

たった一つのコマンドで、プロジェクトの全体像がここまでまとまる。これを Claude Code に渡せば、「このプロジェクトは Next.js + Drizzle で、`@/lib/utils` がユーティリティのハブになっていて、DBには users, teams, products, orders というテーブルがある」ということをAIが即座に把握できる。

### 出力フォーマットの切り替え

Claude Code 以外のAIツールを使っている場合は、`--target` でフォーマットを切り替えられる。

```bash
orbit scan -g                          # CLAUDE.md
orbit scan -g --target cursor          # .cursorrules
orbit scan -g --target copilot         # .github/copilot-instructions.md
orbit scan -g --target windsurf        # .windsurfrules
orbit scan -g --target cursor-mdc      # .cursor/rules/*.mdc (モジュラー形式)
```

## 主な機能

### `orbit scan -g` — コンテキストファイル生成

前述のとおり、メインの機能。解析される項目は以下の通り。

| 検出項目 | 内容 |
|---------|------|
| Tech Stack | フレームワーク、言語、パッケージマネージャー、Node バージョン |
| Project Structure | ページ、API ルート、DB テーブル |
| Import Graph | モジュール間の依存関係、インポート数でランキング |
| Exports | コンポーネント、関数、型のカタログ |
| Dependencies | カテゴリ別の依存パッケージ（UI, DB, Auth, etc.） |
| Git Status | ブランチ、直近コミット、未コミットの変更 |
| Code Metrics | ファイル数、行数、最大ファイル |
| Deployment | プラットフォーム（Vercel, AWS 等）とCI設定 |
| Environment Variables | 変数名のみ検出（値は一切読まない） |
| Scripts | npm/pnpm スクリプト |

### `orbit impact <file>` — 影響範囲分析

あるファイルを変更したときに、どこまで影響が波及するかを分析する。

```bash
orbit impact src/lib/auth.ts
```

import graph を辿って、直接・間接的にそのファイルに依存しているモジュールを一覧表示する。リファクタリング前に「これ変えたら何が壊れそうか」を把握するのに使える。

### `orbit watch` — ファイル監視 & 自動再生成

```bash
orbit watch
```

ファイルの変更を監視して、自動でコンテキストファイルを再生成する。開発中にルーティングを追加したりスキーマを変更したりしても、コンテキストが常に最新の状態に保たれる。

### `orbit scan` (ファイル生成なし) — プロジェクト概要の表示

`-g` なしで実行すると、ファイルを生成せずにプロジェクトの解析結果をターミナルに表示する。プロジェクトの概要をさっと確認したいときに便利。

```bash
orbit scan
```

### カスタムルールの保持

生成されたコンテキストファイルには、手書きのルールを追記できるセクションがある。

```markdown
<!-- ORBIT:USER-START -->
## コーディング規約
- `const` を `let` より優先
- API ルートは必ず Zod でバリデーション
- コンポーネントは named export を使う
<!-- ORBIT:USER-END -->
```

`orbit scan -g` を再実行しても、このマーカー内のテキストは保持される。自動生成の構造情報とチーム固有のルールを共存させられる。

## 技術的な仕組み

Zenn 読者向けに、内部の実装について少し掘り下げる。

### 全体構成

Orbit は TypeScript + Commander.js で構築されている。主要なモジュールは2つ。

- **detector.ts** — プロジェクトの構造をスキャンする解析エンジン
- **context-generator.ts** — 解析結果を Markdown のコンテキストファイルに変換

`detector.ts` がスキャンした結果を `ScanResult` 型で返し、`context-generator.ts` がそれを整形して出力する。

```typescript
// ScanResult の主要フィールド（抜粋）
export interface ScanResult {
  techStack: string[];
  packageManager: string | null;
  structure: {
    pages: string[];
    apiRoutes: { method: string; path: string }[];
    dbTables: { name: string; columns: number }[];
  };
  exports: {
    file: string;
    name: string;
    kind: 'function' | 'component' | 'type' | 'const';
  }[];
  importGraph: { file: string; imports: string[] }[];
  codeMetrics: {
    totalFiles: number;
    totalLines: number;
    largestFiles: { path: string; lines: number }[];
  };
  envVars: string[];
  scripts: Record<string, string>;
  // ...
}
```

### 正規表現ベースの軽量パーサー

Orbit はソースコードの解析に AST（抽象構文木）を使っていない。代わりに正規表現ベースの軽量パーサーを採用している。

理由は単純で、速度とポータビリティだ。AST パーサー（ts-morph や @babel/parser 等）を使うと依存関係が大きくなるし、パース速度もそのぶん遅くなる。Orbit がやりたいのは「export 文を見つける」「import 文を見つける」「テーブル定義を見つける」程度のパターンマッチングなので、正規表現で十分だし、むしろその方が高速に動く。

`package.json` の `dependencies` からフレームワーク検出、ファイルパスのパターンからルーティング抽出、export/import 文の正規表現マッチからエクスポートカタログとインポートグラフを構築する。

### Import Graph とハブモジュールのランキング

Import graph の構築は、各ファイルの `import` 文から依存先を抽出し、有向グラフとして組み立てる。

```
src/components/Dashboard.tsx → src/lib/utils.ts
src/components/Dashboard.tsx → src/lib/auth.ts
src/pages/api/users.ts      → src/lib/auth.ts
src/pages/api/users.ts      → src/db/schema.ts
```

このグラフで「多くのファイルから import されているモジュール」をハブモジュールとしてランキングする。AIにとって、「`@/lib/utils` は12ファイルから使われている」という情報は非常に有用で、コードを書く際にこのモジュールの存在を前提にしてくれるようになる。

`orbit impact <file>` でも同じグラフを使っていて、BFS（幅優先探索）で依存の連鎖を辿り、影響範囲を可視化する。

### DB スキーマの自動検出

Drizzle ORM と Prisma のスキーマ定義に対応している。

- **Drizzle**: `pgTable()` / `mysqlTable()` / `sqliteTable()` の呼び出しを検出
- **Prisma**: `schema.prisma` ファイルの `model` 定義を検出

テーブル名とカラム数を抽出し、コンテキストに含める。AIがDBの設計を把握した上でクエリやマイグレーションを提案してくれるようになる。

### 環境変数の検出

`process.env.XXX` や `.env.example` からの変数名検出も行う。値は一切読まず、名前だけを収集する。これにより「このプロジェクトでは `DATABASE_URL` と `STRIPE_SECRET_KEY` が必要」という情報がコンテキストに入り、AIが `.env` の設定を案内する際に正確な変数名を使えるようになる。

## Repomix との違い

同じ領域のツールとして [Repomix](https://github.com/yamadashy/repomix) がある。ここでは公平に比較したい。

### アプローチの違い

| | Orbit | Repomix |
|---|---|---|
| **何を出力するか** | プロジェクトの構造情報（地図） | ソースコードの全量テキスト |
| **トークン消費** | 小さい（構造だけなので） | 大きい（ソースコード全体） |
| **AIに渡す情報** | 「どういうプロジェクトか」 | 「コードの中身」 |
| **対応フォーマット** | CLAUDE.md / .cursorrules / copilot / windsurf | XML / Markdown / Plain text |

### 使い分け

Repomix は「AIにコードを丸ごと読ませたい」ときに強い。例えば「このリポジトリ全体をレビューして」「このコードベースの問題点を指摘して」のような、コードそのものが分析対象になるケースだ。

Orbit は「AIと一緒にコードを書く」ときに強い。開発中に常にプロジェクト構造を把握させておくことで、AIが既存のコンポーネントや関数を活用した提案をしてくれるようになる。import graph でハブモジュールがわかっているから、「`@/lib/utils` の `cn()` を使って」みたいな適切な提案ができる。

正直、どちらか一方ではなく併用が最も効果的だと思う。Orbit でプロジェクト構造を常に共有しつつ、必要なときに Repomix で特定ディレクトリのコードを渡す、というワークフローが個人的にはうまくいっている。

## MCP サーバー

Orbit は MCP（Model Context Protocol）サーバーとしても動作する。

```bash
orbit mcp-serve
```

Claude Desktop などの MCP クライアントから、以下のツールをリアルタイムで呼び出せる。

- `orbit_getProjectContext` — プロジェクト全体のコンテキスト取得
- `orbit_getModuleContext` — 特定モジュールのコンテキスト取得
- `orbit_getFileRelations` — ファイルの依存関係取得

ファイルを生成して渡すのではなく、AIツールが直接 Orbit に問い合わせる形。MCP 対応のエディタが増えてきているので、今後はこちらの使い方が主流になるかもしれない。

## おわりに

Orbit は、AIアシスタントに「プロジェクトの地図」を渡すためのツールだ。CLAUDE.md や .cursorrules を手書きで維持する手間をなくし、常に最新の構造情報をAIに共有できる。

```bash
npx @orbit-cli/core scan -g
```

この一行で試せるので、ぜひ自分のプロジェクトで使ってみてほしい。

GitHub リポジトリはこちら。Star やフィードバックをもらえると開発の励みになります。Issue での機能リクエストも歓迎。

https://github.com/s4kuraN4gi/orbit-app

npm パッケージ:

https://www.npmjs.com/package/@orbit-cli/core
