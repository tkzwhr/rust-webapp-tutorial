# テーブルリレーション

このページでは、2つのテーブルがリレーションしている状態でデータの保存と取得をしてみます。

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/11...12?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード</span>
</a>

## テーブルのリレーション追加

まず、前回作成したアカウントとツイートを関連させるために、データベースのテーブル定義を変更します。

```sql
DELETE FROM tweets;
ALTER TABLE tweets ADD COLUMN posted_by integer not null;
ALTER TABLE tweets ADD CONSTRAINT fk_tweets_posted_by_accounts FOREIGN KEY (posted_by) REFERENCES accounts (id);
```

テーブルに列を足すにあたり、ツイートデータが残っていると都合が悪いため、事前に削除しています。

## 複数テーブルのデータからビューの作成

複数テーブルのデータを取得してビューを作成する方法は2通りあります。

1. `INNER JOIN` 等を使ったSQLを発行し、データベースで連結した状態のデータを取得する
2. それぞれのテーブルからエンティティを取得し、プログラムでそれぞれのエンティティを組み合わせてビューのデータを作成する

今回は2の方法で取得しています。1の方法は1度のSQL発行で済むため **処理性能が良い** のですが、連結した状態のデータがエンティティではない場合、コードが荒れるのを防ぐために [CQS](https://rakusui.org/cqs/) など別の概念を導入してコードを整理する必要があり、少し実装が難しいです。とはいえ、実際は1の方法を採ることが多いので、今後のチャレンジとしても良いでしょう。

<div class="filename"><div>src/services/tweets.rs</div></div>

```rust,ignore
pub async fn list_tweets(repo: &impl Tweets, account_repo: &impl Accounts) -> Home {
    // ツイート一覧を取得する
    let tweets = repo.list().await;
    // ツイート一覧にある posted_by を重複無しの一覧にする
    let posted_account_ids = tweets.iter().map(|x| x.posted_by).collect::<HashSet<i32>>();
    // posted_by のidでアカウント一覧を取得する
    let accounts = account_repo.find(posted_account_ids).await;
    let tweets = tweets
        .into_iter()
        .map(|x| {
            let account = accounts.get(&x.posted_by).unwrap();
            // ツイートとアカウントのタプルからビューを作る
            (x, account).into()
        })
        .collect();
    Home { tweets }
}
```

<div class="filename"><div>src/views/partial/tweet.rs</div></div>

```rust,ignore
use crate::entities::{Account, Tweet as TweetEntity};

pub struct Tweet {
    pub id: String,
    pub name: String,
    pub message: String,
    pub posted_at: String,
}

impl From<(TweetEntity, &Account)> for Tweet {
    fn from(e: (TweetEntity, &Account)) -> Self {
        Tweet {
            id: e.0.id().unwrap_or(-1).to_string(),
            name: e.1.display_name.clone(),
            message: e.0.message,
            posted_at: e.0.posted_at.format("%Y/%m/%d %H:%M").to_string(),
        }
    }
}
```

## tokio-postgres での where in の使用について

tokio-postgres では、`where` 句の `in` に パラメータとして `&Vec` を渡すことができませんでした。

```rust,ignore
let ids = vec![1, 2, 3];
let rows = conn
    .query(
        "SELECT * FROM accounts WHERE id in ($1)",
        &[&ids],
    )
    .await
    .unwrap();
```

展開された状態であれば大丈夫なようです。（が、これでは `in` に指定するパラメータの数が限定されてしまい、使い勝手が悪いです。）

```rust,ignore
let rows = conn
    .query(
        "SELECT * FROM accounts WHERE id in ($1,$2,$3)",
        &[&1, &2, &3],
    )
    .await
    .unwrap();
```

そこで、`in` に指定する文字列を生成し、SQL文に直接指定する方法にしています。

```rust,ignore
let ids = vec![1, 2, 3];
let ids_str = ids
    .into_iter()
    .map(|x| x.to_string())
    .collect::<Vec<String>>()
    .join(",");
let rows = conn
    .query(
        &format!("SELECT * FROM accounts WHERE id in ({})", ids_str),
        &[],
    )
    .await
    .unwrap();
```

> **Note:** リクエストのパラメータを使用する場合は、別途、SQLインジェクション対策が必要です。
