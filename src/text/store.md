# データの更新

このページではデータの更新として、保存と削除を行ってみます。

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/7...8?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード（保存）</span>
</a>

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/8...9?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード（削除）</span>
</a>

## パスパラメータの取得

データの削除については、パスに含まれるIDから対象を特定して削除するようにしているため、パスパラメータを取得する必要があります。

パスパラメータを取り扱うには、 `axum::extract::Path` を使用します。

[https://docs.rs/axum/latest/axum/extract/struct.Path.html](https://docs.rs/axum/latest/axum/extract/struct.Path.html)

上記ページに記載されているサンプルです。

```rust,ignore
use axum::{
    extract::Path,
    routing::get,
    Router,
};
use uuid::Uuid;

async fn users_teams_show(
    Path((user_id, team_id)): Path<(Uuid, Uuid)>,  // 引数にPath<(T, ...)>型の変数を宣言することで取り扱いができる
) {
    // ...
}

let app = Router::new().route("/users/:user_id/team/:team_id", get(users_teams_show));
```

パスに `:` で始まる名前を含めると、その部分をパラメータと解釈して、リクエストされたときに `Path<(T, ...)>` 型の値として受け取ることができます。

> **Note:** 通常、Pathの型パラメータはタプルですが、パラメータが1つだけのときはタプルを省略できます。

Pathのタプルに使う型 `T` は `serde::Deserialize` を満たす必要があります。パラメータの値が型と合わない場合はエラーになります。 

## 更新系SQLの発行

更新系SQL文（INSERT, UPDATE, DELETE）を発行するためのメソッドは `execute` になります。メソッドの使い方は `query` と同じです。

```rust,ignore
let name = "Bob";
let id = 42;
let row = conn
        .execute("UPDATE hoge SET name = $1 WHERE id = $2", &[&name, &id])
        .await?;
```
