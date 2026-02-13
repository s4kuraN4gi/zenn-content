# Skills プロジェクト

Claude Code の拡張機能（Skills, MCP, CLAUDE.md, Hooks）を実際に試しながら、Zenn記事の素材を蓄積するプロジェクト。

記事全体の構成・計画は @article-concept.md を参照。

## プロジェクト構成

- `article-log.md` — 各Levelの作業ログ・記事素材
- `articles/` — Zenn記事のドラフト（Zennフォーマット準拠）
- `article-concept.md` — 記事全体の構成・計画
- `level1-1/` — Level 1-1 の成果物（PixelMind LP）
- `level1-2/` — Level 1-2 の成果物（Next.jsサンプル + dep-visualizer）

## 記事の執筆ルール

- Zenn記事は `articles/` ディレクトリに配置する
- frontmatter（title, emoji, type, topics, published）を必ず含める
- `published: false` で保存し、手動で公開する
- スクリーンショットが必要な箇所は `<!-- TODO: -->` コメントで残す

## 作業ログのルール

- 新しい作業の知見は `article-log.md` に追記する
- セクション見出しは `## Level X-Y: タイトル` の形式
- 「やったこと」「学び」「記事に書くポイント」「スクショ撮影ポイント」を記録する

## コマンド

- 記事プレビュー: `npx zenn preview`
- 新規記事作成: `npx zenn new:article --slug スラッグ名`
