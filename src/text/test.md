# テスト

このページではサービス（ユースケース）のテストをしてみます。

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/6...7?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード</span>
</a>

## テスト概要

テストについては [The Rust Programming Language 日本語版](https://doc.rust-jp.rs/book-ja/ch11-00-testing.html) の11章にて解説されているため、そちらに沿ってテストを実装していきます。

## テストダブル

サービスはリポジトリに依存しており、テストをするためにはデータベースシステムを用意しないといけません。 もちろん用意しても良いのですが、純粋にサービスだけをテストしたい場合には手間が大きいですので、ここではテストダブルを使用したテストを書きます。

テストダブル用のクレートも様々なものがありますが、ここでは使い勝手の良い [mockall](https://github.com/asomers/mockall) を使用します。 サンプルを見てみましょう。

```rust,ignore
#[cfg(test)]
use mockall::{automock, mock, predicate::*};
#[cfg_attr(test, automock)]
trait MyTrait {
    fn foo(&self, x: u32) -> u32;
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn mytest() {
        let mut mock = MockMyTrait::new();
        mock.expect_foo()  // expect_関数名で、モック対象の関数を指定する
            .with(eq(4))  // 引数が4であるはず
            .times(1)  // 1度だけ呼ばれるはず
            .returning(|x| x + 1);  // 返却する値は「引数の数値+1」
        assert_eq!(5, mock.foo(4));
    }
}
```

モックしたいトレイト `MyTrait` に対して `#[automock]` を指定すると、 `MockHogeTrait` という構造体が自動生成されます。これはテストのときにだけ欲しいので、各所で `#[cfg(test)]` による条件分岐を行って、コンパイルを制御しています。

> **Note:** `#[cfg_attr(A, B)]` は `#[cfg(A)]` のときだけ `#[B]` が指定されていると解釈してコンパイルします。

## 非同期処理のテスト

非同期処理には `#[test]` が使えません。代わりに `#[tokio::test]` を使用します。
