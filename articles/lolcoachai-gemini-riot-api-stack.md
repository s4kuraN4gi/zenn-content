---
title: "AI動画分析を1円以下で実現 — Next.js × Gemini × Riot APIで作るLoLコーチングアプリ"
emoji: "🎮"
type: "tech"
topics: ["nextjs", "gemini", "react", "個人開発"]
published: true
---

## はじめに

League of Legendsの試合を**AIがコーチング**してくれるアプリを個人開発した。動画をアップロードすると、Gemini 2.0 Flashがフレームを解析し、「このトレードは不利だった」「ワードを置くべきだった」と具体的にアドバイスしてくれる。

動画分析1回あたりのAIコストは**1円以下**。この記事では、どうやって低コストを実現しつつ実用的なコーチング品質を出しているかを、技術選定の観点から解説する。

**スタック**: Next.js 16 / React 19 / Supabase / Gemini 2.0 Flash / Riot API / Stripe
**サイト**: [lolcoachai.com](https://lolcoachai.com)

## アーキテクチャ全体像

```
┌──────────────┐     ┌───────────────┐     ┌──────────────┐
│   Client     │────▶│  Next.js 16   │────▶│  Supabase    │
│  (React 19)  │     │  App Router   │     │  PostgreSQL  │
└──────────────┘     └───────┬───────┘     └──────────────┘
                             │
                     ┌───────┴───────┐
                     ▼               ▼
              ┌────────────┐  ┌────────────┐
              │  Riot API  │  │ Gemini AI  │
              │ (試合データ)│  │(分析/チャット)│
              └────────────┘  └────────────┘
```

### なぜこの構成にしたか

- **Next.js 16** (App Router + Turbopack): SSRでダッシュボードを高速描画しつつ、Server Actionsで外部API呼び出しをサーバー側に閉じ込める。クライアントにAPIキーが漏れない
- **Supabase**: 認証・DB・Row Level Security（RLS）・RPC関数を一括で提供。個人開発でバックエンドを別途構築する必要がなくなる
- **Riot API**: 試合データ・サモナー情報・ランク情報の取得。公式APIなので信頼性が高い
- **Gemini 2.0 Flash**: 画像認識 + テキスト生成を1つのAPIで実現。GPT-4 Visionと比較してコストが桁違いに低い
- **SWR**: クライアント側のキャッシュ + バックグラウンド再検証。ダッシュボードの体感速度が劇的に改善した

## 動画分析パイプライン — 低コストの鍵

このアプリの核心機能。ユーザーがLoLのリプレイ動画をアップロードすると、AIが具体的な改善点を返す。

### Step 1: Canvas APIでフレーム抽出

ブラウザ上で `<video>` → `<canvas>` → WebP画像に変換する。サーバーに動画全体を送る必要がない。

```typescript:フレーム抽出の概要
// 動画をリサイズし、WebPで圧縮（容量を大幅に削減）
const scale = Math.min(1, MAX_DIMENSION / Math.max(video.videoWidth, video.videoHeight));
canvas.width = video.videoWidth * scale;
canvas.height = video.videoHeight * scale;

ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

// WebP → JPEG フォールバック（Safari対応）
const blob = await new Promise<Blob>((resolve) => {
    canvas.toBlob((b) => resolve(b!), 'image/webp', 0.7);
});
```

:::message
**コスト削減のポイント**: 動画全体ではなく、分析対象のシーンから少数のキーフレームだけを抽出する。フレーム数を最小限に抑えることで、Geminiに送るトークン数を大幅に削減している。
:::

### Step 2: Gemini 2.0 Flashで分析

抽出したフレームをGemini APIに送信。試合データと組み合わせたプロンプトを構築し、AIがゲーム状態を読み取って構造化されたコーチングを返す。

AIの出力例（実際の分析結果から抜粋）:

```
【トレード分析】
結果: 不利トレード
改善点: 視界のないブッシュ付近で前に出すぎた
推奨: ワードで視界を確保してからポジションを取る

【ポジショニング改善】
現状: ブッシュ近くに不用意に立つ
理想: ワードを置くか、安全な距離を保つ
練習: カスタムゲームでフック系チャンピオンとのレーン戦を練習する
```

### Step 3: モデルフォールバック

Gemini APIの可用性を担保するため、複数モデルを順番に試行する仕組みを入れている。最新モデルが利用できない場合、自動的に安定版にフォールバックする。

:::message
**コスト実績**: Gemini 2.0 Flashは画像認識タスクにおいてGPT-4 Visionと比較して大幅に低コスト。個人開発の動画分析機能を現実的な価格で運用できる最大の要因。
:::

## Riot API連携 — レート制限への対応

Riot APIにはレート制限がある。これを超えると429ステータスが返る。

### Retry-Afterヘッダーを尊重する

```typescript:レート制限対応の基本パターン
if (res.status === 429 && retries > 0) {
    const retryAfterSec = parseInt(
        res.headers.get("Retry-After") || "1"
    );
    await delay((retryAfterSec + 1) * 1000); // +1秒のバッファ
    return fetchMatchDetail(matchId, retries - 1);
}
```

試合データは不変（一度終わった試合のデータは変わらない）なので、積極的にキャッシュしている。並列取得時はチャンクに分けてバックオフを挟むことで、レート制限に引っかかるリスクを最小化した。

## DDragonだけでは足りない

ダメージ計算機能を実装する際、Riotが提供するDDragon（Data Dragon）のスペルデータが**全てゼロ**であることに気づいた。

```
// DDragonのスペルデータ
{
    "effect": [],    // ← 常に空
    "vars": [],      // ← 常に空
}
```

:::message alert
DDragonの `effect[]` と `vars[]` は全てのチャンピオンで空配列を返す。スペルのダメージ値や倍率を取得するには、コミュニティが運営する別のデータソースを利用する必要がある。
:::

コミュニティが提供するデータソースにはスペルの基礎ダメージ、スケーリング係数、ランク別の数値が正確に格納されている。これをパースすることで、「Qランク3でAP100の時のダメージ」を正確に計算できるようになった。

LoL関連のツールを開発する際、DDragonだけで完結すると思い込むのは危険。この罠にハマると数日無駄にする。

## Supabaseの設計 — RPCでレースコンディションを防ぐ

週間分析回数の制限を「チェック → インクリメント」の2ステップで実装すると、同時リクエストでリミットを超える可能性がある。

PostgreSQL RPCで**アトミックな操作**にした:

```sql:アトミックなカウンター操作
-- 1回のUPDATEでチェックとインクリメントを同時に実行
UPDATE profiles
SET weekly_analysis_count = weekly_analysis_count + 1
WHERE id = p_user_id
  AND weekly_analysis_count < p_limit
RETURNING weekly_analysis_count;

-- リミット到達済みなら -1 を返す
IF NOT FOUND THEN RETURN -1; END IF;
```

この「アトミックチェック＆インクリメント」パターンはSaaSの利用制限全般に応用できる。チャット回数制限やWebhookの冪等性チェックにも同じアプローチを採用している。

## AIコーチのペルソナ設計

チャット機能ではAIコーチが応答する。設計で最も重要にしたのは**回答の長さ制限**。

AIの回答が長すぎると読む気が失せる。システムプロンプトで回答長を制限し、箇条書きを推奨することで、チャットの読みやすさを確保した。

もう一つ工夫したのは**教育的アプローチ**。すぐに正解を教えるのではなく、ユーザー自身に考えさせるクイズ形式の問いかけを入れている。

```
「例えばこの状況でバロンが湧いていたら、君ならどう動く？」
```

知識を「教わる」のではなく「考えて身につける」方が定着率が高い。コーチングアプリとしての本質的な価値はここにあると考えている。

## デプロイ構成

```
AWS Lightsail（月数千円）
├── Docker Compose
│   ├── lolcoachai-web (Next.js 16)
│   └── nginx-proxy (SSL終端)
└── Supabase（マネージド PostgreSQL）
```

Docker Composeでアプリとリバースプロキシをまとめ、Nginxで動的DNS解決を設定することで、再デプロイ時の502エラーを防止している。

:::details なぜ動的DNS解決が必要か
Nginxは `proxy_pass` のDNS解決を起動時に1回だけ行いキャッシュする。Dockerコンテナを再作成するとIPが変わるため、Nginxの `resolver` ディレクティブでDockerの内部DNSを参照し、短い間隔で再解決させる必要がある。
:::

個人開発でもDocker + Lightsailなら、低コストで本番環境を運用できる。

## まとめ

- **動画分析 1円以下/回**: Canvas APIでフレーム抽出 → WebP圧縮 → Gemini 2.0 Flashで分析
- **Riot APIレート制限対策**: Retry-Afterヘッダー尊重 + キャッシュ + チャンク並列取得
- **DDragonの罠**: スペルデータは全てゼロ。コミュニティデータソースを併用する
- **RPCでアトミック操作**: レースコンディションをDB側で防止
- **低コスト運用**: Lightsail + Docker Compose + Supabase

個人開発でもAIを活用すれば、プロコーチに近い分析品質を低コストで提供できる。ゲームデータ × AI分析の組み合わせは、LoLに限らず可能性が大きい分野だと感じている。

**サイト**: [lolcoachai.com](https://lolcoachai.com)
