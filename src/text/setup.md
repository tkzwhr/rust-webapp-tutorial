# フレームワークのセットアップ

今回は非同期ランタイムライブラリで有名な [tokio](https://tokio.rs/) が開発している [axum](https://github.com/tokio-rs/axum) を使います。axumはマクロを使わずに作られており、見通しの良い実装がし易くなっています。

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/0...1?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード</span>
</a>

## Hello, world!

まずはRustプロジェクトを作成します。

```shell
$ mkdir rustwi
$ cd rustwi
$ cargo init
     Created binary (application) package
```

次に `Cargo.toml` の `[dependencies]` 節にクレートを追加します。

<div class="filename"><div>Cargo.toml</div></div>

```toml
[package]
name = "rustwi"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.4"
tokio = { version = "1.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version="0.3", features = ["env-filter"] }
```

メインのソースコードも変更します。

<div class="filename"><div>src/main.rs</div></div>

```rust,ignore
use axum::{response::Html, routing::get, Router};
use std::net::SocketAddr;

#[tokio::main]  // main関数を非同期関数にするために必要
async fn main() {
    if std::env::var_os("RUST_LOG").is_none() {
        std::env::set_var("RUST_LOG", "rustwi=debug")
    }
    tracing_subscriber::fmt::init();

    let app = Router::new().route("/", get(handler));

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::debug!("listening on {}", addr);
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}
```

## ルーティング

ルーティングは `Router` で行います。`Router#route` でハンドラー関数を登録します。

```rust,ignore
fn main() {
    let app = Router::new().route("/", get(handler));
}

async fn handler() -> impl IntoResponse {
    // ...
}
```

上記のルーティングは、あえてマクロで表現すると

```rust,ignore
#[get("/")]
async fn handler() -> impl IntoResponse {
    ...
}
```

のようなイメージです。 **（axumでは上記のコードは動作しません）**

## ハンドラー関数

ハンドラー関数では、 `Extractor` という仕組みで、リクエストデータを引数で受け取ることができます。これについては以降のページで説明していきます。 

戻り値は `IntoResponse` トレイトに準拠したオブジェクトを返す必要があります。例えば、前述の `Html<&'static str>` も `IntoResponse` を実装しています。

その他にも、axumは様々な型に対して `IntoResponse` を実装してくれているので、そちらを確認してみると良いでしょう。

[https://docs.rs/axum/latest/axum/response/trait.IntoResponse.html](
https://docs.rs/axum/latest/axum/response/trait.IntoResponse.html)

> **Note:** Routerにはハンドラー関数だけでなく、 [towerのService](https://docs.rs/tower-service/latest/tower_service/trait.Service.html) に準拠したサービスを登録することもできます。
> 
> axumに [指定したディレクトリをホスティングする例](https://github.com/tokio-rs/axum/blob/main/examples/static-file-server/src/main.rs#L25) があるので、それを見てみるのも良いでしょう。

## サーバーの起動

サーバーを起動してみましょう。cargoコマンドで起動できます。

```shell
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running `target/debug/rustwi`
2021-01-01T09:00:00.000000Z DEBUG rustwi: listening on 127.0.0.1:3000
```

ブラウザを開いて [http://localhost:3000/](http://localhost:3000/) にアクセスすると、 `Hello, World!` と表示されるはずです。数行のコードでウェブサービスを立ち上げることができました！

次ページ以降で、より柔軟なHTMLを返却する方法を学びます。
