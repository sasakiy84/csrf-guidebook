# CSRF の位置づけ

この節では、CSRF がどのような脆弱性として認識されているのか、その位置づけを見ていきます。

## セキュリティフレームワーク・ガイドライン

### CWE

英語：https://cwe.mitre.org/data/definitions/352.html#Common_Consequences
日本語：https://jvndb.jvn.jp/ja/cwe/CWE-352.html

CVE において、CSRF は "Insufficient Verification of Data Authenticity" (不適切な入力確認) に分類されるとしています。
また、XSS (クロスサイトスクリプティング) により CSRF が発生する可能性にも言及されています。

### 安全なウェブサイトの作り方

https://www.ipa.go.jp/security/vuln/websecurity/ug65p900000196e2-att/000017316.pdf#page=32

『安全なウェブサイトの作り方』は、IPA が公開する資料です。
「IPA が届出を受けた脆弱性関連情報を基に、届出件数の多かった脆弱性や攻撃による影響度が大きい脆弱性を取り上げ、ウェブサイト開発者や運営者が適切なセキュリティを考慮したウェブサイトを作成」できるようにすることを目的としています。
https://www.ipa.go.jp/security/vuln/websecurity.html

この資料では、ウェブアプリケーションに関わる 11 種類の主要な脆弱性が挙げられているのですが、そのなかの一つとして「CSRF」が入っています。
そこでは、発生しうる脅威として以下の例が挙げられています。

- ログイン後の利用者のみが利用可能なサービスの悪用
- ログイン後の利用者のみが編集可能な情報の改ざん、新規登録

また、根本的解決として、以下の手法が挙げられています

- 処理を実行するページを POST メソッドでアクセスするようにし、その「hidden パラメータ」に秘密情報が挿入されるよう、前のページを自動生成して、実行ページではその値が正しい場合のみ処理を実行する。
- 処理を実行する直前のページで再度パスワードの入力を求め、実行ページでは、再度入力されたパスワードが正しい場合のみ処理を実行する。
- Referer が正しいリンク元かを確認し、正しい場合のみ処理を実行する。

### MITRE ATT&CK

https://attack.mitre.org/

MITRE ATT&CK 　(Adversarial Tactics, Techniques, and Common Knowledge) は、「脆弱性を悪用した実際の攻撃を戦術と技術または手法の観点で分類したナレッジベース」です。
https://www.intellilink.co.jp/column/security/2020/060200.aspx

ここでは、直接 CSRF に言及されてはいないですが、CSRF で発生するような事象が攻撃の全体のどこに位置づけられるかを調べることができます。
たとえば、"Defense Evasion (防御回避) > Use Alternate Authentication Material (代替された認証情報の使用) > Web Session Cookie" という項目があります。
そこでは、Web Session Cookie のテクニックを使うことで、"an adversary can access sensitive information, read email, or perform actions that the victim account has permissions to perform." (攻撃者は、機密情報へのアクセスやメールの閲覧、被害者のアカウントで実行可能な操作などができる) とされています。
https://attack.mitre.org/techniques/T1550/004/

CSRF は、攻撃者が cookie を入手できるわけではないので、なんらかの情報を閲覧される危険はありませんが、副作用のある操作はされてしまいます。
ここでは、CSRF が防御の回避、とくにパスワード認証や二段階認証の回避などに使われることが確認できました。

また、"Execution > User Execution > Malicious Link" では、ユーザーが悪意のあるリンクをクリックすることによって、ユーザーのクライアント上で悪意のあるコードを実行させる方法が示されています。
このような悪意のあるリンクを踏ませるというテクニックは、CSRF を実行させるためのテクニックとして使われることが多いです。
このような攻撃手法への対策としては、"User Training" などが、検知手法としては、ネットワークトラフィックの監視と解析などが挙げられています
https://attack.mitre.org/techniques/T1204/001/
