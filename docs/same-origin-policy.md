# 同一オリジンポリシー

前節では、csrf token を用いた防御手法を解説しました。
この節では、csrf token を用いた場合に考えられる攻撃手法を挙げ、その手法が同一オリジンポリシーによって無効化されていることを確認します。

## csrf token は本当に安全？

前節で紹介した csrf token を使った手法は、ランダムに生成されたトークンをサーバー側とクライアント側で保持して、それらを突き合わせてリクエストの検証を行うという手法でした。このような防御手法は、ランダムに生成されたトークンが流出すると安全性が保てなくなります。

では、前節で紹介したような、csrf token をフォームに埋め込むという方法に、トークンが流出する危険性はないのでしょうか？
たとえば、`XMLHttpRequest` や、 `fetch` という JavaScript から使用できる機能を使えば、GET リクエストをブラウザから送ることができます。これを使って、正規サイトに GET リクエストを送り、そのレスポンスから有効な csrf token を抜き出すことができそうな気がします。

## 実際に攻撃してみる with XMLHttpRequest

次のようなページを作成して、GET リクエストのレスポンスの取得を試してみます。ここで、csrf token が取得できれば、その csrf token を使って悪意のあるリクエストを送って攻撃することができます。

```html
<script>
  // XMLHttpRequest を初期化する
  const req = new XMLHttpRequest();
  // URL と HTTP メソッドを指定する
  req.open("GET", "http://localhost:3000/3-post-password-change-csrf-token");
  // レスポンスが返ってきたときの処理を指定する
  req.onreadystatechange = () => {
    if (req.readyState === XMLHttpRequest.DONE) {
      console.log("status: " + req.status);
      console.log("response: " + req.responseText);
    }
  };
  // いままでの設定でリクエストを送る
  req.send();
</script>
```

このスクリプトを埋め込んだページが次のリンクです。リンクを開いたら、検証画面からコンソールを確認してください。
http://localhost:4000/3-post-password-change-with-xml-http-request

画像のようなのエラーが発生しているでしょうか。
![](img/same-origin-policy_20221020135940.png)

response も空になっていて、うまく取得できていません。

次に、以下のリンクを開いて見てください。こちらは、正規サイトで攻撃サイトと同じ HTML を表示したものです。
http://localhost:3000/3-post-password-change-with-xml-http-request

今度はきちんと取得できているようです。二つのページの HTML は全く同じです。逆に異なる点は、URL の部分、正確にはポート番号のみです。
実際にエラーメッセージで文句を言われているのも、`http://localhost:4000`の部分のようですね。

> Access to XMLHttpRequest at 'http://localhost:3000/3-post-password-change-csrf-token' from origin 'http://localhost:4000' has been blocked by CORS policy: No Access-Control-Allow-Origin' header is present on the requested resource.

これについては、あとで説明します。今は、どうやら攻撃者のサイトから csrf token を取得できないらしい、ということがわかれば大丈夫です。

## 実際に攻撃してみる with iframe

理解を深めるために、異なる手法で csrf token の取得を試してみます。

HTML には、`iframe` というタグがあります。これは、他ページを埋め込むために使われるもので、たとえば youtube や twitter の画面を埋め込むなどの用途で使われます。
さて、ブラウザ上の JavaScript は、画面上に表示しているものにアクセスして操作することができます。今までも `getElementById` などを使ってフォームを操作していました。

では、iframe に csrf token を埋め込んだフォームがあるページを埋め込んで、その中身を `getElementById`で取得することはできるのでしょうか？
具体的には、以下のようなページで csrf token を取得することはできるのでしょうか？

```html
<iframe
  id="inlineFrame"
  width="400"
  height="300"
  src="http://localhost:3000/3-post-password-change-csrf-token"
>
</iframe>
<script>
  // iframe を取得
  const iframe = document.getElementById("inlineFrame");
  console.log("try to load iframe content via js");
  // iframe の中身が完全に取得されるのを待つ
  iframe.contentWindow.onload = () => {
    // iframe の中身の構造をコンソールに出してみる
    console.log(iframe.contentWindow.document);
    // csrf token を取得してみる
    const csrfToken =
      iframe.contentWindow.document.getElementsByName("csrfToken")[0];
    console.log(csrfToken);
  };
</script>
```

次のリンクにアクセスして、コンソール画面を確認してみましょう。
http://localhost:4000/3-post-password-change-with-iframe

コンソールにはなにも表示されないと思います。画面上には表示されているので、きちんと通信はできているようです。
![](img/same-origin-policy_20221020142150.png)

では、次のリンクにもアクセスしてみてください。
http://localhost:3000/3-post-password-change-with-iframe

こちらはちゃんと中身が取得できたと思います。今回も HTML は同じで、ポート番号が違うだけです。
![](img/same-origin-policy_20221020143611.png)

## 同一オリジンポリシーという防御機構

上で挙げた二つの例の種明かしをします。
簡単にいうと、同一オリジンポリシーに基づいた処理がされた結果、`localhost:4000` ではコンテンツの中身が取得できず、`localhost:3000` では中身が取得できました。

