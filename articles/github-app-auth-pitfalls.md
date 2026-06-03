---
title: "GitHub Appの認証実装でハマった5つのこと"
emoji: "🔐"
type: "tech"
topics: ["GitHubApp", "GitHub", "Python", "認証", "個人開発"]
published: true
---

## この記事の答え

GitHub Appを実装する際の主なハマりポイントは以下の5つです。

1. App JWTとInstallation Access Tokenは別物で、2段階の認証が必要
2. Installation Access Tokenには1時間のTTLがあり、キャッシュ管理が必要
3. JWTのiatは「60秒前」で発行しないとクロックスキューで弾かれる
4. Webhookのイベントは1つのエンドポイントで全種類を受け取り、自前でルーティングする
5. ボット自身のpushでWebhookが無限ループする

GitHubのドキュメントはフローごとに分散していて、全体像を把握するまでに時間がかかりました。自分が詰まったポイントをまとめます。

---

## GitHub Appの認証フロー全体像

まず整理です。GitHub Appには「App自体の認証」と「リポジトリへのアクセス」の2つが必要で、それぞれ別のトークンを使います。

```
RSA秘密鍵
  ↓ 署名
App JWT（有効10分）
  ↓ POST /app/installations/{id}/access_tokens
Installation Access Token（有効1時間）
  ↓ 使う
GitHub API（リポジトリ操作）
```

OAuth Appに慣れていると、「トークン1つで動く」と思いがちですが、GitHub Appは2段階です。

---

## ハマりポイント1: JWTのiatを「60秒前」にする

App JWTを生成するとき、`iat`（issued at）を現在時刻にするとGitHubから弾かれることがあります。

```python
# ❌ これで弾かれることがある
now = int(time.time())
jwt_payload = {
    "iat": now,
    "exp": now + 540,
    "iss": app_id,
}

# ✅ iatを60秒前にする
now = int(time.time())
jwt_payload = {
    "iat": now - 60,   # クロックスキュー対策
    "exp": now + 540,  # 有効9分（上限10分）
    "iss": app_id,
}
```

GitHubのドキュメントに「クロックスキュー対策で60秒前を推奨」と書いてありますが、最初は見落としていました。サーバーとGitHubの時計がわずかにずれていると`iat`が未来の時刻になり弾かれます。

---

## ハマりポイント2: Installation Access Tokenのキャッシュ管理

Installation Access Tokenは1時間で失効します。毎回APIを叩くとレート制限に引っかかるため、キャッシュが必要です。

ポイントは「失効5分前に再取得する」ことです。ギリギリまで使うと、取得直後に失効するケースがあります。

```python
_installation_token_cache: dict[int, tuple[str, float]] = {}

def _get_installation_token(installation_id: int) -> str:
    # キャッシュチェック（失効5分前まで利用）
    cached = _installation_token_cache.get(installation_id)
    if cached and time.monotonic() < cached[1] - 300:
        return cached[0]

    # App JWT生成
    app_jwt = _generate_app_jwt()

    # Installation Access Token取得
    resp = requests.post(
        f"https://api.github.com/app/installations/{installation_id}/access_tokens",
        headers={
            "Authorization": f"Bearer {app_jwt}",
            "Accept": "application/vnd.github+json",
            "X-GitHub-Api-Version": "2022-11-28",
        },
        timeout=10,
    )
    resp.raise_for_status()

    data = resp.json()
    token = data["token"]

    # expires_atはISO8601形式 "2024-01-01T00:00:00Z"
    expires_at_str = data.get("expires_at", "")
    try:
        dt = datetime.fromisoformat(expires_at_str.replace("Z", "+00:00"))
        expires_monotonic = time.monotonic() + (
            dt - datetime.now(timezone.utc)
        ).total_seconds()
    except Exception:
        expires_monotonic = time.monotonic() + 3600  # フォールバック

    _installation_token_cache[installation_id] = (token, expires_monotonic)
    return token
```

`expires_at`はAPIレスポンスに含まれているので、計算で求めるより直接使う方が正確です。

---

## ハマりポイント3: Webhookイベントの全種類を1エンドポイントで受ける

GitHub Appのウェブフックは、インストール・push・PR・marketplace購入など、あらゆるイベントが同一のエンドポイントに届きます。`X-GitHub-Event`ヘッダーでイベント種別を判定し、自前でルーティングします。

```python
@app.post("/webhooks/github")
async def github_webhook(
    request: Request,
    x_github_event: str = Header(None),
    x_hub_signature_256: str = Header(None),
):
    payload = await request.body()

    # まず署名検証
    if not x_hub_signature_256:
        raise HTTPException(status_code=401, detail="Missing signature")
    if not verify_github_signature(payload, x_hub_signature_256, secret):
        raise HTTPException(status_code=401, detail="Invalid signature")

    body = json.loads(payload)

    # イベント種別でルーティング
    if x_github_event == "ping":
        return {"status": "ok"}

    if x_github_event == "installation":
        # インストール・アンインストール処理
        ...

    if x_github_event == "push":
        # pushイベント処理
        ...

    if x_github_event == "pull_request":
        # PRイベント処理
        ...

    return {"message": "ignored"}
```

`ping`イベントはApp登録直後にGitHubから疎通確認として送られます。署名検証をスキップして即返さないと、App設定が保存されません。

---

## ハマりポイント4: Webhook署名検証の実装

`X-Hub-Signature-256`はHMAC-SHA256です。タイミング攻撃対策として、`hmac.compare_digest`を使います。

```python
import hashlib
import hmac

def verify_github_signature(
    payload: bytes,
    signature_header: str,
    secret: str,
) -> bool:
    if not signature_header.startswith("sha256="):
        return False

    expected = hmac.new(
        secret.encode("utf-8"),
        payload,
        hashlib.sha256,
    ).hexdigest()

    actual = signature_header[len("sha256="):]

    # タイミング攻撃対策
    return hmac.compare_digest(expected, actual)
```

`==`で比較すると文字列の一致箇所が増えるにつれて処理時間が変わり、タイミング攻撃の余地が生まれます。`compare_digest`は常に固定時間で比較します。

---

## ハマりポイント5: ボット自身のpushによる無限ループ

GitHub Appがドキュメント更新のコミットをpushすると、そのpushイベントが再びWebhookとして届きます。そのままドキュメント生成を走らせると、永遠にループします。

pusherのnameで判定して弾きます。

```python
if x_github_event == "push":
    pusher_name = body.get("pusher", {}).get("name", "")

    # ボット自身のpushはスキップ
    if pusher_name == "github-actions[bot]" or pusher_name.endswith("[bot]"):
        return {"message": "ignored"}

    # 以降のドキュメント生成処理へ
    ...
```

GitHub Actionsを使ってコミットする場合、pusherは`github-actions[bot]`になります。自前のbotアカウントを使う場合は、そのアカウント名を条件に追加してください。

---

## まとめ

| ハマりポイント | 対処 |
|--------------|------|
| App JWTが弾かれる | `iat`を60秒前に設定 |
| トークンが頻繁に切れる | 失効5分前にキャッシュ再取得 |
| イベントが届かない | `ping`イベントを即返す |
| 署名検証が通らない | `hmac.compare_digest`を使う |
| 無限ループ | pusher名でbotを弾く |

GitHub Appは設定できると強力ですが、認証フローの全体像を把握するまでが一番大変でした。公式ドキュメントの[Creating a GitHub App](https://docs.github.com/en/apps/creating-github-apps)と[Authenticating as a GitHub App installation](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation)を行き来しながら実装することをおすすめします。
