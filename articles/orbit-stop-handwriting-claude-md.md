---
title: "CLAUDE.md を手書きするのをやめた話 — Orbit で AI コンテキストを自動生成する"
emoji: "✨"
type: "tech"
topics: ["claude", "cursor", "nextjs", "ai", "devtools"]
published: true
---

## 手書き CLAUDE.md、つらくないですか？

Claude Code の `CLAUDE.md`、Cursor の `.cursorrules`、GitHub Copilot の `copilot-instructions.md`。AI アシスタントにプロジェクトの構造を伝えるためのコンテキストファイルを、手で書いている人は多いと思います。

ただ、これがなかなかしんどい。そもそも何を書けばいいのかわからないし、書いたとしてもファイルを追加したりリファクタしたりするたびに古くなっていく。「あとで更新しよう」と思って放置した結果、AI が存在しないファイルを参照し始めて逆に邪魔になる、という経験をした人もいるのではないでしょうか。

特に **import graph**（どのモジュールがどこから参照されているか）や**ハブモジュールのランキング**なんて、手書きでは到底カバーできません。でもこの情報があると AI の提案精度がかなり変わってきます。プロジェクト内で `@/lib/utils` が 20 箇所から import されている、という事実を知っているかどうかで、AI が生成するコードの質は全然違います。

## Orbit で自動化してみた

