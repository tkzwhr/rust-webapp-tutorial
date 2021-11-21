# データベースのセットアップ

フォームデータも受け取れるようになったので、このページからデータベースを使ってデータの永続化をしていきます。

> **Note:** 今回はデータベースにPostgreSQLを使用します。

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/4...5?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード</span>
</a>

## データベースのセットアップ

dockerなどを利用して、ローカル環境にデータベースを起動しておきましょう。

```shell
$ docker run --rm -d -p 5432:5432 -e POSTGRES_HOST_AUTH_METHOD=trust postgres:14-alpine
```

データベースに入り、必要なテーブルを作成しておきます。

```sql
CREATE TABLE tweets
(
    id        serial primary key,
    message   text        not null,
    posted_at timestamptz not null
);
```

## データベース取り扱いの準備

rustのデータベース用クレートはいくつかありますが、今回はaxumのサンプルでも使用されている [bb8](https://github.com/djc/bb8) を使用します。

rustで使えるDBドライバーもいくつか提供されていますが、今回は非同期処理での取り扱いに優れている [tokio-postgres](https://github.com/sfackler/rust-postgres) を利用します。

bb8 で tokio-postgres を直接利用はできないのですが、 bb8-postgres というアダプターが bb8 に同梱されているので、それを使用します。サンプルを見てみましょう。

```rust,ignore
use bb8::Pool;
use bb8_postgres::PostgresConnectionManager;
use tokio_postgres::NoTls;

fn main() {
    let manager = PostgresConnectionManager::new_from_stringlike("postgres://postgres@localhost/postgres", NoTls).unwrap();  // --(1)
    let pool = Pool::builder()
        .max_size(15)
        .build(manager)  // --(2)
        .unwrap();

    for _ in 0..20 {
        let pool = pool.clone();
        tokio::spawn(move || {
            let conn = pool.get().await.unwrap();  // --(3)
            // use the connection
            // it will be returned to the pool when it falls out of scope.
        })
    }
}
```

データベースを使う流れは下記のとおりです。

1. データベース接続情報を渡して `ConnectionManager` を作成する（今回は bb8-postgres を使っているため `PostgresConnectionManager` を作成している）
2. `ConnectionManager` を渡して `ConnectionPool` を作成する
3. `ConnectionPool` から `PooledConnection` を取得する
4. `PooledConnection` を使ってSQLを発行する

> **Note:** データベースへの接続は時間がかかるので、使うたびに接続と切断をするのではなく、使い終わったあとも接続したままにしておき、また使いたくなったときに再利用するのが一般的です。この再利用のために使い終わった接続を管理しておくのが `ConnectionPool` の役割です。

## dotenvの使用

上記ではデータベースの接続情報を直接記載していますが、通常、セキュリティの観点からこのような実装をすることはなく、環境変数などから取得するのが一般的です。

サービスをデプロイする際には環境変数から取得できる前提で、ローカル開発では [dotenv](https://github.com/dotenv-rs/dotenv) を使うことで、環境変数への変数設定を簡略化します。

下記はサンプルです。

```rust,ignore
use dotenv::dotenv;
use std::env;

fn main() {
    dotenv().ok();  // 「.env」ファイルに設定した内容が環境変数として設定される

    for (key, value) in env::vars() {
        println!("{}: {}", key, value);
    }
}
```

ルートディレクトリに `.env` というファイルを作成し、 `KEY=VALUE` の形式で記載することで、 `dotenv` クレートは実行時にそれを読み取って環境変数に設定してくれます。

## 取得系SQL文の発行

tokio-postgres では、発行するSQLは文字列として書きます。

```rust,ignore
let id = 42;
let row = conn
        .query_opt("SELECT * FROM hoge WHERE id = $1", &[&id])
        .await?;
```

パラメータはSQL文中では `$1, $2, ...` と記載し、第2引数で参照のスライスを渡します。

取得系SQL文（SELECT）を発行するためのメソッドは3種類あります。

- query
    - 複数行を取得するSQLを発行する場合に使い、結果を `Vec<Row>` で受け取れます
    - データが取得できないケースでも正常終了します
- query_one
    - 1行を取得するSQLを発行する場合に使い、結果を `Row` で受け取れます
    - データが取得できない、または、2行以上取得できるケースではエラー終了します
- query_opt
    - 1行を取得するSQLを発行する場合に使い、結果を `Option<Row>` で受け取れます
    - データが取得できないケースでも正常終了します
    - 2行以上取得できるケースではエラー終了します

## `Extension` の利用

上記の流れで作成した `ConnectionPool` は、何回リクエストが来ても、また、別のメソッドにリクエストが来ても同じ `ConnectionPool` を参照して `PooledConnection` を取得できる必要があります。（そうでないと接続の再利用ができません）

axumには `Extension` というものがあり、これを使うことで全てのリクエストで同じオブジェクトを共有することができます。

[https://docs.rs/axum/latest/axum/extract/struct.Extension.html](https://docs.rs/axum/latest/axum/extract/struct.Extension.html)

上記のドキュメントにあるサンプルを見てみましょう。

```rust,ignore
use axum::{
    AddExtensionLayer,
    extract::Extension,
    routing::get,
    Router,
};
use std::sync::Arc;

// Some shared state used throughout our application
struct State {
    // ...
}

async fn handler(state: Extension<Arc<State>>) {  // 引数でExtensionを受け取る
    // ...
}

let state = Arc::new(State { /* ... */ });

let app = Router::new().route("/", get(handler))
    // Add middleware that inserts the state into all incoming request's
    // extensions.
    .layer(AddExtensionLayer::new(state));  // ExtensionをRouterに追加する
```

`Extension` も `Extractor` の一種なので、ハンドラー関数の引数で受け取ることができます。
