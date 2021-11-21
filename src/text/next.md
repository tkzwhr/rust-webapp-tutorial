# おわりに

おめでとうございます！ここまでで、rustとaxumを使ったウェブアプリの開発が一通りできるようになりました。今後は下記のチャレンジをしてみるのが良いでしょう。

- 今回スコープアウトしたグレーになっているユースケースを機能拡張してみる
- バリデーションを実装してみる
    - axumに [バリデーションの例](https://github.com/tokio-rs/axum/tree/main/examples/validator) があります
- 別途Reactなどのフロントエンドのフレームワークを用意し、JSONリクエストとJSONレスポンスをするAPIサーバーに作り変えてみる
    - axumに [JSONを使ったTODO管理アプリの例](https://github.com/tokio-rs/axum/tree/main/examples/todos) があります
- ファイルアップロードできるようにする
    - axumに [multipart/form-dataを取り扱う例](https://github.com/tokio-rs/axum/tree/main/examples/multipart-form) や [ファイル配信する例](https://github.com/tokio-rs/axum/tree/main/examples/static-file-server) があります
- Socket通信に対応してみる
    - axumに [Websocketの例](https://github.com/tokio-rs/axum/tree/main/examples/websockets) があります

その他 [多くの例](https://github.com/tokio-rs/axum/tree/main/examples) が実装されているので、一度覗いてみると良いと思います。
