# HTMLレスポンス

このページではテンプレートエンジン [askama](https://github.com/djc/askama) を利用して、データを埋め込んだHTMLをレスポンスできるようにします。

<a class="source" href="https://github.com/tkzwhr/rustwi/compare/1...2?diff=split" target="_blank" rel="noopener noreferrer">
    <div class="icon">&nbsp;</div>
    <span>今回のコード</span>
</a>

## テンプレートエンジン

テンプレートエンジンを使うことで、テンプレートとなるHTMLをベースに、条件による表示/非表示の切り替えや配列データをループして表示させることができます。表示制御の書き方については [テンプレート記法](https://djc.github.io/askama/template_syntax.html) を確認してください。

askamaではデフォルトで `templates/` ディレクトリ配下のHTMLファイルをテンプレートと認識します。（コンフィギュレーションで変更可能です）

今回は、他のテンプレートから利用されるテンプレートも含め、3つ作成しています。

## 構造体の実装

テンプレートをプログラムから扱えるようにするためには、テンプレートに対応する構造体を作成する必要があります。

`templates/hoge.html` にベースをする場合、下記のような構造体を用意します。

<div class="filename"><div>templates/hoge.html</div></div>

```html
<html>
<body>
{{hoge_hoge}}
</body>
</html>
```

<div class="filename"><div>src/hoge.rs</div></div>

```rust,ignore
use askama::Template;

#[derive(Template)]  // Templateトレイトを実装
#[template(path = "hoge.html")]  // template(path)でHTMLファイルを指定
struct Hoge {
    hoge_hoge: String,
}
```

`#[template(...)]` に指定できる属性については、 [The template() attribute](https://docs.rs/askama/latest/askama/#the-template-attribute) を確認してください。

## IntoResponseへの変換

`Template` をderiveした構造体を用意したので、 `render()` を呼び出せるようになります。呼び出すことで構造体のフィールドのデータが埋め込まれたHTML文字列を `Result<String, Error>` 型で取得することができます。

```rust,ignore
fn handler() -> impl IntoResponse {
    let hoge = Hoge { "こんにちは".to_string() };
    let html = hoge.render().unwrap();
    Html(html).into_response()
}
```
