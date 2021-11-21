# セッション

このページでは、ログインしたユーザーのセッションを保持する仕組みを実装します。

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/9...10?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード（アカウントモデルの作成）</span>
</a>

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/10...11?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード（セッションの保存）</span>
</a>

## データベースのテーブル追加

アカウントモデルを実装したので、それを保存するためのテーブルが必要です。データベースに入り、作成しておきます。

```sql
CREATE TABLE accounts
(
    id           serial primary key,
    email        varchar(256) not null unique,
    password     varchar(64)  not null,
    display_name varchar(16)  not null
);
```

## セッション管理のセットアップ

ユーザーセッションを管理するクレートも様々なものがありますが、ここではaxumのサンプルでも利用されている [async-session](https://github.com/http-rs/async-session) を使用します。

また、セッションの保存先にもデータベースを利用したいので、 [async-sqlx-session](https://github.com/jbr/async-sqlx-session) を使ってデータベースに保存できるようにします。サンプルを見てみましょう。

```rust,ignore
use async_sqlx_session::PostgresSessionStore;
use async_session::{Session, SessionStore};
use std::time::Duration;

let store = PostgresSessionStore::new(&std::env::var("PG_TEST_DB_URL").unwrap()).await?;  // --(1)
store.migrate().await?;  // --(2)
store.spawn_cleanup_task(Duration::from_secs(60 * 60));  // --(3)

let mut session = Session::new();  // --(4)
session.insert("key", vec![1,2,3]);

let cookie_value = store.store_session(session).await?.unwrap();  // --(5)
let session = store.load_session(cookie_value).await?.unwrap();  // --(6)
assert_eq!(session.get::<Vec<i8>>("key").unwrap(), vec![1,2,3]);
```

セッションストアを使う流れは下記のとおりです。

1. データベース接続情報を渡して `SessionStore` を作成する（今回は async-sqlx-session を使っているため `PostgresSessionStore` を作成している）
2. （任意）セッション保存用のテーブルがなければ作成する
3. （任意）有効期限切れのセッションを削除する処理を定期実行する設定をする
4. セッションを作成し、 `key = value` の形で保存するデータを格納する
5. セッションストアにセッションを渡して保存する（Cookie文字列が返却される）
6. Cookie文字列をキーにしてセッションストアからセッションを取得する

## セッションの作成

セッションを作成するとキーとなるCookie文字列が取得できるので、ログイン成功時にこのクッキー文字列を返却します。

<div class="filename"><div>src/controllers/accounts.rs</div></div>

```rust,ignore
async fn new_session(
    form: Form<SignInForm>,
    Extension(repository_provider): Extension<RepositoryProvider>,
) -> Result<impl IntoResponse, impl IntoResponse> {
    let account_repo = repository_provider.accounts();
    if let Some(session_token) =
        services::create_session(&account_repo, &form.email, &form.password).await
    {
        let headers = Headers(vec![("Set-Cookie", session_token.cookie())]);  // レスポンスヘッダーの作成
        let response = Redirect::to(Uri::from_static("/"));
        Ok((headers, response))
    } else {
        Err(Redirect::to(Uri::from_static("/login?error=invalid")))
    }
}
```

axumではレスポンス時に `Headers` を使ってレスポンスヘッダーの指定ができるため、 `Set-Cookie` ヘッダーをセットしてレスポンスし、Cookieをブラウザに保存させています。 `<T: IntoResponse>(Headers, T)` 型も `IntoResponse` を実装しています。

## セッションのチェック

セッションがなかったり、有効期限切れの場合はログイン画面を表示する必要があります。このようにリクエストの検証を行いたい場合には `FromRequest` トレイトを実装します。

> **Note:** 今まで使ってきた `Extractor` も `FromRequest` を実装しています。このように、 `FromRequest` は非常に応用の幅が広いトレイトになっています。

`FromRequest` を使って [セッションを検証するサンプル](https://docs.rs/axum/latest/axum/extract/index.html#accessing-other-extractors-in-fromrequest-implementations) を見てみましょう。

```rust,ignore
#[derive(Clone)]
struct State {
    // ...
}

struct AuthenticatedUser {
    // ...
}

#[async_trait]
impl<B> FromRequest<B> for AuthenticatedUser
where
    B: Send,
{
    // （追記）Rejectionに from_request() が失敗した場合の型を宣言する
    type Rejection = Response;

    async fn from_request(req: &mut RequestParts<B>) -> Result<Self, Self::Rejection> {
        // （追記）Authorizationヘッダーの値を読み取る
        let TypedHeader(Authorization(token)) = 
            TypedHeader::<Authorization<Bearer>>::from_request(req)
                .await
                .map_err(|err| err.into_response())?;

        // （追記）共有しているstateの情報を取得する
        let Extension(state): Extension<State> = Extension::from_request(req)
            .await
            .map_err(|err| err.into_response())?;

        // actually perform the authorization...
        unimplemented!()
        
        // （追記）これ以降の処理として、tokenとstateのデータを見て
        //         Ok(...)を返すかErr(...)を返すか制御する必要がある
    }
}

async fn handler(user: AuthenticatedUser) {  // ハンドラー関数の引数に追加するとAuthenticatedUserのfrom_requestが実行される
    // （追記）AuthenticatedUser#from_requestがOk(...)のときにだけ実行される
    //         Err(...)の場合は実行されずにErrの中身が返却される
    
    // ...
}

let state = State { /* ... */ };

let app = Router::new().route("/", get(handler)).layer(AddExtensionLayer::new(state));
```

このサンプルでは

- `Authorization: Bearer XXXXXX` というヘッダーがクライアントから指定されリクエストされてくることを想定している
- `State` 構造体のデータをアプリケーションサーバーで共有情報として持っている
    - つまりアプリケーションサーバーが再起動されると共有情報が消える

という前提で書かれているようです。

同じ要領で、今回のセッションの仕組みに対応した `FromRequest` を実装しています。

大まかな流れは下記のとおりです。

1. リクエストヘッダーにある Cookie を読み取り、予め発行しておいた値（キーが `AXUM_SESSION_COOKIE_NAME` のもの）を読み取る
2. 読み取った値でセッションストアに問い合わせを行い、保存しているユーザーIDを取得する
3. ユーザーIDを含む `UserContext` をハンドラー関数に返却する
4. 上記までのどこかでエラーが発生した場合はログイン画面にリダイレクトする

<div class="filename"><div>src/request.rs</div></div>

```rust,ignore
use crate::constants::{database_url, AXUM_SESSION_COOKIE_NAME, AXUM_SESSION_USER_ID_KEY};
use async_session::SessionStore;
use async_sqlx_session::PostgresSessionStore;
use axum::extract::{FromRequest, RequestParts, TypedHeader};
use axum::headers::Cookie;
use axum::http::Uri;
use axum::response::Redirect;
use serde::{Deserialize, Serialize};

#[derive(Deserialize, Serialize)]
pub struct UserContext {
    pub user_id: i32,
}

#[axum::async_trait]
impl<B> FromRequest<B> for UserContext
where
    B: Send,
{
    type Rejection = Redirect;

    async fn from_request(req: &mut RequestParts<B>) -> Result<Self, Self::Rejection> {
        let redirect = || Redirect::to(Uri::from_static("/login"));

        let database_url = database_url();
        let store = PostgresSessionStore::new(&database_url)
            .await
            .map_err(|_| redirect())?;
        let cookies = Option::<TypedHeader<Cookie>>::from_request(req)
            .await
            .unwrap()
            .ok_or(redirect())?;
        let session_str = cookies.get(AXUM_SESSION_COOKIE_NAME).ok_or(redirect())?;
        let session = store
            .load_session(session_str.to_string())
            .await
            .map_err(|_| redirect())?;
        let session = session.ok_or(redirect())?;
        let context = UserContext {
            user_id: session.get::<i32>(AXUM_SESSION_USER_ID_KEY).unwrap(),
        };
        Ok(context)
    }
}
```

あとは、認証が必要なリクエストのハンドラー関数の引数に `UserContext` を追加すれば完了です。
