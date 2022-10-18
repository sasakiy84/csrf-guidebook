# SPA での運用について考える
この節では、SPA (Single Page Application)での運用について考えます。
SPA とは簡単に言えば、初回のローディング時にコンテンツを全て読み込み、あとは JavaScript による DOM 操作でページ遷移を行っているように見せかけるというものです。React や Vue といったライブラリが有名です。

## SPA における論点
CSRF 対策において SPA を分けて考えなければならない理由は二つあります。
一つ目は、従来のサーバー側でページごとに HTML を生成して返すという方式で使えていた、csrf token をフォームや meta タグに埋め込む方法が使えないからです。
二つ目は、フロントとバックエンドの処理を別アプリケーションで行うことが多く、バックエンド API とフロントのページが別のドメインで運用される例があるためです。この方式にすると、同一オリジンポリシーの制限を受けます。

なお、一つ目の csrf token をフォームに埋め込むことができないという理由は関係がありません。SPA はページ遷移を JavaScript によって行うので、HTML のフォーム送信機能をデフォルトのまま使うことはないためです。

## 考えられる対策
まず考えたいのは、meta タグに埋め込むことができないという点です。csrf token の受け渡しができなくなってしまうためです。
これに対しては、たとえば、csrf token を発行する別のエンドポイントを作成するという方法が考えられます。同一オリジンポリシーに抵触しない限り、安全に csrf token を受け渡すことができますし、CORS を使って適切に同一オリジンポリシーを緩和すればクロスオリジンでも問題ありません。

## Cookie を使わないという方法
一方で、視点を変えて別の方法で CSRF 対策を行うこともできます。それは、そもそも session の認証に cookie を使わないという方法です。
そもそも CSRF の危険性は、cookie が見境なく送信されてしまうことから来ていました。攻撃者のサイトからリクエストを送った場合でも、sessionId などのトークンが送信され、認証を回避されてしまうという危険性だったのです。
では、sessionId を保管し受け渡す場所を、cookie ではなく HTTP ヘッダにするとどうでしょうか？ 
HTTP ヘッダに値を追加するときは、明示的にその値を指定しなければいけません。そのため、同じように CSRF 攻撃をしようと思ったら、攻撃者は sessionId をきちんと知る必要があります。sessionId を攻撃者が知ることができたとしたら、それはもう CSRF 脆弱性というレベルではないので、このシナリオは無視していいでしょう。

### cookie を捨てられる理由
では、なぜ今まで cookie を使っていたのでしょうか？
その理由のひとつは、まさに cookie が全てのリクエストに紐づくという点にあります。GET では POST でも関係なく自動でリクエストに紐づくという特徴は、サーバー側で HTML を生成するような構成のウェブアプリケーションでは必須でした。ヘッダにトークンを含める方法では、a タグによる GET リクエストのときなどにトークンが紐づきません。そのため、ログイン済みのユーザー用のページというものが機能しなくなります。

一方で SPA は、ページ遷移のような処理をクライアント側で行います。そして、必要なデータのみを JavaScript を使って、API 側に取得するという方式をとります。そのため、「自動で送信される」という cookie の機能が必要なくなったのです。

## Authentication ヘッダ
上記のような方策の一例として、Authentication ヘッダを用いたトークンの受け渡しがあります。
たとえば、JWT (Json Web Token)と呼ばれる構造体をエンコードしてトークンとして受け渡す仕組みでは、トークンの受け渡し場所として Authentication ヘッダがよく使われます。この JWT のなかに sessionId を含めれば、cookie で sessionId を受け渡していたときとほとんど同じことができるわけです。
一方で、cookie を使わないとなると、新たに考えなければならないこともでてきます。それは、sessionId の保管場所です。cookie には、`httpOnly`という JavaScript からのアクセスを禁止するオプションがあり、XSS の脆弱性があっても即座に sessionId が流出するわけではありませんでした。一方で、JWT を使うような方法は、必ず JavaScript からのアクセスを許可する必要があるため、cookie に保存する方法と比べると XSS に対する防御力が低いということになります。XSS に限らずとも、サプライチェーンアタックのように、使っているフロントのライブラリに悪意のあるコードが入っていると、sessionId が抜き取られてしまうリスクもあります。