---
title: "ブラウザのブックマーク管理に限界を感じて、単一HTMLでURL管理PWAを作った話"
emoji: "🔗"
type: "tech"
topics: ["PWA", "JavaScript", "個人開発", "WebCryptoAPI", "CloudflarePages"]
published: true
---

## はじめに

ブラウザのブックマーク、使いこなせているだろうか。

「後で読む」つもりで保存したURLが数百件に膨れ上がり、フォルダ分けしても結局探せない。タブを閉じるのが怖くて100個以上開きっぱなし。メモリは圧迫され、目的のタブも見つからない。

この問題を解決するために **URL Manager** というPWAアプリを作った。

https://url-manager-ase.pages.dev/

単一HTMLファイルで完結し、サーバー不要。暗号化・タグ・メモ・ブックマークレットによるワンクリック保存まで備えている。この記事では、ブラウザのブックマークとの違い、実装で直面した問題、そしてその解決策を紹介する。

![URL Manager - メイン画面](/images/url-manager-overview.png)
*ダークテーマでのメイン画面。フォルダツリーとURLカード一覧。*

## ブラウザのブックマークとの違い

まず「ブックマークがあるのになぜ専用ツールが必要なのか？」という疑問に答えたい。

| 機能 | ブラウザのブックマーク | URL Manager |
|------|----------------------|-------------|
| フォルダ管理 | あり | あり（ネスト対応） |
| タグ付け | なし | あり（複数タグ・フィルタ） |
| メモ | なし | Markdownメモ付き |
| 暗号化 | なし | AES-256-GCM暗号化 |
| 検索 | タイトルのみ | タイトル・URL・タグ横断 |
| 一括操作 | なし | 移動・削除・ピン留め |
| バックアップ | ブラウザ依存 | JSON形式でエクスポート |
| クロスブラウザ | ブラウザごとに独立 | JSON経由で移行可能 |

ブラウザのブックマークは「保存して終わり」になりがちだ。URL Managerでは**保存した後の管理**に重点を置いている。タグで横断的にフィルタし、メモにIDやパスワード、補足情報を残せる。暗号化すれば他人にデータを見られる心配もない。

:::message
例えば、開発中のサービスのステージング環境URL・本番URL・管理画面URLをまとめてフォルダ管理し、各URLのメモに認証情報を記録する、といった使い方ができる。ブラウザのブックマークでは不可能なワークフローだ。
:::

## 技術構成 — なぜ単一HTMLなのか

```
URL Manager/
├── index.html      # アプリ本体（CSS + JS 内蔵、約1900行）
├── manifest.json   # PWAマニフェスト
├── sw.js           # Service Worker
├── icon-192.png    # アプリアイコン
└── icon-512.png    # アプリアイコン
```

フレームワークもバンドラーも使わず、`index.html` 1ファイルにHTML・CSS・JavaScriptをすべて内蔵した。

**なぜこの構成にしたか:**

- デプロイが楽。ファイルをホスティングするだけ
- 依存関係ゼロで壊れない。npm installもビルドも不要
- Service Workerでキャッシュすればオフラインで動く
- 個人ツールに複雑なインフラは不要

データはすべて `localStorage` に保存する。サーバーサイドのコードは一切ない。

## 暗号化の設計 — Web Crypto API

パスワード管理やログイン情報をメモに残すケースを想定し、暗号化機能を実装した。

```javascript:index.html
async function deriveKey(password, salt) {
  const enc = new TextEncoder();
  const km = await crypto.subtle.importKey(
    'raw', enc.encode(password), 'PBKDF2', false, ['deriveKey']
  );
  return crypto.subtle.deriveKey(
    { name: 'PBKDF2', salt, iterations: 600000, hash: 'SHA-256' },
    km,
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt', 'decrypt']
  );
}
```

**PBKDF2で60万回ストレッチング**した上で **AES-256-GCM** で暗号化する。localStorageに保存されるデータ自体が暗号文になるため、ブラウザの開発者ツールからも中身を読み取れない。

![マスターパスワード入力画面](/images/url-manager-password.png)
*暗号化を有効にすると、起動時にパスワード入力が必要になる。*

### セッション管理の工夫

暗号化を有効にすると、ブックマークレットで新しいタブが開くたびにパスワード入力を求められる。これはかなり不便だった。

解決策として **セッション管理** を導入した。

```javascript:index.html
const SESSION_GAP = 180000; // 3分

function saveSession(pw) {
  localStorage.setItem(SESSION_KEY,
    JSON.stringify({ pw, ts: Date.now() }));
}
```

パスワード入力後、localStorageにタイムスタンプ付きで保存する。60秒ごとにハートビートで更新し、**3分間操作がなければ自動で失効**する。これにより、ブラウザを使っている間はパスワードを再入力する必要がなく、離席すれば自動ロックされる。

:::message alert
sessionStorageではなくlocalStorageを使っている理由がある。sessionStorageはタブごとに独立しているため、ブックマークレットが開く新しいタブとデータを共有できない。localStorageならタブをまたいでセッション情報を共有できる。
:::

## ブックマークレット — ワンクリック保存の裏側

