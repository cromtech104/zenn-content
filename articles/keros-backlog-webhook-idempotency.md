---
title: "Backlog Webhookは二重に来る前提で受ける"
emoji: "🔁"
type: "tech"
topics: ["backlog", "webhook", "設計", "fastapi", "postgresql"]
published: true
---

Backlogのwebhookを受けて何か処理を走らせる構成を作ると、最初に困るのが「同じ通知が二回届く」ことだ。課題を一回更新しただけのつもりが、受け側のログには同じissueが二回流れてくる。処理が軽いうちは気づかないが、webhookをトリガーに重い処理（外部API呼び出し、PR作成、課金が絡む処理など）を走らせていると、二重処理がそのまま実害になる。

この記事は、Backlog webhookを冪等に受けるための設計をまとめたもの。特定のフレームワークやインフラに依存しない話なので、Lambdaでも常駐サーバーでも考え方は同じだ。GitHub側のwebhookで同じことをやったときの話は[別の記事](/articles/github-webhook-claude-docs)に書いたけれど、Backlogだと事情が少し違う。

## なぜ二重に届くのか

webhookは「最低一回は届く（at-least-once）」前提の配送方式が多い。送信側はレスポンスを受け取れなかったら再送する。つまり受け側のレスポンスがネットワークの都合で送信側に届かなければ、処理自体は成功していても再送が来る。

Backlog固有の事情もある。

- 課題の更新は、複数フィールドの変更が別々のアクティビティとして届くことがある。
- 一覧操作や一括更新で、短時間に同じ課題の通知が連続することがある。
- 受け側が遅いと、送信側のタイムアウト後に再送される。

なので「同じ通知は来ない」という前提でコードを書くと、いつか必ず壊れる。来る前提で、来ても困らないように作っておくのが現実的だ。

## 入口と実処理を分けて考える

受け側を「入口（HTTPを受ける層）」と「実処理（重い処理を走らせる層）」に分けると整理しやすい。

冪等性は基本的に実処理の層で担保する。入口で重複を弾こうとすると、メモリ上のキャッシュや短命なロックに頼ることになって、プロセスが複数あるときや再起動したときに穴が空く。永続化された一意制約で弾くほうが確実だ。

入口の層がやるのは次のあたりに絞る。

- 認証（誰からの通知か、宛先は正しいか）
- ペイロードの最小バリデーション（必要なフィールドがあるか）
- 対象外の早期リターン（後述）
- 実処理へ渡す（同期でも非同期でもよい）

## 入口で早めに捨てる

冪等性の前に、そもそも処理する必要がない通知を早めに捨てておくと後段が楽になる。

Backlogのペイロードには`type`（アクティビティ種別）が入っている。課題の作成・更新だけ処理したいなら、それ以外は受け取った時点で`ignored`として返してしまう。課題種別（バグ・要望など）で対象を絞るのも、同じくここでやる。

```python
# 対象のアクティビティ種別だけ通す
if activity_type not in (ISSUE_CREATED, ISSUE_UPDATED):
    return {"status": "ignored", "reason": f"activity_type={activity_type}"}

# 対象の課題種別だけ通す
if issue_type_name not in SUPPORTED_ISSUE_TYPES:
    return {"status": "ignored", "reason": f"issueType={issue_type_name!r}"}
```

ここで弾いたものはそもそも実処理に渡らないので、冪等性の対象にもならない。重複対策のコストを下げる意味でも、入口のフィルタは効く。

## 冪等キーを決める

実処理の冪等性は「冪等キー（idempotency key）」で担保する。同じ仕事を表すキーを決めて、それを永続化層の一意制約に乗せる。

キーの作り方は、何を「同じ仕事」と見なすかで決まる。Backlogの課題を起点にするなら、最低限「どのプロジェクトの、どの課題か」が要る。課題キー（`PROJ-123`のような文字列）はリネームやプロジェクト移動で変わりうるので、変わらない内部IDを使うほうが安全だ。

再処理（リトライ）を別の仕事として扱いたい場合は、リトライ回数もキーに含める。これをやらないと「失敗したから手動で再実行」が一意制約に弾かれて動かなくなる。

```python
def build_idempotency_key(*, project_id, issue_id, retry_count):
    return f"{project_id}:{issue_id}:{retry_count}"
```

文字列連結でもJSONのハッシュでもよいが、次の点を満たしていればいい。

- 同じ仕事なら必ず同じキーになる（決定的）
- 違う仕事なら衝突しない
- 意図的な再実行を別キーにできる余地がある

