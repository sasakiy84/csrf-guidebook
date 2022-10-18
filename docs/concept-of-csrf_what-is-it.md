# CSRF とはなにか

CSRF は、正式名称を Cross-Site Request Forgery といい、Web に関する脆弱性、あるいはその脆弱性をついた攻撃手法のことです。略語は、シーサーフと読むらしいです。
Web に関する攻撃手法の中では XSS (Cross Site Scripting) などと並んで有名なものです。

ここでは、脆弱性を説明するときによく参照される CWE の説明を元に、CSRF とはなにかを説明します。

## CWE とは

CWE（Common Weakness Enumeration）とは、共通脆弱性タイプ一覧と訳され、「ソフトウェアにおけるセキュリティ上の弱点（脆弱性）の種類を識別するための共通の基準を目指して」策定されたものです。

https://www.ipa.go.jp/security/vuln/CWE.html

## CSRF とはなにか

### CWE による簡潔な説明

まず、CWE の簡潔な説明を読んでみます。

英語版

> The web application does not, or can not, sufficiently verify whether a well-formed, valid, consistent request was intentionally provided by the user who submitted the request.
> https://cwe.mitre.org/data/definitions/352.html

日本語版

> 本脆弱性が存在する Web アプリケーションは、フォーマットに沿った、妥当で一貫性のあるリクエストが、送信したユーザの意図通りに渡されたものかを十分に検証しない、あるいは検証が不可能です。
> https://jvndb.jvn.jp/ja/cwe/CWE-352.html

一つずつかみ砕いて説明します。英語の授業のようになりますが、公式っぽい堅苦しくて読みにくそうなページを読んでみるという経験をしてみようという趣旨で、丁寧に解説します。

#### 主部

まず、主語は「The web application」であり、動詞は「does not / can not verify」、目的語は「whether a request was provided by the user」です。つまり、概要としては、「ウェブアプリケーションが、リクエストがユーザーが渡したものかどうかを検証しない / できない」脆弱性が CSRF に該当します。

#### 修飾部

修飾語で補足している部分を見ていきましょう。

目的語の部分に補足があります。

まずどのようなリクエストかについての補足です。
該当部分では、「well-formed, valid, consistent」と述べられています。日本語では、「フォーマットに沿った、妥当で一貫性のある」と訳されています。

これは、入力として正当な意味があるリクエストである、という補足をしているのだと思います。つまり、バッファオーバーフローのような、予期しない不正な値のリクエストなどではない、ということです。

次に、どのように送信されたかについての補足です。該当部分は「intentionally provided by the user who submitted the request」で、日本語では「送信したユーザの意図通りに渡されたもの」と訳されています。

これは、リクエストの補足と合わせて考えれば意味は明確です。

「入力として正当な意味があるリクエスト」が「正当なユーザーから」送信されただけであれば、なにも問題はないわけです。

しかし、その「攻撃者が正当なユーザーを装って」送信したリクエストの場合、そのリクエストに紐づく処理を実行してしまうと何らかの問題が発生してしまうでしょう。

リクエストが本当に正当なユーザーが意図して送信したものか、という点が、CSRF で問題にされているわけです。

### CWE による詳細な説明

詳細説明の項目も確認しておきましょう

英語版

> When a web server is designed to receive a request from a client without any mechanism for verifying that it was intentionally sent, then it might be possible for an attacker to trick a client into making an unintentional request to the web server which will be treated as an authentic request. This can be done via a URL, image load, XMLHttpRequest, etc. and can result in exposure of data or unintended code execution.
> https://cwe.mitre.org/data/definitions/352.html#Extended_Description

日本語版

> Web サーバがリクエストを検証せずに受け取るよう設計されている場合、攻撃者がクライアントを騙し、意図しないリクエストを Web サーバに送信させる可能性があります。その場合、Web サーバはそのリクエストを正規のものとして取り扱います。
> この攻撃は URL、画像の読み込み、XMLHttpRequest 等を介して行われ、データの漏えいや意図しないコードの実行を招く可能性があります。
> https://jvndb.jvn.jp/ja/cwe/CWE-352.html

ここで注目したいのは、「it might be possible for an attacker to trick a client into making an unintentional request to the web server」の部分です。日本語では「攻撃者がクライアントを騙し、意図しないリクエストを Web サーバに送信させる可能性があります」と訳されています。