「URLをコピーしてアプリに貼り付ける」のは手間がかかる。ブックマークレットを使えば、**閲覧中のサイトでワンクリックするだけ**でURL ManagerにタイトルとURLが自動入力される。

```javascript
javascript:void(window.open(
  'https://url-manager-ase.pages.dev/'
  + '?title=' + encodeURIComponent(document.title)
  + '&url=' + encodeURIComponent(location.href)
))
```

仕組みはシンプルで、`document.title` と `location.href` をURLパラメータに載せてURL Managerを開くだけだ。受け取り側では `URLSearchParams` でパラメータを取得し、追加モーダルにprefillする。

### クロスオリジンの壁 — タブが増え続ける問題

ここで大きな問題に直面した。**ブックマークレットをクリックするたびに新しいタブが開く**のだ。

最初の対策は `window.open()` の第2引数に名前を指定する方法だった。

```javascript
// 同じ名前を指定すれば既存タブを再利用するはず…
window.open(url, 'urlmgr')
```

しかし、これは動作しなかった。

**原因はChromeのクロスオリジンセキュリティポリシー。** ブックマークレットは実行元サイト（例: zenn.dev）のオリジンで動作する。`window.open()` で別オリジン（url-manager-ase.pages.dev）のウィンドウを開いても、クロスオリジンのため既存ウィンドウの参照が取得できず、名前付きウィンドウが機能しない。

次に試したのは **BroadcastChannel API** だ。

```javascript
// 既存タブにデータを送信して、新しいタブを閉じる作戦
const bc = new BroadcastChannel('url-manager');
bc.postMessage({ title, url });
```

同一オリジン間なら通信できるため、新しいタブから既存タブにデータを送れる。しかし、**既存タブをフォアグラウンドに持ってくる手段がない**。`window.focus()` はバックグラウンドタブからは呼べないというブラウザの制限がある。新しいタブは閉じられるが、ユーザーは既存タブを手動で探す必要があり、UXとしては悪化した。

### 解決策 — 保存後にタブを自動クローズ

最終的にたどり着いたのは「タブ再利用を諦め、**保存後にタブを自動で閉じる**」というアプローチだ。

```javascript:index.html
// 保存処理の中で
if (state.fromBookmarklet) {
  state.fromBookmarklet = false;
  showToast('保存しました');
  setTimeout(() => { window.close(); }, 500);
  return;
}
```

URLパラメータ経由で開かれた場合に `state.fromBookmarklet = true` をセットし、保存完了後に `window.close()` でタブを閉じる。ユーザーの体験はこうなる。

1. 閲覧中のサイトでブックマークレットをクリック
2. 新しいタブでURL Managerが開く（タイトル・URL入力済み）
3. フォルダやタグを選んで「保存」
4. **タブが自動で閉じて元のサイトに戻る**

タブが蓄積せず、元のサイトに自動で戻れるため、タブ再利用と同等のUXを実現できた。

:::message
`window.close()` は `window.open()` で開かれたタブでのみ動作する。ブックマークレットは `window.open()` を使っているため、この条件を満たしている。URL Managerを直接開いた場合は `fromBookmarklet` がfalseなので、タブは閉じない。
:::

## Service Worker — キャッシュ更新の落とし穴

PWA化のためにService Workerを導入した。キャッシュ戦略は **Stale-while-revalidate** を採用している。

```javascript:sw.js
self.addEventListener('fetch', e => {
  e.respondWith(
    caches.match(e.request).then(cached => {
      const fetchPromise = fetch(e.request).then(response => {
        if (response.ok) {
          const clone = response.clone();
          caches.open(CACHE_NAME)
            .then(cache => cache.put(e.request, clone));
        }
        return response;
      }).catch(() => cached);
      return cached || fetchPromise;
    })
  );
});
```

キャッシュがあればそれを即座に返し、バックグラウンドで最新版を取得して更新する。起動が高速で、次回アクセス時に最新版が反映される。

### キャッシュ更新で苦労した点

開発中、コードを更新してデプロイしても**ユーザーに反映されない**という問題が何度も発生した。

原因は `CACHE_NAME` を変更し忘れていたこと。Stale-while-revalidateは「まずキャッシュを返す」ため、Service Worker自体が更新されなければ古いキャッシュが返され続ける。

```diff:sw.js
- const CACHE_NAME = 'url-manager-v3';
+ const CACHE_NAME = 'url-manager-v4';
```

**`CACHE_NAME` のバージョンを上げる**ことで、新しいService Workerがインストールされ、`activate` イベントで古いキャッシュが削除される。デプロイのたびにバージョンを上げるのを忘れないようにする必要がある。

:::message alert
Service Workerのデバッグは難しい。「コードを変更したのに反映されない」場合、まずDevToolsの **Application → Service Workers → Unregister** を試すこと。それでも反映されない場合は、キャッシュバージョンの変更漏れを疑う。
:::

## PWAとブラウザの関係 — データ同期の注意点

PWAとしてインストールすると、アプリとブラウザは**別のウィンドウとして扱われる**。ブックマークレットは `window.open()` でブラウザ側のURL Managerを開くため、保存先はブラウザ側のlocalStorageになる。