同一オリジンポリシー (Same-Origin Policy / SOP) とは、異なるオリジン間でのコンテンツのやりとりに制限をかける仕様です。このポリシーに基づいて、各ブラウザなどが実装を行います。

MDN の解説
https://developer.mozilla.org/ja/docs/Web/Security/Same-origin_policy

### オリジンとは

次のページを参考にしてください。端的にいうと、URL のうち`http://localhost:3000`の部分です。
https://developer.mozilla.org/ja/docs/Web/Security/Same-origin_policy#%E3%82%AA%E3%83%AA%E3%82%B8%E3%83%B3%E3%81%AE%E5%AE%9A%E7%BE%A9

### どのように同一オリジンポリシーが影響したのか

実際の事例に照らし合わせながら見ていきましょう。

#### XMLHttpRequest の例の解説

まずは、`XMLHttpRequest`の例です。この例では、localhost:4000 のページから、localhost:3000 にネットワーク越しに GET リクエストを送ってアクセスしようとしています。これは、同一オリジンポリシーの以下の部分で禁止されています。

> reading information from another origin is forbidden.
> https://www.rfc-editor.org/rfc/rfc6454#section-3.4.2

そのため、ブラウザは GET リクエストのレスポンスをコンソールに表示させなかったのです。

ネットワーク越しのアクセスについては、MDN にも解説が載っています。
https://developer.mozilla.org/ja/docs/Web/Security/Same-origin_policy#%E7%95%B0%E3%81%AA%E3%82%8B%E3%82%AA%E3%83%AA%E3%82%B8%E3%83%B3%E3%81%B8%E3%81%AE%E3%83%8D%E3%83%83%E3%83%88%E3%83%AF%E3%83%BC%E3%82%AF%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9

#### iframe の例の解説

次に、`iframe`の例です。この例では、localhost:4000 のページから、localhost:3000 に`iframe`を使った GET リクエストを送信しています。これは、先ほど挙げた読み取り禁止の規約では許されないはずですが、実は`iframe`を使ったコンテンツの表示は許可されています。

> an origin can display content from another origin, such as an HTML document in an HTML frame.
> https://www.rfc-editor.org/rfc/rfc6454#section-3.4.2

では、なぜ csrf token を取得できなかったかというと、異なるドメイン間でのオブジェクトの読み取りが許可されていないためです。具体的には、RFC の以下の部分で定義されています。

> Most objects (also known as application programming interfaces or APIs) exposed by the user agent are available only to the same origin.
> https://www.rfc-editor.org/rfc/rfc6454#section-3.4.1

`iframe.contentWindow.document`は、ブラウザが公開している API ですので、異なるドメインから読み取ることができないのです。

### 同一オリジンポリシーの落とし穴

ここで、同一オリジンポリシーでそもそも危険なクロスオリジンのリクエストを送らないようにすればいいのでは？という疑問を持った方もいると思います。
実際 RFC でもこのことには言及されていて、以下のように危険性を認めています。

> the ability to send network requests to another origin gives rise to cross-site request forgery vulnerabilities.
> https://www.rfc-editor.org/rfc/rfc6454#section-3.4.3

ですが、リスクと利益のバランスを取った結果、このような仕様になったのだとも言っています。たとえばクロスオリジンの GET リクエストを禁止にすると、ウェブのコア機能であるハイパーリンクが使えなくなるとしています。

> However, user agent implementors often balance these risks against the benefits of allowing the cross-origin interaction. For example, an HTML user agent that blocked cross-origin network requests would prevent its users from following hyperlinks, a core feature of the web.
> https://www.rfc-editor.org/rfc/rfc6454#section-3.4.3

GET リクエストは禁止せず、POST リクエストだけを禁止にするという選択肢もあったと思いますが、おそらく後方互換性の観点から諦めたのではないでしょうか。同一オリジンポリシーが策定された当時には、多くのクロスオリジンで POST をするウェブサイトがあり、それをデフォルトで禁止にしてしまうと影響が大きかったのだと推測します。

同じようなことは、こちらの記事にも書かれています。
CORS の仕様はなぜ複雑なのか - Zenn
https://zenn.dev/qnighy/articles/6ff23c47018380#%E7%B4%A0%E6%9C%B4%E3%81%AA%E4%BB%95%E6%A7%98

これが、現代でも csrf token などの防御機構を用いないと、安全なフォームのエンドポイントが作れないという状況が残っている一因だと思います。

### 同一オリジンポリシーをもっと詳しく

ここで紹介したのは、あくまで CSRF から見た同一オリジンポリシーの世界です。同一オリジンポリシーの他の側面について学びたい人は、以下の資料をおすすめします。

同一オリジンポリシー - ウェブセキュリティ | MDN
https://developer.mozilla.org/ja/docs/Web/Security/Same-origin_policy

『Web ブラウザセキュリティ ― Web アプリケーションの安全性を支える仕組みを整理する』米内貴志 著
https://www.lambdanote.com/collections/wbs

また、同一オリジンポリシーによる制限を緩和したい場合もあると思います。そのために作られた、CORS という仕組みがあります。CORS はこの後の章で軽く出てくるので、その記述とそこの参考資料を参照してください。