新しい情報として、クライアントを騙すという条件が追加されています。ここでいうクライアントとは、被害者のクライアントのことです。これは、SQL Injection のように攻撃者の PC からリクエストを送るのではなく、一般人である被害者がリクエストを送るということを表しています。

また、CWE では、攻撃手法についても言及されています。攻撃手法については、後の章で紹介します。

## 具体例

具体的なシナリオに沿って考えてみましょう。

多くのログインが必要なウェブサイトには、パスワードリセット機能があります。実際にパスワードをリセットするときには、「パスワードをリセットするよ～」というリクエストをサーバーに送信するわけです。

このとき、CSRF の脆弱性がある実装方法だった場合、つまり、「パスワード変更のリクエストが正規ユーザーからのリクエストかどうかを検証できないような実装方法だった場合」のことを考えましょう。

アプリケーションは、正規ユーザーかどうか検証できないため、攻撃者のリクエストを受け取るとパスワード変更を実行してしまいます。つまり、リクエスト内に含まれる変更後のパスワードを攻撃者が指定できる場合、そのアカウントの乗っ取りが成功してしまうのです。

もう一つ具体例を出します。

掲示板のようなウェブサービスがあったとします。この掲示板は会員登録が不要で、誰でも送信が可能です。ユーザーの識別には IP アドレスを使っていました。
このとき、攻撃者が CSRF の脆弱性を利用して、被害者の PC から不適切な発言を投稿するリクエストを送信したとします。
この場合、送信元の IP アドレスは被害者のものですから、ほかのユーザーからみると被害者が不適切発言を投稿したように見えます。
このような事例が実際に発生したのが、「パソコン遠隔操作事件」です。CSRF の脆弱性があるウェブサイトの不適切投稿がされた事件です。警察が投稿者の識別に IP アドレスを使った結果、誤認逮捕につながりました。

https://ja.wikipedia.org/wiki/%E3%83%91%E3%82%BD%E3%82%B3%E3%83%B3%E9%81%A0%E9%9A%94%E6%93%8D%E4%BD%9C%E4%BA%8B%E4%BB%B6

なお、次の章では、例に挙げたような掲示板をつくり、不適切な投稿を送るという実験をやってみます。

## CSRF の被害

脆弱性がある場合、気になるのはどのような被害が発生するか、という点です。
CWE には次のように書かれています

英語版

> Gain Privileges or Assume Identity; Bypass Protection Mechanism; Read Application Data; Modify Application Data; DoS: Crash, Exit, or Restart.
> The consequences will vary depending on the nature of the functionality that is vulnerable to CSRF. An attacker could effectively perform any operations as the victim. If the victim is an administrator or privileged user, the consequences may include obtaining complete control over the web application - deleting or stealing data, uninstalling the product, or using it to launch other attacks against all of the product's users. Because the attacker has the identity of the victim, the scope of CSRF is limited only by the victim's privileges.
> https://cwe.mitre.org/data/definitions/352.html#Common_Consequences

日本語版

> 権限の取得やなりすまし、防御メカニズムの回避、アプリケーションデータの読み取り。重大性は CSRF の脆弱性が存在する機能の性質によって変わります。攻撃者は事実上、被害者と同じように操作を行うことが可能です。被害者が管理者あるいは権限のあるユーザだった場合には、web アプリケーションの完全なコントロール(データの削除や窃取、製品のアンインストールや製品の全てのユーザに対する攻撃の基盤としての利用等)を与えることになります。攻撃者は被害者の識別を持っているため、CSRF の及ぶ範囲は被害者の持つ権限内に制限されます。
> https://jvndb.jvn.jp/ja/cwe/CWE-352.html

重要なのは、脆弱性が存在する機能の性質によって重大性が変わるということです。

たとえば、具体例で見たのは、パスワードリセットの機能が悪用された例でした。この場合、変更できるパスワードが一般ユーザーのものだけであれば、被害は一人のユーザーにとどまりますが、管理者ユーザーのパスワードが変更された場合は、より大きな被害になるでしょう。クレジットカードが使えるかどうかなどでも重大度は変わりそうです。
例では、掲示板に不適切投稿される例もみました。こちらはパスワードの変更に比べれば重大度は低いといえそうです。もちろん脆弱性には変わりませんが。

逆に、パスワード変更の方式がもう少し慎重なものだった場合を考えてみましょう。たとえば、パスワードが変更される前にメールでユーザーに確認が行く場合は、CSRF の脆弱性はありつつも、そこまで重大性は大きくないでしょう。
