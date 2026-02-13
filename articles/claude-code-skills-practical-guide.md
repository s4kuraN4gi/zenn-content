---
title: "Claude Codeの「Skills」で開発が変わる — 既存スキル活用から自作まで実践3ステップ"
emoji: "🛠"
type: "tech"
topics: ["ClaudeCode", "Claude", "AI", "個人開発", "効率化"]
published: true
---

## はじめに

Claude Codeに「LPを作って」と指示して、それなりのページが出てくる。この体験だけで満足していないだろうか。実は Claude Code には **Skills（スキル）** という仕組みがあり、これを使うと出力品質が別次元に上がる。

この記事では、実際にSaaS風のランディングページを作りながら、以下の3ステップでSkillsの使い方を体系的に学ぶ。

1. 既存スキルを使いこなす
2. スキルの仕組みを理解する
3. 自作スキルを作る

:::message
この記事は、Claude Codeの基本的な操作（ファイル作成、コード生成の指示出し等）ができる方を対象としている。
:::

## スキルとは何か — 30秒で理解する

スキルの正体は**「プロンプトエンジニアリングのパッケージ」**だ。

通常の指示では、毎回「フォントはこうして」「レイアウトはこうして」と書く必要がある。スキルを使うと、そうした品質制約が**自動的にプロンプトに注入される**。

```
通常の指示:
  ユーザーの指示 → Claude → 出力

スキル経由:
  ユーザーの指示 + スキルの品質制約 → Claude → 高品質な出力
```

スキルは `/スキル名` で呼び出せる。たとえば `/frontend-design` と打つだけで、デザイン品質を担保する制約が自動適用される。

## Step 1: 既存スキルを使い倒す

このステップでは、3つの既存スキルを実際に使って効果を体感する。

### 1-1. `/frontend-design` — デザイン品質が変わる

架空のAI画像生成SaaS「PixelMind AI」のLPを作る。

通常の指示で「LPを作って」と言うと、ダークモード + ネオングラデーション + グラスモーフィズムの「定番パターン」になりがちだ。どのSaaSサイトでも見るデザインで、AIが好むテンプレートに収束する。

`/frontend-design` スキル経由で作り直すと、全く違うものが出てくる。

![PixelMind LP — /frontend-design スキルで生成したエディトリアルデザイン](/images/pixelmind-lp-v2.png)

Instrument Serif + Outfit のフォントペアリング、バーミリオン1色のアクセント、非対称グリッド。エディトリアル/ラグジュアリー方向のデザインだ。

:::message
**なぜ品質が変わるのか？** `/frontend-design` の中身には以下のようなルールが書かれている:
- Inter, Roboto, Arial 等の定番フォント使用禁止
- 紫グラデーション on 白背景の禁止
- Cookie-cutter レイアウトの排除
- Purpose → Tone → Differentiation の設計思考フレームワーク

スキルが「避けるべきこと」を明示しているから、AIの定番パターンから脱却できる。
:::

#### スキルを最大限活用するコツ

「やってほしいこと」だけでなく **「やってほしくないこと」** も指示に入れると効果的。

```diff
- ダークモードでかっこいいLPを作って
+ エディトリアル/マガジン系のLPを作って
+ グラデーション禁止、グラスモーフィズム禁止
+ カード並びではなくストーリーテリング型レイアウト
```

この差分だけで出力が大きく変わる。

### 1-2. `/dep-visualizer` — プロジェクト構造を一目で把握

Next.jsプロジェクトの `import` 依存関係を、インタラクティブなHTMLグラフとして可視化するスキル。

```bash
/dep-visualizer src/
```

![dep-visualizer レポート — 13ファイルの依存関係グラフ](/images/dep-visualizer-report.png)

13ファイル・16依存関係を解析し、以下を検出できた:
- **孤立ファイル**: どこからもimportされていない `analytics.ts` を検出
- **依存のハブ**: 多くのファイルから参照される `utils.ts` を可視化
- **依存の深さ**: page.tsx → Header → NavLinks のような依存チェーン

:::message
**個人開発者にとっての価値**: コードレビューしてくれる人がいなくても、循環依存や不要ファイルをツールで発見できる。
:::

### 1-3. `stripe:*` — 決済統合を3スキルでカバー

Stripe関連は3つのスキルがあり、開発フェーズごとに使い分ける。

| フェーズ | スキル | 得られるもの |
|---------|--------|-------------|
| 設計段階 | `/stripe:stripe-best-practices` | 推奨アーキテクチャ、避けるべきAPI |
| 開発中 | `/stripe:explain-error` | エラーの原因・解決策・コード例 |
| テスト | `/stripe:test-cards` | テストカード番号一覧 |

特に `best-practices` は「やってはいけないこと」を教えてくれるのが大きい。

:::message alert
Stripeの非推奨APIを使ってしまう事故を防げる:
- Charges API → Checkout Sessions を使う
- Card Element → Payment Element を使う
- Sources API → Setup Intents を使う
:::

## Step 2: スキルの仕組みを理解する

既存スキルの威力を体感したところで、中身を見てみよう。

