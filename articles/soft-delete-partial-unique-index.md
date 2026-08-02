---
title: "論理削除してるテーブルに、同じキーで登録し直せなくてハマった（Postgres）"
emoji: "🔑"
type: "tech"
topics: ["postgresql", "sql", "sqlalchemy", "database", "個人開発"]
published: true
---

「1つのアカウントが同じ外部プロジェクトを二重に連携できない」ようにしたくて、素直にユニーク制約を張ってた。

```sql
CREATE TABLE project_links (
  id          uuid PRIMARY KEY,
  account_id  uuid NOT NULL,
  project_key text NOT NULL,
  is_active   boolean NOT NULL DEFAULT true,
  created_at  timestamptz NOT NULL DEFAULT now(),
  UNIQUE (account_id, project_key)
);
```

連携の解除は、行を消さずに `is_active = false` にしてる。履歴を残したいし、消すと子データの外部キーが困るので。

で、これでハマった。いちど解除したプロジェクトを、同じアカウントがまた連携しようとすると、同じ `(account_id, project_key)` のINSERTが弾かれる。

```
ERROR: duplicate key value violates unique constraint "project_links_account_id_project_key_key"
```

行は残ってるんだから、DB的にはそりゃそうなる。頭では分かってても、解除したつもりのやつに再登録を止められるのは地味に困った。

## キーをいじる方向は両方ダメだった

最初に試したのが「ユニークキーに `is_active` を足す」。

```sql
UNIQUE (account_id, project_key, is_active)
```

`(A, X, true)` と `(A, X, false)` が共存できるから一瞬いけた気がした。けど2回目の解除で死ぬ。すでに `(A, X, false)` があるところにまた連携→解除すると、`(A, X, false)` が2つになって衝突する。

じゃあ `deleted_at`（アクティブならNULL）を足すか、と思ったらこっちは逆にザルだった。Postgresは既定でNULL同士を別物扱いするので、`deleted_at IS NULL` の行＝アクティブな行がいくつでも作れる。「アクティブは1個」が丸ごと効かない。本末転倒。

欲しいのは「アクティブな行の中だけで一意」。解除済みは数えたくない。それ、部分ユニークインデックスでそのまま書ける。

## 部分ユニークインデックスで済んだ

テーブルのUNIQUEは外して、こう張る。

```sql
CREATE UNIQUE INDEX uq_active_project_link
  ON project_links (account_id, project_key)
  WHERE is_active;
```

`WHERE is_active` があるので、インデックスが見てるのはアクティブな行だけ。解除済みは端から対象外だから、同じキーで新しいアクティブ行を足せる。それでいてアクティブが2個になるのはちゃんと弾く。やりたかったのはこれ。

SQLAlchemyだと制約じゃなくインデックスで書く。

```python
__table_args__ = (
    Index(
        "uq_active_project_link",
        "account_id", "project_key",
        unique=True,
        postgresql_where=text("is_active"),
    ),
)
```

## テストで見逃しがちなので実Postgresで見た

この部分インデックス、マイグレーションで張り忘れててもテストが素通りする。モデル定義には書いてあるのに実DBには無い、って状態でも、テストがSQLiteやモックだと気づけない。で本番だけ二重登録できる。

なので使い捨てのPostgresを立てて確認した。見たのはこの3つ。

- 同じキーでアクティブ2件 → 弾かれる
- 解除してから再登録 → 通る
- 再連携と再解除を何回か繰り返す → 解除済みが何個あっても衝突しない

あと同時に同じキーのINSERTが走る競合もあるので、アプリ側は `IntegrityError` を拾って「すでに連携済み」に変えといた。これでレースでも壊れない。

論理削除と一意制約は相性が悪いってよく言うけど、一意にしたいのはアクティブな行だけ、って割り切れば部分インデックスで普通に済む。物理削除にするか迷う前に、まずこれでいいと思う。

---

ちなみにこの `project_links`、Backlogのプロジェクト連携を管理してるテーブルで、Backlogのチケットから GitHub の Draft PR を作る Keros ってやつを個人で作ってて、その中で踏んだ話です。動くところ置いてあるので、気になる人はどうぞ（登録不要）。
https://keros.repocarta.jp/example