## DBの一意制約で弾く

冪等キーが決まったら、それを保存するテーブルに一意制約を張る。そして「まずINSERTを試みて、一意制約違反なら重複と判断する」順序にする。

```sql
CREATE TABLE ticket_processing_runs (
    id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      uuid NOT NULL,
    issue_id        text NOT NULL,
    idempotency_key text NOT NULL UNIQUE,
    status          text NOT NULL,
    created_at      timestamptz NOT NULL DEFAULT now()
);
```

```python
run = TicketProcessingRun(
    project_id=project_id,
    issue_id=issue_id,
    idempotency_key=build_idempotency_key(
        project_id=project_id, issue_id=issue_id, retry_count=0
    ),
    status="processing",
)
db.add(run)
try:
    db.commit()
except IntegrityError:
    db.rollback()
    # 既に同じキーのレコードがある = 二重通知。処理せず終わる
    existing = get_by_idempotency_key(db, run.idempotency_key)
    raise DuplicateProcessingRun(existing)
```

ポイントは「先に確認してからINSERTする」のではなく「INSERTしてみて、ダメだったら重複とみなす」順序にすること。

確認してからINSERTする書き方（selectしてなければinsert）は、二つの通知がほぼ同時に来たときに、両方が「まだ無い」と判断して両方insertに進む隙間がある。一意制約に任せれば、その競合はDBが直列化してくれて、片方だけが成功する。アプリ側でロックを持つより、この一行の制約のほうが堅い。

PostgreSQLなら`INSERT ... ON CONFLICT DO NOTHING`でも書けるが、SQLAlchemyなどのORMをまたいで使うなら、例外を捕まえる書き方のほうが移植しやすい。どちらでも本質は同じで、「一意制約に二重処理の判定を委ねる」ことが肝になる。

## 処理中のまま固着したレコードに備える

一意制約があれば「完全に同じキー」の二重処理は防げる。ただし運用ではもう少し考えることがある。

ひとつは、処理中（`status=processing`）のまま落ちたレコードの扱いだ。プロセスが途中で死ぬと、レコードは`processing`のまま残り、後続の再処理が「まだ動いている」と誤認しうる。`started_at`から一定時間（たとえば数分）を超えた`processing`は「固着」とみなして、リトライ対象に回す仕組みを入れておくと、手動復旧の手間が減る。

もうひとつは、同じ課題を短時間に何度も更新したときの扱い。これは二重通知（同じ仕事）とは別で、「別の仕事が連続している」状態だ。最新の更新だけ処理したいのか、すべて処理したいのかは要件次第なので、冪等キーの設計（更新時刻を含めるかどうか）でそこを表現し分ける。

## 入口の認証も忘れずに

冪等性とは別軸だが、webhookの入口では認証も要る。BacklogのwebhookはGitHubのように署名ヘッダーを付けてくれるわけではなく、リクエストヘッダを自由に足せない。なので共通の固定シークレットを使うか、宛先ごとに固有のトークンをURLに埋める方式が現実的になる。

固有トークン方式なら、漏れたときの影響範囲がその宛先だけに閉じて、宛先側でローテーションもできる。照合は`secrets.compare_digest`のような定数時間比較を使い、トークンやシークレットの値はログにもエラーレスポンスにも出さない。存在しない宛先と無効なトークンは区別せず同じエラーにして、宛先の存在を漏らさないようにする。このあたりも冪等性と同じく、「来る前提・破られる前提」で固めておく領域だ。

## まとめ

Backlog webhookを冪等に受けるための要点はこのあたり。

- webhookは最低一回配送。二重に来る前提で作る。
- 入口で対象外を早めに捨て、冪等性の対象を減らす。
- 「同じ仕事」を表す決定的な冪等キーを決める。リトライは別キーにできる余地を残す。
- 確認してからINSERTではなく、INSERTして一意制約違反を重複とみなす。競合はDBに直列化させる。
- 処理中のまま固着したレコードのリトライ経路を用意する。

自分は、Backlogの課題を起点にGitHubのDraft PRを作る[Keros](https://keros.repocarta.jp)というサービスを作っていて、その受け口でこの設計を使っている。webhookで重い処理を駆動するときの二重処理は地味に効いてくるので、最初から一意制約に寄せておくのがおすすめ。どこまでをAIに任せて、どこから人がマージするかの線引きは[別の記事](/articles/ai-fix-review-before-merge)に書いた。
