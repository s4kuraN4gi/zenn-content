---
title: "Claude Codeを"拡張"する — MCP・CLAUDE.md・Hooksで自分だけの開発環境を作る"
emoji: "🔌"
type: "tech"
topics: ["ClaudeCode", "Claude", "AI", "MCP", "個人開発"]
published: false
---

## はじめに

前回の記事では、Claude Code の **Skills** を使って出力品質を上げる方法を紹介した。Skills は「プロンプトのパッケージ化」であり、Claude に**知識**を与える仕組みだった。

この記事では、その先にある3つの拡張機能を扱う。

1. **MCP** — Claude に「手足」を与え、外部ツールを直接操作させる
2. **CLAUDE.md** — プロジェクトの「ルールブック」を自動読み込みさせる
3. **Hooks** — Claude の動作に「自動パイプライン」を組み込む

Skills が「何を知っているか」を拡張するなら、この3つは「何ができるか」「どう振る舞うか」を拡張する。

:::message
この記事は、Claude Code の基本操作と Skills の概念を理解している方を対象としている。Skills については[前回の記事](https://zenn.dev/)を参照。
:::

## MCP — Claude に「手足」を与える

### Skills との根本的な違い

Skills と MCP は混同しやすいが、役割が全く違う。

| | Skills | MCP |
|---|--------|-----|
| 何を与えるか | **知識**（プロンプト注入） | **手足**（外部ツール操作） |
| 仕組み | SKILL.md の内容をコンテキストに追加 | 外部サーバー経由でAPIを直接呼び出し |
| 例 | 「Stripeのベストプラクティス」を教える | Stripe APIを直接叩いて決済を作成する |

Skills は Claude に「こう考えろ」と教える。MCP は Claude に「これを使え」と道具を渡す。

### MCPサーバーの3タイプ

MCPサーバーには接続方式が3つある。

```
HTTP型:  Claude ←→ SaaS API（Supabase, Stripe, GitHub）
SSE型:   Claude ←→ リアルタイム通信（Slack, Asana）
stdio型: Claude ←→ ローカルプロセス（Playwright, Firebase）
```

**個人開発で最も試しやすいのはstdio型**。アカウント不要、ローカルで完結する。

### 実践: Playwright MCP を導入する

ブラウザ操作を Claude に任せられる Playwright MCP を設定してみよう。

プロジェクトルートに `.mcp.json` を配置する。

```json:.mcp.json
{
  "mcpServers": {
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

これだけで Claude が「ブラウザを開いてスクリーンショットを撮る」「フォームに入力してテストする」といった操作を実行できるようになる。

:::message
**MCP設定の3つのスコープ**:
- **プラグイン内蔵** `.mcp.json` → プラグインをインストールするだけで自動設定
- **グローバル** `~/.claude.json` → 全プロジェクトで使う MCP
- **プロジェクト固有** `プロジェクトルート/.mcp.json` → そのプロジェクトだけ
:::

### 利用可能な主要MCPサーバー

個人開発で特に有用なものを挙げる。

| サーバー | タイプ | 用途 | 認証 |
|---------|--------|------|------|
| Playwright | stdio | ブラウザ操作・E2Eテスト | 不要 |
| Supabase | HTTP | DB・認証・Storage操作 | サーバー側 |
| Stripe | HTTP | 決済管理 | APIキー |
| GitHub | HTTP | リポジトリ・Issue管理 | トークン |
| Firebase | stdio | Googleインフラ操作 | 不要（ローカル） |
| Slack | SSE | メッセージ連携 | OAuth |
| Linear | HTTP | プロジェクト管理 | APIキー |

:::message alert
HTTP型のMCPサーバーはAPIキーや環境変数での認証が必要。`"headers": {"Authorization": "Bearer ${ENV_VAR}"}` のように環境変数を参照する形式が一般的。**APIキーを `.mcp.json` にハードコードしないこと**。
:::

### MCP の本質

従来は人間がAPIドキュメントを読み、コードを書いてAPIを呼んでいた。MCP はこの構造を変える。**Claude 自身がAPIを直接操作する**。

たとえば Supabase MCP を設定すれば、「ユーザーテーブルを作って」と言うだけで Claude が SQL を生成し、Supabase API 経由で実行する。SQL が書けなくてもデータベースが使える。

## CLAUDE.md — プロジェクトの「ルールブック」

### Skills と何が違うのか

Skills は `/<スキル名>` で明示的に呼び出すもの。CLAUDE.md は**セッション開始時に自動で読み込まれるもの**。

| | Skills | CLAUDE.md |
|---|--------|-----------|
| スコープ | タスク単位 | プロジェクト単位 |
| 起動 | 明示的に `/スキル名` で呼ぶ | セッション開始時に自動読み込み |
| 内容 | タスクの実行手順・ルール | プロジェクトの規約・アーキテクチャ |

**「毎回同じことを言っている」はCLAUDE.mdに書くべきサイン**。実際、この記事を書いているプロジェクトでも毎回「article-log.mdに追記して」「Zennフォーマットで」と伝えていた。それをCLAUDE.mdに書いたら、以降は言わなくてもClaudeが従うようになった。

### 実践: 実際にCLAUDE.mdを書いてみた

このZenn記事プロジェクトに作成したCLAUDE.mdの内容がこれだ。

```markdown:CLAUDE.md
# Skills プロジェクト

Claude Code の拡張機能を試しながら、Zenn記事の素材を蓄積するプロジェクト。

記事全体の構成・計画は @article-concept.md を参照。

## プロジェクト構成
- `article-log.md` — 各Levelの作業ログ・記事素材
- `articles/` — Zenn記事のドラフト（Zennフォーマット準拠）
- `article-concept.md` — 記事全体の構成・計画

## 記事の執筆ルール
- Zenn記事は `articles/` ディレクトリに配置する
- frontmatter（title, emoji, type, topics, published）を必ず含める
- `published: false` で保存し、手動で公開する
- スクリーンショットが必要な箇所は `<!-- TODO: -->` コメントで残す

## 作業ログのルール
- 新しい作業の知見は `article-log.md` に追記する
- 見出しは `## Level X-Y: タイトル` の形式
- 「やったこと」「学び」「記事に書くポイント」「スクショ撮影ポイント」を記録

## コマンド
- 記事プレビュー: `npx zenn preview`
- 新規記事作成: `npx zenn new:article --slug スラッグ名`
```

ポイントは `@article-concept.md` の行。`@ファイルパス` と書くだけで、そのファイルの内容がClaudeに自動参照される。CLAUDE.md本体は軽量に保ち、詳細は別ファイルに任せる — 前回の記事で紹介した Skills の `references/` と同じ **Progressive Disclosure** の考え方だ。

### 何を書くか迷ったら

テンプレを考える必要はない。**自分が過去のセッションで繰り返し言ったこと**を振り返ればいい。

```diff
- 毎回伝えていたこと:
  「article-log.mdに知見を追記して」
  「記事はarticles/に置いて」
  「Zennフォーマットで書いて」

+ CLAUDE.md に書いたこと:
  作業ログのルール: article-log.md に追記する
  記事の執筆ルール: articles/ に配置、frontmatter必須
```

`/init` コマンドで自動生成もできるが、プロジェクトの文脈を知っている自分が書いた方が的確だった。

### CLAUDE.md の階層構造

CLAUDE.md は複数のスコープで使い分けられる。

```
組織全体    /Library/.../ClaudeCode/CLAUDE.md   ← IT管理者が配布
  ↓
チーム共有  ./CLAUDE.md                          ← Gitにコミット
  ↓
トピック別  ./.claude/rules/*.md                 ← ファイル単位でルール分離
  ↓
個人全体    ~/.claude/CLAUDE.md                  ← 自分の全プロジェクト
  ↓
個人ローカル ./CLAUDE.local.md                   ← このプロジェクトだけ、Git対象外
```

**個人開発者なら `./CLAUDE.md` 1ファイルで始めて十分**。共通設定が出てきたら `~/.claude/CLAUDE.md` に分離する。`.claude/rules/` はパス限定ルールが必要になってから考えればいい。

:::message
**パス限定ルール**: `.claude/rules/` に置いたMarkdownファイルにYAMLフロントマターで `paths: ["src/app/api/**/*.ts"]` を指定すると、そのパスのファイルを編集するときだけルールが適用される。API開発ルール、テストの書き方ルール等をトピック別に分離するのに便利。
:::

## Hooks — 自動パイプラインを組み込む

### Hooks の本質

Hooks は **CI/CD パイプラインのローカル版**。「Claude が何かしたら、自動で○○する」を実現する。

```
従来の CI/CD:
  git push → GitHub Actions → lint → test → deploy

Hooks:
  Claude がファイル編集 → 自動lint → 自動テスト → 結果をフィードバック
```

CI/CD を組む前の段階で、ローカルで品質チェックを自動化できる。

### 実践: Markdown編集後に文字数を自動カウントする

この記事プロジェクトで実際に作ったフックを紹介する。Zenn記事は5,000〜8,000字が適切な範囲なので、**ファイル編集のたびに文字数をカウントして表示する**フックを設定した。

まず、フックスクリプトを作成する。

```bash:.claude/hooks/count-chars.sh
#!/bin/bash
# stdin から JSON を受け取り、編集されたファイルが .md なら文字数をカウント
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // .tool_input.filePath // empty')

# .md ファイル以外は無視
if [[ "$FILE_PATH" != *.md ]]; then
  exit 0
fi

if [[ ! -f "$FILE_PATH" ]]; then
  exit 0
fi

CHAR_COUNT=$(wc -m < "$FILE_PATH" | tr -d ' ')
LINE_COUNT=$(wc -l < "$FILE_PATH" | tr -d ' ')
FILENAME=$(basename "$FILE_PATH")

echo "${FILENAME}: ${CHAR_COUNT}文字, ${LINE_COUNT}行"
exit 0
```

次に、`.claude/settings.local.json` にフックを登録する。

```json:.claude/settings.local.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/count-chars.sh"
          }
        ]
      }
    ]
  }
}
```

フックスクリプトの基本パターンは **stdin で JSON を受信 → jq でパース → 処理 → exit code で結果通知**。`exit 0` で成功（stdout がフィードバックとして Claude に返る）、`exit 2` でブロック（stderr がエラーメッセージになる）。

:::message alert
**注意: フックはセッション開始時にスナップショットされる**。セッション途中でフックを追加しても、発火するのは次回セッションから。これは実際にハマったポイントだった。
:::

### フックで何ができるか — 主要イベント5つ

Hooks には14種類のイベントがあるが、実用上押さえるべきはこの5つ。

| イベント | タイミング | 用途 |
|---------|-----------|------|
| `PostToolUse` | ツール実行**後** | 自動リント・テスト・文字数カウント |
| `PreToolUse` | ツール実行**前** | 危険なコマンドのブロック |
| `Stop` | Claude応答完了時 | 最終チェック |
| `SessionStart` | セッション開始時 | 環境変数の注入 |
| `UserPromptSubmit` | ユーザー入力送信時 | 入力のバリデーション |

**最初の1つは `PostToolUse` + `Write|Edit` matcher がおすすめ**。ファイル編集のたびに何かが走る、という体験が最もフックの威力を実感できる。

### 応用: 危険コマンドのブロック

`PreToolUse` を使えば、ツール実行**前**に介入できる。たとえば `rm -rf` のような破壊的コマンドをブロックする。

```bash:.claude/hooks/block-dangerous.sh
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

if echo "$COMMAND" | grep -qE 'rm\s+-rf|drop\s+table|--force'; then
  echo "Blocked: destructive command detected" >&2
  exit 2  # exit 2 = ブロック
fi

exit 0
```

### 応用: 非同期フック

`"async": true` を指定すると、フックがバックグラウンドで実行される。テストのように時間がかかる処理でも、メインの作業をブロックしない。

```json
{
  "type": "command",
  "command": ".claude/hooks/run-tests.sh",
  "async": true,
  "timeout": 300
}
```

### フックの3タイプ

```
command:  シェルコマンドを実行（最も汎用的。上の例はすべてこれ）
prompt:   Claudeモデルにyes/no判定を委譲（コード不要で手軽）
agent:    マルチターンのサブエージェントで条件検証（最も高機能）
```

まずは **command** で始めるのが現実的。`/hooks` コマンドでインタラクティブに追加・削除もできるので、JSONを直接編集する必要もない。

:::message
**プロジェクトに合ったフックを作る**: コードプロジェクトなら自動リント、記事プロジェクトなら文字数カウント。「Claude が編集したあと、自分が毎回手動でやっていること」がフック化の候補になる。
:::

## 補足: Plan Mode — 考えてから動かす

CLAUDE.md と Hooks がプロジェクトの「常時設定」なら、Plan Mode は**一時的に Claude を読み取り専用にするモード**だ。

実はこの記事を書く過程で Plan Mode を使った。「記事を1本にまとめるか2本に分割するか」を決める必要があったとき、Shift+Tab で Plan Mode に入り、Claude に既存ドラフトの文字数分析と構成案の提示だけをさせた。Edit や Write は使えないので、勝手にファイルを変更される心配がない。

```
Shift+Tab でモード切替:
  Normal → Auto-Accept → Plan Mode

Plan Mode でできること:
  Read, Glob, Grep（ファイル調査）
  WebSearch, WebFetch（情報収集）
  Task（調査エージェント）

Plan Mode でできないこと:
  Edit, Write, Bash（ファイル変更・コマンド実行）
```

**個人開発者にとっての最大の価値は「設計レビューの代替」**。一人開発ではレビュアーがいない。Plan Mode でまず計画を出させ、自分がレビュアーとして承認/修正する — この「計画→レビュー→実行」のサイクルが強制的に入る。

複数ファイルにまたがるリファクタリングや、方針が複数ありえる設計判断のときに使うのがおすすめ。逆に「このバグを直して」のような方針が明確なタスクでは不要。

## 3つの拡張機能の組み合わせ

MCP・CLAUDE.md・Hooks は単体でも有用だが、組み合わせると効果が倍増する。

```
CLAUDE.md:  「このプロジェクトではpnpmを使う」「テストはvitest」
    ↓
MCP:        Supabase MCPでDBを直接操作、Playwright MCPでE2Eテスト
    ↓
Hooks:      ファイル編集後に自動リント、テスト失敗時にフィードバック
```

### 個人開発者のための推奨セットアップ

| 優先度 | 機能 | 最初にやること |
|--------|------|---------------|
| 高 | CLAUDE.md | `/init` でプロジェクトルールを生成 |
| 高 | MCP | Playwright MCPを `.mcp.json` に追加 |
| 中 | Hooks | PostToolUse でリントを自動化 |
| 中 | CLAUDE.md rules | `.claude/rules/` でパス別ルールを追加 |
| 低 | Hooks（非同期） | テストのバックグラウンド実行 |

まずは CLAUDE.md と MCP から始め、Hooks は慣れてきてから導入するのが現実的。

## まとめ

Claude Code の3つの拡張機能について学んだ。

- **MCP**: Claude に「手足」を与える。外部ツール・APIを直接操作させ、ブラウザ操作やDB操作を自動化
- **CLAUDE.md**: プロジェクトの「ルールブック」。セッション開始時に自動で読み込まれ、規約やコマンドを毎回伝える必要がなくなる
- **Hooks**: CI/CDのローカル版。ファイル編集後の自動リント、危険コマンドのブロック、テストの自動実行

### Skills との関係を整理する

| 機能 | Claude に与えるもの | スコープ |
|------|-------------------|---------|
| Skills | 知識（タスク単位のプロンプト） | タスク |
| MCP | 手足（外部ツール操作） | プロジェクト |
| CLAUDE.md | ルール（自動読み込みの規約） | プロジェクト |
| Hooks | 自動化（アクション後のパイプライン） | プロジェクト |
| Plan Mode | 制約（読み取り専用にして計画に集中） | セッション |

### 次のアクション

1. `/init` で CLAUDE.md を生成し、プロジェクトのルールを書き出す
2. `.mcp.json` に Playwright MCP を追加し、ブラウザ操作を試す
3. `.claude/hooks/` に自動リントスクリプトを配置する

これらは難しい設定ではない。**それぞれ数分で導入でき、以降のすべてのセッションで効果を発揮する**。