### スキルの構造はシンプル

スキルは `~/.claude/skills/スキル名/` ディレクトリに配置する。必須ファイルは **SKILL.md の1つだけ**。

```
~/.claude/skills/my-skill/
├── SKILL.md           ← 唯一の必須ファイル
├── scripts/           ← 実行スクリプト（任意）
├── references/        ← 参照ドキュメント（任意）
└── assets/            ← テンプレート等（任意）
```

### SKILL.md の書き方

```markdown:~/.claude/skills/my-skill/SKILL.md
---
name: my-skill
description: >
  ○○するときに使うスキル
---

# スキルタイトル

ここに実行手順・ルール・制約を書く。
Claudeはこの内容をプロンプトとして受け取って実行する。
```

YAMLフロントマター（`name` と `description`）+ Markdown本文。これだけでスキルが作れる。

:::details オプションのフロントマター設定
```yaml
---
name: my-skill
description: 説明文
trigger: "発動キーワード"
disable-model-invocation: true   # ユーザー明示実行のみ
allowed-tools: Read, Glob, Bash  # 使えるツールを制限
context: fork                    # サブエージェントで実行
---
```
:::

### スキル vs プラグイン

| | スキル | プラグイン |
|---|--------|-----------|
| 構成 | SKILL.md 1ファイル〜 | plugin.json + skills/ + commands/ 等 |
| 配置 | `~/.claude/skills/` | `~/.claude/plugins/` |
| 配布 | 手動コピー | マーケットプレイス経由 |
| 用途 | 個人の効率化 | チーム・コミュニティへの配布 |

個人開発なら**スキルで十分**。プラグインはスキルの集合体であり、配布の仕組みが付いたもの。

## Step 3: 自作スキルを作る

仕組みが分かったところで、実際に自作スキルを作ってみよう。

### 例: Zenn記事ドラフト生成スキル

Zennに技術記事を書く際、毎回以下を考える必要がある:
- フロントマターのフォーマット（emoji, type, topics...）
- 記事の構成（リード文、セクション分割、まとめ）
- 読みやすい文章のルール（1文の長さ、具体例の挿入）
- SEOを意識したタイトル付け

これをスキル化すると `/zenn-draft テーマ名` の一言で全て自動適用される。

```markdown:~/.claude/skills/zenn-draft/SKILL.md
---
name: zenn-draft
description: >
  Zenn記事のドラフトを構成・執筆するスキル。
---

# Zenn 記事ドラフト生成

## 記事の執筆ルール

### トーン & スタイル
- 客観的に書く。体験談は一人称OK
- 1文は60字以内を目安
- 抽象的な説明の後に必ず具体例を入れる

### 構成要素
- コードブロックには必ずファイル名を付ける
- 重要ポイントは :::message で囲む
- 注意事項は :::message alert で囲む

### SEO
- タイトルに具体的な数字や課題を含める
- topics は検索されやすいタグを3-5個
```

:::message
**スキル設計のコツ**: 「自分が毎回書いている指示」を振り返り、それをそのままSKILL.mdに書けばいい。スキルとは「繰り返し使うプロンプトの保存」に過ぎない。
:::

### references/ で知識を補完する

SKILL.md は軽量に保ち、詳細な仕様は `references/` に分離する。

```
~/.claude/skills/zenn-draft/
├── SKILL.md              ← 執筆ルール・構成テンプレ
└── references/
    └── zenn-format.md     ← Zennフォーマット仕様（frontmatter等）
```

SKILL.md の中で「Zennフォーマット仕様は `references/zenn-format.md` を参照」と書いておけば、Claudeが必要な時に自動的に読み込む。これを **Progressive Disclosure** パターンと呼ぶ。

## まとめ

Claude Code の Skills について、3ステップで学んだ。

- **Step 1**: 既存スキル（`/frontend-design`, `/dep-visualizer`, `stripe:*`）を使い、スキルがもたらす品質向上を体感した
- **Step 2**: スキルの正体は「SKILL.md に書かれたプロンプト」。構造はシンプルで、1ファイルから作れる
- **Step 3**: 自分専用のスキル（Zenn記事ドラフト）を実際に作成し、繰り返す作業を自動化した

### 次のアクション

1. `~/.claude/skills/` ディレクトリを作り、最初のスキルを書いてみる
2. 「毎回同じ指示を書いている」作業がないか振り返る
3. それをSKILL.mdに書き出す — それだけでスキルの完成

スキルは難しい技術ではない。**繰り返すプロンプトを保存する仕組み**だ。一度作れば、以降のすべてのプロジェクトで使い回せる。

---

:::message
**次の記事**: Skills の「その先」に興味がある方へ。Claude Code には **MCP（外部ツール連携）**、**CLAUDE.md（プロジェクトルール）**、**Hooks（自動化）** といった拡張機能がある。次回の記事「Claude Codeを"拡張"する — MCP・CLAUDE.md・Hooksで自分だけの開発環境を作る」では、これらを使って Claude Code をさらにカスタマイズする方法を解説する。
:::
