# フォームリクエスト

このページではフォームから投稿された内容を取り扱ってみます。

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/3...4?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード</span>
</a>

## ハンドラー関数でデータを受け取る

リクエストデータを取り扱うには、 `axum::extract::Form` を使用します。

[https://docs.rs/axum/latest/axum/extract/struct.Form.html](https://docs.rs/axum/latest/axum/extract/struct.Form.html)

上記ページに記載されているサンプルです。

```rust,ignore
use axum::{
    extract::Form,
    routing::post,
    Router,
};
use serde::Deserialize;

#[derive(Deserialize)]  // Form<T>のTはserde::Deserializeを実装している必要がある
struct SignUp {
    username: String,
    password: String,
}

async fn accept_form(form: Form<SignUp>) {  // 引数にForm<T>型の変数を宣言することで取り扱いができる
    let sign_up: SignUp = form.0;

    // ...
}

let app = Router::new().route("/sign_up", post(accept_form));
```

`serde` はデータ構造をシリアライズ/デシリアライズするためのユーティリティです。例えば、構造体をjsonにする場合（逆の場合も）などにも使われます。必要に応じて `Cargo.toml` に依存関係を追加してください。

このサンプルは、下記のフォーム送信に対応します。

```html
<form action="/sign_up" method="post">
  <input type="text" name="username" placeholder="ユーザー名" />
  <input type="password" name="password" placeholder="パスワード" />
  <button type="submit">送信</button>
</form>
```
