---
title: "ChatGPT Plus + Playwright で月コスト0円のサムネ画像自動生成 ── ブラウザ自動化と認証突破の実装ノウハウ"
emoji: "🎨"
type: "tech"
topics: ["chatgpt", "playwright", "python", "ai", "automation"]
published: true
---

## はじめに ── なぜ「ブラウザ経由」なのか

ChatGPT Plus に月額 $20 払っているのに、画像生成 API（gpt-image-1）を別途使うと **$0.04〜$0.17/枚** 取られる ── この二重課金を許せず、Plus サブスクの中で完結する自動化パイプラインを作りました。本稿では、**Playwright + Brave で ChatGPT を自走させ、生成画像を引き抜いてローカル保存する**まで、3 日かけて踏んだ落とし穴をすべて共有します。

結論から言うと、この構成で **1 記事あたり cover + inline 4 枚 = 5 画像を約 5 分で生成、月コスト 0 円**で運用できています。同様のことをやろうとしている方の参考になれば。

## アーキテクチャ全体像

```
[記事タイトル + H2 見出し]
    ↓
Gemma3 (ローカル LLM)  ← 日本語ビジュアルプロンプト生成
    ↓
[「○○のサムネ作って。Ghibli 風で」プロンプト 5 本]
    ↓
Playwright + Brave (永続プロファイル直結)
    ↓
ChatGPT (Plus サブスク経由) ── 1 チャットで連続 5 生成
    ↓
[backend-api/estuary/content URL 5 本]
    ↓
page.evaluate(fetch) ← Cookie 認証付き DL
    ↓
ローカル PNG 5 枚
```

`Brave + 永続プロファイル直結` がポイントで、同じプロファイルに ChatGPT Plus がログイン済みのため、毎回ログインせずに済みます。

## 落とし穴 1: Cookie の DPAPI 暗号化

最初に試したのは「Brave のプロファイルから Cookie ファイルを別ディレクトリにコピーして、Playwright にそれを user_data_dir として渡す」方式。これが **動きません**。Chromium 系のブラウザは、Windows なら DPAPI、macOS なら Keychain で Cookie を暗号化しています。Cookie ファイルだけ複写しても、復号鍵が一致せず ChatGPT 側に「ゲストユーザー」として認識されてしまう。

```python
# ✗ これは動かない
shutil.copy(brave_default / "Network/Cookies", isolated_profile / "Network/Cookies")
```

正解は **Brave のプロファイルそのものを user_data_dir として直接使う**こと。Brave が起動中だとプロファイルがロックされるので、`taskkill /F /IM brave.exe` で完全停止してから Playwright を起動する必要があります。

```python
ctx = p.chromium.launch_persistent_context(
    user_data_dir=str(BRAVE_USER_DATA),  # 実プロファイル直結
    executable_path=BRAVE_PATH,
    headless=False,
    args=["--disable-blink-features=AutomationControlled"],
)
```

## 落とし穴 2: 共有 URL は読み取り専用

ChatGPT には `https://chatgpt.com/share/<id>` という共有 URL があり、画像生成に使い回せそうに見えます。しかし共有ビューは **読み取り専用**で、composer に文字を入力しても送信ボタンが反応しない。

検証中に「Continue this conversation」ボタンが表示される場合もあると気づきましたが、UI バージョンによって出る・出ないが揺れる。実装としては **「共有 URL に飛んだ後、composer が visible でなければ chatgpt.com/ にフォールバック」** という安全網を入れて回避しました。

```python
if "/share/" in self._page.url and not composer.is_visible():
    self._page.goto("https://chatgpt.com/")  # 直近のチャットへ
```

## 落とし穴 3: 画像 URL は認証が必要

ChatGPT の生成画像は `https://chatgpt.com/backend-api/estuary/content?id=file_xxx` という URL で配信されます。これに **plain な `requests.get` で叩くと 403 Forbidden**。Cookie 認証が前提なので、Playwright のページコンテキストから fetch する必要があります。

```python
b64 = self._page.evaluate("""async (url) => {
    const r = await fetch(url, { credentials: 'include' });
    if (!r.ok) throw new Error('HTTP ' + r.status);
    const buf = await r.arrayBuffer();
    let binary = '';
    const bytes = new Uint8Array(buf);
    for (let i = 0; i < bytes.byteLength; i++) {
        binary += String.fromCharCode(bytes[i]);
    }
    return btoa(binary);
}""", img_url)
data = base64.b64decode(b64)
dest.write_bytes(data)
```

base64 でやり取りしているのは、Playwright の `evaluate` が ArrayBuffer を JSON シリアライズできないためです。

## 落とし穴 4: Batch で同じ画像が 5 枚返る

5 連続で生成するとき、`_wait_for_image` が DOM をポーリングして「最新の `<img>` を返す」実装にしていました。これだと **直前の画像がまだ DOM に残っているので、5 枚すべて 1 枚目の URL になる**バグが発生。

修正は単純で、**返した URL を skip set に追加し、ポーリングはそれ以外の新しい URL を待つ**仕様に変更:

```python
seen_urls: set[str] = set()
for i in range(5):
    self._send_prompt(prompts[i])
    img_url = self._wait_for_image(skip_urls=seen_urls)
    seen_urls.add(img_url)
    self._download_via_browser(img_url, dest_paths[i])
```

## 落とし穴 5: チャット削除に Bearer Token が必要

batch 1 回ごとに新しいチャットがサイドバーに残るので、後始末で `/backend-api/conversation/<id>` に `PATCH {is_visible: false}` を投げる実装にしました。**Cookie だけだと 401**。`/api/auth/session` を先に叩いて access token を取得し、`Authorization: Bearer <token>` を付けて初めて通ります。

```python
const sess = await fetch('/api/auth/session', { credentials: 'include' });
const { accessToken } = await sess.json();
await fetch('/backend-api/conversation/' + chatId, {
    method: 'PATCH',
    headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer ' + accessToken,
    },
    body: JSON.stringify({is_visible: false}),
    credentials: 'include',
});
```

## ToS とコストの整理

ChatGPT の利用規約は「自動化されたプログラム的アクセス」を一般禁止しています。ただし **ChatGPT Plus アカウント保持者が自分のアカウントを自動化する**ケースは、現実問題ほぼ取り締まりの対象外。本実装は捨て垢ではなくメインアカウントで 3 日連続稼働、現時点で警告 0 件です。とはいえ規約上はグレーゾーンなので、**業務利用や量産には API 経由を推奨**します。

コスト面では、本実装で生成した画像は累計 50 枚超（テスト含む）。同等の画質を gpt-image-1 で生成すると約 **$3〜$8.5** の課金。ChatGPT Plus サブスクが既にあるなら、確実に元が取れる構成です。

## まとめ ── ローカル AI と SaaS の境界線を縫う

ローカル LLM（Gemma3）でプロンプトの「日本語下書き」を作り、SaaS（ChatGPT Plus）で「絵」を生成し、結果はローカルに保存する。**それぞれの強みが効くフェーズだけ呼び出す**設計にしたら、コストと品質のバランスが噛み合いました。

似たことを試してみたい方は、Brave 完全停止 → 永続プロファイル直結 → fetch でダウンロード、の 3 点を押さえれば 1 日で動きます。落とし穴をひと通り踏んだ筆者の実装は GitHub で公開していますので、参考にしてください。

## 出典

- ChatGPT 利用規約: https://openai.com/policies/terms-of-use/
- Playwright Python 公式ドキュメント: https://playwright.dev/python/
- Brave Browser ダウンロード: https://brave.com/
- gpt-image-1 API 価格: https://openai.com/api/pricing/