ここで混乱しやすいポイントがある。

- ブラウザとPWAアプリは**同じlocalStorageを共有**している
- しかし、ブラウザで保存してもPWAアプリの**画面には即座に反映されない**
- アプリ側でリロードすると反映される

この挙動を知らないとユーザーは「保存したのに消えた？」と不安になる。そこで、**PWAインストール完了時に案内ダイアログ**を表示するようにした。

```javascript:index.html
window.addEventListener('appinstalled', () => {
  // インストール完了時にブックマークレットとデータ同期の
  // 説明ダイアログを表示
  if (!localStorage.getItem(PWA_WELCOME_KEY)) {
    pwaWelcomeModal.classList.add('active');
  }
});
```

`appinstalled` イベントと `display-mode: standalone` の検出を組み合わせることで、インストール直後とアプリ初回起動の両方で案内を出している。

## テーマ切替 — FOUCを防ぐ工夫

ライト/ダークテーマの切替はCSS変数で実装している。

![テーマ比較](/images/url-manager-theme-compare.png)
*ライトテーマ。CSS変数の切り替えだけで配色が変わる。*

注意すべきは **FOUC（Flash of Unstyled Content）** だ。SPAでよくある「一瞬白く光る」現象は、CSSの適用前にデフォルトテーマが描画されることで起きる。

対策として、`<head>` 内にインラインスクリプトを配置し、HTML解析中にテーマを即座に適用している。

```html:index.html
<script>
(function(){
  var t = localStorage.getItem('url-manager-theme');
  if (!t) t = window.matchMedia(
    '(prefers-color-scheme:light)').matches
    ? 'light' : 'dark';
  document.documentElement.setAttribute('data-theme', t);
})()
</script>
```

このスクリプトはレンダリングをブロックするため、CSSが適用される前にテーマが確定する。

## その他の機能

### タグ機能と日本語IME対応

URLにタグを付与して横断的に管理できる。サイドバーからタグでフィルタリングも可能。

![タグ機能](/images/url-manager-tags.png)
*カードにタグが表示され、サイドバーからフィルタできる。*

実装で注意したのは**日本語IMEとの相性**だ。Enterキーでタグを追加する仕様だが、IMEの変換確定時もEnterキーが発火する。`e.isComposing` をチェックすることで、変換中のEnterを無視している。

```javascript:index.html
formTags.addEventListener('keydown', e => {
  if (e.isComposing) return; // IME変換中は無視
  if (e.key === 'Enter' || e.key === ',') {
    e.preventDefault();
    addTag(formTags.value.trim());
  }
});
```

### URL重複チェック

`www` の有無、末尾スラッシュ、クエリパラメータの順序を正規化した上で比較する。ブックマークレットで追加する際も自動で重複チェックが走り、既に登録済みなら警告を表示する。

### 一括操作

複数のURLカードを選択し、フォルダ間の一括移動・一括削除・ピン留め切替ができる。

![一括操作](/images/url-manager-bulk.png)
*複数選択時にアクションバーが表示される。*

### Share Target API（Android向け）

`manifest.json` に `share_target` を追加することで、Androidの共有メニューからURL Managerを選択できる。

```json:manifest.json
{
  "share_target": {
    "action": "./index.html",
    "method": "GET",
    "params": {
      "title": "title",
      "text": "text",
      "url": "url"
    }
  }
}
```

:::message
Share Target APIはAndroid ChromeとChromeOSで動作する。macOS/iOSのChromeでは現時点で非対応のため、Mac環境ではブックマークレットが主な追加手段になる。
:::

## デプロイ — Cloudflare Pages

Cloudflare Pagesを利用している。選定理由は以下の通り。

- **無料枠で十分**（静的サイトなのでリクエスト制限に余裕がある）
- **privateリポジトリからのデプロイが無料**（GitHub Pagesは有料プランが必要）
- グローバルCDNで高速配信

デプロイコマンドは1行。

```bash
npx wrangler pages deploy . --project-name url-manager
```

:::message alert
wrangler CLI でのデプロイ時、プロジェクト名を間違えると新しいプロジェクトが作成され、別のURLにデプロイされてしまう。実際にこのミスで「デプロイしたのに反映されない」と悩んだ。プロジェクト名は `wrangler pages project list` で必ず確認すること。
:::

## まとめ

- ブラウザのブックマークでは難しい**タグ管理・メモ・暗号化**を備えたURL管理ツールを、**単一HTMLファイル**で構築した
- **外部ライブラリゼロ**。バニラJS + CSS変数 + Web Crypto APIで全機能を実装
- ブックマークレットの**クロスオリジン問題**は、名前付きウィンドウもBroadcastChannelも通用しない。**保存後のタブ自動クローズ**で解決した
- Service Workerの**キャッシュ更新漏れ**は要注意。`CACHE_NAME`のバージョンアップを忘れずに
- PWAアプリとブラウザの**localStorage共有**は直感的ではない。ユーザー案内を忘れずに

実際に使ってみたい方はこちらからどうぞ。

https://url-manager-ase.pages.dev/
