---
title: "Diesel ORM 動的なクエリ発行ハック"
emoji: "🕌"
type: "tech"
topics: ["Rust", "diesel", "SQL"]
published: true
---

# 背景・問題

Diesel ORMで動的にクエリを生成したいことがある。
特定の条件に当てはまるときのみ、`WHERE`を追加したいときはよくあるだろう。

そういった場合は`into_boxed`(https://docs.rs/diesel/latest/diesel/prelude/trait.QueryDsl.html#method.into_boxed) を使用すれば容易に解決できる。
例はdocs.rsの方に載っているのでそちらを参照。

この記事で紹介するのはそういった基本的な解決法ではなく、特定のユースケースに対応するための少しハッキーな方法だ。

このようなSQLをORMで実現したいときに、Dieselではなかなか実現しづらかったが、試行錯誤の結果、マクロも使わずコンパイルが問題なくできる方法にできた。
```article.sql
SELECT
*
FROM articles
WHERE
    articles.status = "published"
AND
    (
        articles.content LIKE "%スペースで区切られた文字列の一部A%"
        OR articles.content LIKE "%スペースで区切られた文字列の一部B%"
        ....
    )
```
このように、`条件A AND (条件B OR 条件C OR...) `の形になっており、条件Bなどが入っている`()`の中の条件の和が動的に決まることは稀にある。
今回の例だと、検索機能の実現のために、検索文字列をスペースで区切り、それぞれをORで繋げたい。

これ以外の条件(`articles.status \ "published"`)がなかったり、条件の数が動的に決まるのではなく固定されているのであれば、前述の`into_boxed`や、`or_filter`、`or`等の便利DSLを活用するだけで問題なく実装できる。
今回はそれに当てはまらないので、ややハッキーな(は言いすぎかもしれないが、素直にSQLを書くときには99%の人が書かないであろうクエリを生成する)コードになった。

# 解決法

以下のようにself-joinをして解決する。その際、aliasを貼って一つの独立したテーブルのように扱えばコンパイルエラーも回避できる。

```query.rs
let mut query = schema::articles::table.into_boxed().filter(schema::articles::status.eq("published"));
let sub_article = diesel::alias!(schema::articles as sub_articles);
let mut query_2 = sub_article.into_boxed();
for keyword in keywords {
    query_2 = query_2.or_filter(sub_article.field(schema::articles::content).like(keyword));
}
let articles = query
    .inner_join(
        sub_article.on(
            sub_article.field(schema::articles::id)
            .eq(schema::articles::id)
        )
    ).filter(exists(query_2))
    .load(connection);
```