[Orbit](https://github.com/s4kuraN4gi/orbit-app) はプロジェクトを解析して、構造化された AI コンテキストファイルを自動生成する CLI ツールです。OSS で完全無料。やることは 3 ステップだけです。

### Step 1: インストール

```bash
npm i -g @orbit-cli/core
```

### Step 2: プロジェクトのルートでスキャン

```bash
orbit scan -g
```

`-g` は `--generate` の省略形で、スキャン結果から `CLAUDE.md` を自動生成します。

### Step 3: 生成された CLAUDE.md を確認

Next.js の SaaS プロジェクトで実行した結果がこちらです。

```markdown
# Project: example-nextjs-saas

## Tech Stack
React 19.0.0 / TypeScript / Tailwind CSS / Drizzle ORM

## Project Structure
- **Pages (4):** /dashboard /login /pricing /settings
- **API Routes (11):** GET, POST, PUT, PATCH
- **DB Tables (9):** users, sessions, teams, team_members, products, orders, order_items, subscriptions, audit_logs

## Key Files
- Largest: schema.ts (85 lines), billing-settings.tsx (34 lines), pricing-table.tsx (33 lines), data-table.tsx (32 lines), index.ts (31 lines)
- 38 source files, ~848 lines

## Exports
- **Components (22):** DashboardPage, RootLayout, LoginPage, PricingPage, SettingsPage, ProductList, RecentOrders, RevenueChart, StatsCards, Hero, PricingTable, BillingSettings, TeamSettings, Navbar, Sidebar, Badge, Button, Card, CardHeader, CardContent +2
- **Functions (27):** GET, GET, POST, GET, POST, GET, POST, POST, PUT, GET, PATCH, useAuth, useLogout, useOrders, useOrderStats, useProducts, useCreateProduct, useTeam, useUpdateTeam, getSession +7
- **Types (14):** DB, CreateProductInput, CreateOrderInput, User, Team, Product, Order, Subscription, UserRole, OrderStatus, Plan, ApiResponse, PaginatedResponse, DashboardStats

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
- `@/components/ui/badge` (5 imports)
- `@/lib/validators` (4 imports)
- `@/lib/stripe` (2 imports)

## Environment Variables
DATABASE_URL, NEXT_PUBLIC_URL, STRIPE_ENTERPRISE_PRICE_ID, STRIPE_PRO_PRICE_ID, STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET

## Scripts
- `dev`: `next dev`
- `build`: `next build`
- `start`: `next start`
- `lint`: `eslint .`
- `db:push`: `drizzle-kit push`
- `db:studio`: `drizzle-kit studio`
```

コマンド一発でここまでの情報が整理されます。手書きでは絶対にやらないレベルの網羅性です。

## 生成されるコンテキストの中身

各セクションが何を意味しているか、もう少し詳しく見ていきます。

### Tech Stack

```markdown
## Tech Stack
React 19.0.0 / TypeScript / Tailwind CSS / Drizzle ORM
```

`package.json` や設定ファイルを解析して、フレームワーク・言語・主要ライブラリを自動検出します。AI にとっては「このプロジェクトは React 19 を使っている」という情報があるだけで、古い API を提案するミスが減ります。

### Project Structure

```markdown
## Project Structure
- **Pages (4):** /dashboard /login /pricing /settings
- **API Routes (11):** GET, POST, PUT, PATCH
- **DB Tables (9):** users, sessions, teams, team_members, ...
```

ページ数、API ルート数、DB テーブル数とその一覧。プロジェクトの全体像を AI に一瞬で伝えられます。

### Key Files

```markdown
## Key Files
- Largest: schema.ts (85 lines), billing-settings.tsx (34 lines), ...
- 38 source files, ~848 lines
```

行数の多いファイル = ロジックが集中しているファイルです。AI がコードを修正するとき、どのファイルを重点的に見るべきか判断できます。

### Exports

```markdown
## Exports
- **Components (22):** DashboardPage, RootLayout, ...
- **Functions (27):** GET, POST, useAuth, useLogout, ...
- **Types (14):** DB, CreateProductInput, User, ...
```

プロジェクトで公開されているコンポーネント・関数・型の一覧。AI が「このプロジェクトには `useAuth` というフックがある」と知っていれば、新しいページを作るときに自然とそれを使ってくれます。

### Import Graph

```markdown
## Import Graph
35 files, 81 local imports

Most imported modules:
- `@/lib/utils` (12 imports)
- `@/types` (10 imports)
- `@/lib/auth` (7 imports)
```

**これが一番手書きでは書けないセクション**です。どのモジュールがプロジェクトのハブになっているかをランキング形式で示します。AI はこの情報をもとに、既存のユーティリティやパターンに沿ったコードを生成してくれるようになります。

### Environment Variables

```markdown
## Environment Variables
DATABASE_URL, NEXT_PUBLIC_URL, STRIPE_SECRET_KEY, ...
```

環境変数の **名前だけ** を列挙します（値は含まれません）。AI が「この環境変数を使え」と正しく提案できるようになります。

## 他のフォーマットにも対応

Orbit は `CLAUDE.md` だけでなく、主要な AI ツールのフォーマットすべてに対応しています。

```bash
# Cursor 用
orbit scan -g --target cursor

# GitHub Copilot 用
orbit scan -g --target copilot

# Windsurf 用
orbit scan -g --target windsurf
```

1 つのコマンドでどの AI ツールにも対応できるので、チーム内で使っているツールがバラバラでも問題ありません。

## カスタマイズ方法

「自動生成は便利だけど、プロジェクト固有のルールも書きたい」という場合は、`ORBIT:USER-START` / `ORBIT:USER-END` マーカーを使います。

```markdown
<!-- ORBIT:AUTO-GENERATED - Do not edit above this line -->

## Custom Rules
<!-- ORBIT:USER-START -->
- API レスポンスは必ず camelCase で返す
- エラーハンドリングは AppError クラスを使う
- テストは Vitest で書く
<!-- ORBIT:USER-END -->
```

マーカーで囲まれた部分は、`orbit scan -g` で再生成しても上書きされません。自動生成部分は常に最新に、手書き部分はそのまま保持されます。

## watch モードで常に最新に

開発中にファイルを追加・削除・リファクタするたびに手動で再生成するのは面倒です。そこで `watch` モードを使います。

```bash
orbit watch
```

ファイルの変更を検知して `CLAUDE.md` を自動で再生成してくれます。開発中は常にコンテキストが最新の状態に保たれるので、AI に古い情報を渡してしまう心配がなくなります。

## Before / After

### Before（手書き or コンテキストなし）

- AI が存在しないファイルを import しようとする
- プロジェクトで使っている ORM を無視して生の SQL を書いてくる
- 既存のユーティリティ関数があるのに、同じ処理を一から書いてくる
- 環境変数名を間違える

### After（Orbit で自動生成）

- プロジェクト構造を理解した上でコードを提案してくれる
- 既存の `useAuth` フックや `@/lib/utils` を適切に使ってくれる
- DB スキーマを把握しているので、正しいテーブル名・カラム名を使う
- Import Graph を参考に、プロジェクトの慣習に沿った import パスを書いてくれる

## まとめ

`CLAUDE.md` を手書きするのをやめて Orbit で自動生成に切り替えた結果、AI アシスタントの提案精度が体感でかなり向上しました。特に Import Graph の情報が効いている実感があります。

コマンド 2 つで試せるので、ぜひやってみてください。

```bash
npm i -g @orbit-cli/core
orbit scan -g
```

GitHub にスターをもらえると開発の励みになります。完全無料・OSS で、アカウント登録なしで使えます。

https://github.com/s4kuraN4gi/orbit-app

npm パッケージ:

https://www.npmjs.com/package/@orbit-cli/core
