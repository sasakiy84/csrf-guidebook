# トークンを使って防ぐ part 2

この節では、前章で説明した csrf token を用いた防御方法の変化形と、その危険性を説明します。

## cookie の二重送信

この防御手法は、攻撃者が cookie を盗聴することができない、という前提を用いたものです。
従来の csrf token を用いた防御手法は、サーバー側のアプリケーションと、クライアント側のフォームにランダムな文字列を埋め込み、フォーム送信時に両者を比較することで検証を行っていました。
これは、以下の前提から成り立っています。

- トークンが改ざんされていないか検証するために 2 つ以上の情報源が必要
- トークンの情報源のうち、少なくとも一つは攻撃者が偽造、改ざんできない
- トークンを攻撃者は盗み見ることができない

たとえば、クライアント側にトークンを送信するだけで、サーバー側に保存しないと、クライアントから送信されたトークンが本物かどうかを確かめることができません。
あるいは、トークンの情報を二つとも攻撃者が捏造できると、攻撃者が偽造したトークンと攻撃者が偽造したトークンを検証するようなことになってしまいます。
また、トークンを攻撃者が盗み見ることができると、攻撃者は正規のリクエストであるかのように偽装できてしまいます。

一方で、従来の方法では、csrf token をサーバー側にも保存しなければいけないことで、HTTP のステートレス性が壊されてしまいます。（ログイン認証でもやってるじゃん、ということは見ないふりをします）
まあ、フォームを検証するためだけに DB の機能を追加するのは少しイヤです。

そこで、クライアント側に二つの情報源を保存しつつ、そのうち一つは攻撃者が偽造、改ざんできないことを保証しつつ、盗聴できないようにしつつ、クライアント側にすべてを保存する方法が提案されました。

## フォームと cookie に保存する

この方法が成り立つ理由を、具体的にどこに保存するかという情報と一緒に説明していきます。
まず、「トークンが改ざんされていないか検証するために 2 つ以上の情報源が必要」という点は満たされています。
次に、「トークンの情報源のうち、少なくとも一つは攻撃者が偽造、改ざんできない」という点です。これは、cookie が満たしています。cookie の設定オプションに`httponly`という項目があり、これを`true`にすると、JavaScript で cookie を操作できなくなります。すなわち、改ざんできません。
また、cookie には `domain` 属性と呼ばれる、どのドメインに紐づけるのかを指定できるオプションがありますが、違うドメインに紐づけた cookie を設定することはできません。つまり、トークンを偽造することができません。

https://developer.mozilla.org/ja/docs/Web/API/Document/cookie#%E6%96%B0%E3%81%97%E3%81%84%E3%82%AF%E3%83%83%E3%82%AD%E3%83%BC%E3%82%92%E6%9B%B8%E3%81%8D%E8%BE%BC%E3%82%80

最後に「トークンを攻撃者は盗み見ることができない」という点についてです。まずは cookie に保存するトークンですが、先ほどいった通り`httponly`というオプションをつければ、JavaScript から操作できません。つまり、ブラウザのことだけを考えれば盗聴はされません。
次に、form に保存するトークンについてですが、こちらは SOP により JavaScript で取得することができません。

以上により、この方法はうまく動きそうです。

## 危険性を理解する

ところが、この方法はいくつかリスクがあります。

### クッキーモンスターバグによるリスク

たとえば、クッキーモンスターバグと呼ばれるものです。
少し前に cookie の `domain`属性を紹介しました。この`domain`属性は、異なるドメインには設定できないと書きましたが、例えば次のような場合は同じドメインとみなされます。`aaa.vpc-service.com`における`vpc-service.com`。
つまり、`aaa.vpc-service.com`にリクエストを送るときには、`aaa.vpc-service.com`に紐づけられている cookie だけでなく、`vpc-service.com`に紐づけられた cookie も送信されます。また、`vpc-service.com`に紐づけた cookie を自由に設定することが可能です。

クッキーモンスターバグを使うと以下のような攻撃が可能です。
攻撃者は、攻撃対象のサイトと後半部分が共通しているドメインを取得し、その共通部分に紐づけたトークンをセットします。たとえば、`aaa.vpc-service.com`が攻撃対象のドメインだとすると、`bbb.vpc-service.com`のドメインを取得し、そのドメインに置いたサイトを使って`vpc-service.com`の cookie にトークンをセットします。そうして、フォームにも同じトークンを埋め込めば、サーバー側でのトークンの検証をパスすることができます。

クッキーモンスターバグは、[eTLD+1](https://webtan.impress.co.jp/g/etld1)配下のサブドメインを配布するようなサービス（たとえば、Render や heroku といったホスティングサービス）で発生します。ただし、Render や heroku でこのような挙動になるのは、バグというよりは仕様というべきかもしれません。

### その他のリスク

クッキーモンスターバグ以外にも、ほかの脆弱性が存在した場合に攻撃につながる可能性が大きいようです。古めの記事ですが、徳丸先生のブログにクッキーモンスターバグも含めまとまっています。なお、挙動確認はしていません。
https://blog.tokumaru.org/2018/11/csrf_26.html

### \_\_Host- prefix ヘッダ

なお、クッキーモンスターバグに関して言えば、`__Host-` prefix という機能を使えば対策できるでしょう。これは、cookie の名前の先頭につけると、`domain`属性を無効化するものです。
https://datatracker.ietf.org/doc/html/draft-west-cookie-prefixes-05#section-3.2

MDN のサイトを見る限りブラウザでサポートされていて、少し探した限りでは github で使われているのを見つけたので、実用化されているのではないでしょうか。
https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Set-Cookie#%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%83%BC%E3%81%AE%E4%BA%92%E6%8F%9B%E6%80%A7

![](img/guard-csrf-using-token-part2_20221013003847.png)
