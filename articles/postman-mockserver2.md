---
title: "PostmanのモックサーバーにAPIキー認証をつけてプライベートで公開する"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["postman", "API", ]
published: false
---

この記事は、[Postman Advent Calendar 2024](https://qiita.com/advent-calendar/2024/postman) 20日目の記事です。

[株式会社dott](https://thedott.io/)でバックエンドエンジニアをしているとみぎと申します。

前回の記事では、Postmanのモックサーバーの基本的な使い方を[PostmanのMockServerを使ってみる](https://zenn.dev/miggi/articles/postman-mockserver)の記事で紹介しました。

今回はその応用編ということで、タイトルの通り、作成したモックサーバーにAPIキー認証をつけてプライベートで公開する方法を紹介します。

# TL;DR

* PostmanのモックサーバーにAPIキー認証をつけることで、モックサーバーをプライベートで公開することができる
* APIキーの発行は[アカウント設定](https://dott-sc.postman.co/settings/me/api-keys) から行う
* モックサーバーの設定画面で「モックサーバーをプライベートにする」にチェックを付けて、APIキー認証を有効にする
* モックサーバーのリクエスト時はヘッダーに `x-api-key` を追加してリクエストを行う


# APIキーの発行

まず最初にAPI認証に必要なAPIキーを発行します。

## アカウント設定のAPIKeysにアクセス

* APIキーの発行はPostmanのアカウント設定から行うため、以下のリンクにアクセスします。
    [account settings](https://dott-sc.postman.co/settings/me/api-keys)

* アカウント設定にアクセスをしたら、サイドメニューの「API Keys」をクリックし、以下のAPI Keysの画面を表示します
![](/images/postman-mockserver2/account_setting_1.png)

## APIキーを発行

* API Keysの画面にアクセスをしたら、右上の「Generate API Key」をクリックします

    ![](/images/postman-mockserver2/account_setting_2.png)

* クリックをすると、API Keyの名前を入力するウィンドウが表示されるので、名前を入力して「Generate API Key」をクリックします。

    ![](/images/postman-mockserver2/account_setting_3.png)

* すると、API Keyが発行され、以下のような画面が表示されますので、
API Keyをコピーしておきます。

    ![](/images/postman-mockserver2/account_setting_4.png)

* ウィンドウを閉じて、API Keysの画面に戻ると、発行したAPI Keyが表示されていることが確認できます。

    ![](/images/postman-mockserver2/account_setting_5.png)

API Keyの発行作業は以上です。

# モックサーバーの設定を更新

では、ここからが本題。
モックサーバーの設定を更新します。

:::message
以降はモックサーバーを既に作成していることを前提に進めます。

まだモックサーバーを作成していない場合は、
[PostmanのMockServerを使ってみる](https://zenn.dev/miggi/articles/postman-mockserver)の記事を参考にしてください。
:::

## モックサーバーのプライベート設定を有効化

* Postmanを起動後、作成したモックサーバーを表示したら
右上にある「設定を編集」ボタンをクリックし、モックサーバーの設定画面を表示します。

    ![](/images/postman-mockserver2/enable_private_setting_1.png)

* 設定画面を開いたら、画面中央にある「モックサーバーをプライベートにする」にチェックを入れます。
* チェックを入れたら「モックサーバーを更新」をクリックして設定を保存します。

    ![](/images/postman-mockserver2/enable_private_setting_2.png)

これで、モックサーバーのプライベート設定が有効化されます。

# リクエスト時の認証設定の追加

最後にモックサーバーの動作チェックをしてみましょう

## 認証設定なしの場合

:::message
細かいリクエストの設定については省略しますが、
以下は、モックサーバーをプライベートにする前には200のステータスを返却していたAPIの例になります。
:::

以下のモックサーバーのAPIはプライベート設定前は200のレスポンスを返していましたが、
プライベート設定後は認証設定が無いと401エラーを返すようになりました。

![](/images/postman-mockserver2/request_with_api_key_1.png)

## APIキー認証を追加した場合

401エラーが返ってくることを確認できたので、次にリクエストに認証設定を追加していきます。

* リクエストの「認可」のタブを選択し、Auth Typeを「API キー」に変更します。
* キーには `x-api-key` を入力し、Valueには先ほど発行したAPI Keyを入力します。
* 上記の設定を行ったら、再度リクエストを送信してみます。
* APIキーを含む認証設定があっていれば、200のステータスが返ってくることを確認できます。

    ![](/images/postman-mockserver2/request_with_api_key_2.png)

以上が、PostmanのモックサーバーにAPIキー認証をつけてプライベートで公開する方法になります。

# 参考サイト

* [Postman - Generate a Postman API key](https://learning.postman.com/docs/developer/postman-api/authentication/)
* [Postman - Set up mock servers](https://learning.postman.com/docs/developer/postman-api/authentication/)
