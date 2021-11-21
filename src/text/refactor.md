# リファクタ

このページでは、データの取得・保存と要件実現のための処理の依存性を弱くし、メンテナンスしやすい作りにします。

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/5...6?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード</span>
</a>

## 役割ごとにモジュール化

メンテナンス性を高めるために重要なことは下記2点です

- 技術的な知識と要件実現のための知識を分離する
- 役割の依存方向性を明確にし逆方向への依存をしないようにする

これらを満たす方法として、 [クリーンアーキテクチャ](https://nrslib.com/clean-architecture/) を簡易的に導入してみます。

すでにこのアプリケーションはモジュール化の準備ができているため、下記の単位でモジュールを切ります。

|モジュール|役割|利用されている|技術的関心事|
|:--|:--|:--|:--|
|コントローラー|Webフレームワークを取り扱う|-|axum|
|サービス[^1]|ユースケースを実装する|コントローラー|-|
|エンティティ|ビジネスモデルを実装する|リポジトリ|-|
|リポジトリ（定義）|エンティティの取得・保存の仕様を定義する|サービス|-|
|リポジトリ（実装）|リポジトリ（定義）に沿って具体的な処理を実装したもの|-|PostgreSQL|
|フォーム|フォームデータのデータ構造を定義する|コントローラー|-|
|ビュー|レスポンスのデータ構造を定義する|サービス|-|

## `From` トレイトの利用

階層化して管理すると、逆方向へ依存しないようにするために、構造体のデータの詰め替えが必要になってきます。 例えば、エンティティが表示用の機能を持つことは依存性が逆転してしまうためできません。そのためビューを別途用意し、エンティティからビューに変換する処理が必要になってきます。

このような型変換が必要なケースにおいては、 `From` トレイトや `Into` トレイトを使うのがベターです。 [Rust By Example 日本語版](https://doc.rust-jp.rs/rust-by-example-ja/conversion/from_into.html) にこのトレイトの使用例があります。

このプロジェクトでも `From` トレイトを使用した変換を行っています。

<div class="filename"><div>src/views/partial/tweet.rs</div></div>

```rust,ignore
use crate::entities::Tweet as TweetEntity;

pub struct Tweet {
    pub name: String,
    pub message: String,
    pub posted_at: String,
}

impl From<TweetEntity> for Tweet {  // エンティティをビューに変換する処理を実装する
    fn from(e: TweetEntity) -> Self {
        Tweet {
            name: "太郎".to_string(),
            message: e.message,
            posted_at: e.posted_at.format("%Y/%m/%d %H:%M").to_string(),
        }
    }
}
```

<div class="filename"><div>src/services/tweets.rs</div></div>

```rust,ignore
use crate::repositories::Tweets;
use crate::views::Home;

pub async fn list_tweets(repo: &impl Tweets) -> Home {
    let tweets = repo.list().await;
    Home {
        tweets: tweets.into_iter().map(|x| x.into()).collect(),  // ここで .into() を呼び出し、エンティティをビューに変換している
    }
}
```

---

[^1]: クリーンアーキテクチャでは、この役割は UseCase と Interactor に分かれていますが、今回はそこまではしていません。