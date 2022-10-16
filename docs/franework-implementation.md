# フレームワークの実装を見てみる
この章では、いくつかのフレームワークやライブラリにおいて、CSRF の対策がどのように行われているかを確認します。
これを通して、今まで見てきた CSRF のどの対策が現実で使われているのかや、フレームワークを使う理由のひとつを確認してもらえればと思います。

## Ruby on Rails
広く使わている Web アプリを作るためのフレームワークです。チュートリアルが非常に充実しており、勉強になります。

まずドキュメントには、GET と POST を適切に使い分けましょうと書いてあります。

Rails では、コントローラーと呼ばれる、リクエストを受け取る最外部に、以下のコードを追加するだけで、csrf token を自動で追加し、検証してくれるようです。また、デフォルトでこの設定が使われているようです。
```
protect_from_forgery with: :exception
```
https://railsguides.jp/security.html#csrf%E3%81%B8%E3%81%AE%E5%AF%BE%E5%BF%9C%E7%AD%96

具体的には、csrf token を cookie とフォームへ埋め込み、それらをサーバーで突き合わせる、[cookie の二重送信](/guard-csrf-using-token-part2)方法を取っているようです。また、`XMLHttpRequest`などを使うときのために、フォームへの埋め込みではなくヘッダを使った方法もサポートしているようです。

ここで考えなければならないのは、クッキーモンスターバグです。cookie が自由に設定できてしまうこの脆弱性がある場合、cookie に適当に値をセットしてしまえば、それと同じ値をフォームへ埋め込むことで、攻撃が可能になってしまいます。
この攻撃に対して、Rails ではどのように対策しているのでしょうか。

どうやら、cookie を暗号化しているようです。Rails にはセッション情報を格納できるメソッドがあり、それらはブラウザに cookie の形で送信されます。ただし、その cookie はブラウザに送信されるときに Rails 用に生成された秘密鍵で暗号化されます。そのため、秘密鍵を知らなければ csrf token を偽装することもできなさそうです。

> Your cookies will be encrypted using your application's secret_key_base
https://api.rubyonrails.org/classes/ActionDispatch/Session/CookieStore.html

https://techracho.bpsinc.jp/hachi8833/2021_11_26/46891

また、session 情報を保存する場所を DB などに変更することも可能なようです。
> 3.9.1 config.action_dispatch.session_store
セッションデータのストア名を設定します。デフォルトのストア名は`:cookie_store`です。この他に`:active_record_store`、`:mem_cache_store`、またはカスタムクラス名なども指定できます。
https://railsguides.jp/configuring.html#action-dispatch%E3%82%92%E8%A8%AD%E5%AE%9A%E3%81%99%E3%82%8B

また、`XMLHttpRequest`では cookie とカスタムヘッダにより検証を行っているのですが、カスタムヘッダに入れるための値は`<meta name='csrf-token' content='THE-TOKEN'>`に格納されています。この値をみることができれば攻撃できてしまうのですが、同一オリジンポリシーがあるため、攻撃者はこの値を取得することができません。

他にも、Origin のチェックなどを行っているようです。
https://edgeguides.rubyonrails.org/upgrading_ruby_on_rails.html#forgery-protection-with-origin-check


## Laravel
PHP のフレームワークです。
Laravel では、セッションごとに csrf token が生成されます。フォームの場合は、そのトークンを使うことを明示的に宣言してトークンをフォームに埋め込み、 CSRF の対策を行うようです。
宣言の方法は、`@csrf`を書くだけなので、楽ですね。検証はミドルウェアが自動でしてくれるようです。
また、`XMLHttpRequest`などでは、Rails と同様に`meta`タグを用いてトークンをクライアント側に渡し、HTTP ヘッダを使ってサーバー側に返す手法が推奨されるようです。
https://laravel.com/docs/9.x/csrf

注目したいのは、csrf token の生成がセッションごとだという点です。Rails では、フォーム毎に csrf token が生成されていました。その理由は`code-injection attacks with forms created by JavaScript`への対策と書いてあります。（どのような攻撃かは調べられていません）
https://edgeguides.rubyonrails.org/upgrading_ruby_on_rails.html#per-form-csrf-tokens

## Express
Node.js の有名なフレームワークです。この本のサンプルアプリにも使われています。最小限のフレームワークという謳い文句を掲げています。

最小限のフレームワークという通り、デフォルトでは CSRF の対策は行っていません。ではどうするかというと、ミドルウェアと呼ばれるプラグインのようなものを追加するか、サンプルアプリでやっているように自分で追加することになります。

追加できるミドルウェアのうち、公式がメンテナンスをしているものがありました。しかし、公式のリソースが足りないことと、CSRF の脅威が Nodejs で使われるようなアプリケーション構成では無視できる場合が多いことを理由に、現在ではメンテナンスされていません。
> The Express.js project does not have the resources to put into this module, which is largely unnecessary for modern SPA-based applications.
https://github.com/expressjs/csurf

代わりに、サードパーティーが開発しているミドルウェアを、npm という Node.js のパッケージリポジトリで適宜探して使うように指示しています。
https://www.npmjs.com/search?q=express%20csrf