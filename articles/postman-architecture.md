---
title: "ぼくのかんがえたさいきょうのPostmanかんきょう"
emoji: "📮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Postman, API]
published: true
---

[株式会社dott](https://thedott.io/)でバックエンドエンジニアをしているとみぎと申します

今回は、プロジェクトでPostmanを使う中でさいきょうのPostmanかんきょうを作れたので、その内容をまとめました

# TL;DR

* Environmentを使ってURLやトークンを管理する
* CollectionのVariables・Authを使ってコレクションに依存する値の管理をする
* Testsのスクリプトを使ってEnvironmentの値更新を自動化する


# Environmentを使う

PostmanのEnvironmentはいわゆる環境変数のような立ち位置で、コレクション全体で参照が可能なものです
Environmentの変数は以下の目的でよく使っています

* APIのホストURL(パスを除いたもの)の指定
* アクセストークンの保持
* プルリクのデプロイバージョン番号の指定

## 環境毎にEnvironmentを分ける

ぼくの場合、主に4つの環境でEnvironmentを分けて、URLやトークンを切り替えできるようにしています

### LOCAL

ローカル環境で立ち上げたAPIにリクエストする用のEnvironment
![](https://storage.googleapis.com/zenn-user-upload/06b657b0255d-20220618.png)

|Variable|用途|
|---|---|
|SYS_ACCESS_TOKEN|管理者用APIのトークン用|
|CLI_ACCESS_TOKEN|ユーザーAPIのトークン用|
|BASE_URL|APIのホストURL|

### DEV

開発環境にデプロイされたAPIにリクエストをする環境
主にPull requestのテストで使う事が多いので、デプロイURLのRevisionを指定できるような変数構成にしています
![](https://storage.googleapis.com/zenn-user-upload/dfbeb13380a6-20220618.png)

|Variable|用途|
|---|---|
|SYS_ACCESS_TOKEN|管理者用APIのトークン用|
|CLI_ACCESS_TOKEN|ユーザーAPIのトークン用||
|BASE_URL|APIのホストURLだが、Pull Requestのデプロイバージョンを指定できるよう、URL内に`REVISIONの変数` を参照している|
|REVISION|Pull RequestのRevision指定用。BASE_URLで参照される|
|DEV_URL|プレーンな開発環境のURL用(たまに使うので定義している)|

ぼくのぷろじぇくとでは、PRのデプロイが完了するとこの形のURLが発行される
![](https://storage.googleapis.com/zenn-user-upload/49135bed2a6c-20220618.png)

これを利用して、`https://` と`-dot-the-great-postman...` の間のバージョン番号をコピーして REVISIONに貼り付けるだけで向き先を変えられる簡単仕様にしてみた。
(URL全体を毎回Environmentにコピペする方式もありますが、URL全文を毎回更新するのが面倒なので、この方法を使っています)

:::message
EnvironmentのValue内でも変数が定義できるのがポイント
:::

### STG

ステージング環境のAPIにリクエストする用のEnvironment
構成はLOCALと同じ
![](https://storage.googleapis.com/zenn-user-upload/cb6ce62384c8-20220618.png)

|Variable|用途|
|---|---|
|SYS_ACCESS_TOKEN|管理者用APIのトークン用|
|CLI_ACCESS_TOKEN|ユーザーAPIのトークン用|
|BASE_URL|APIのホストURL|

### PROD

本番環境用のEnvironment
構成はLOCAL, STGと同じ
![](https://storage.googleapis.com/zenn-user-upload/806c070214ff-20220618.png)

|Variable|用途|
|---|---|
|SYS_ACCESS_TOKEN|管理者用APIのトークン用|
|CLI_ACCESS_TOKEN|ユーザーAPIのトークン用|
|BASE_URL|APIのホストURL|

:::message
※基本的に本番環境は触りたくないので、PRODはほぼ使っていません
:::


# Collectionの設定を活用する

## Variablesを使う

Collection変数はEnvironmentと異なり、Collection(APIのフォルダようなもの)配下のリクエストのみが参照できる変数となっている
ぼくの場合、APIのサービス毎にCollectionを作り、そこにAPIに至るまでのパス
を定義することで、RequestのURLを記載する手間を省いています

### 前置き

ぼくのぷろじぇくとのでは、管理者用のAPI(SYS)とユーザー用のAPI(CLI)の大きく分けて2種類のサービスでわかれていて、URLで記載すると

SYSのAPIパス: `https:///the-great-postman-stg.appspot.com/_ah/api/sys/v1/`
CLIのAPIパス: `https:///the-great-postman-stg.appspot.com/_ah/api/cli/v1/`

このようにAPIのパス以前のURLが微妙に異なります

↑このURLをPostmanのRequest毎に記入するのはコピペするとしてもちょっと面倒ですよね
仮に前述のEnvironmentの変数を使った場合は、

SYSのAPIパス: `{{BASE_URL}}/_ah/api/sys/v1/`
CLIのAPIパス: `{{BASE_URL}}/_ah/api/cli/v1/`

このように少しスマートになりましたが、もうちょっとスマートにしたいところです
そのためにCollectionのVariableを使ってこのもやもやを解消します

### 設定方法

#### Collectionの設定

Collection毎に `SERVICE_PATH` という変数をVariablesに設定します
![](https://storage.googleapis.com/zenn-user-upload/e5e5f4337b5b-20220618.png)

cliコレクションの設定
![](https://storage.googleapis.com/zenn-user-upload/5d7473b95f72-20220618.png)

sysコレクションの設定
![](https://storage.googleapis.com/zenn-user-upload/b05b5f158f62-20220618.png)



#### Requestの設定

リクエストのURLを以下のように変更する
({{SERVICE_PATH}}とAPIのパスの間を{{SERVICE_PATH}}に置き換える)

```
{{SERVICE_PATH}}{{REQUEST_PATH}}/[APIのパス]
```

![](https://storage.googleapis.com/zenn-user-upload/f2f7921f6ca4-20220618.png)

#### 設定後の値

実際の値(CLIのコレクション配下)
![](https://storage.googleapis.com/zenn-user-upload/3f4707d3d45f-20220618.png)

実際の値(SYSのコレクション配下)
![](https://storage.googleapis.com/zenn-user-upload/93588f55e964-20220618.png) 


はい、これでAPIのパス以前の文字列は共通化することができたので、リクエストを作成する時は

```
{{BASE_URL}}{{SERVICE_PATH}}/リクエストするAPIのパス
``` 

という、APIのパスだけ考慮すればいい形になりました
見た目もスッキリ！ 


## Authorization を使う

Collectionには認証情報の設定もでき、これもVariables同様Collection配下のリクエストに対して認証情報を一括で付与することが出来ます

CLIではHeaderに `X-Access-Token` のkeyにEnvironmentのユーザー用トークン `CLI_ACCESS_TOKEN` の値を参照してセットしています
![](https://storage.googleapis.com/zenn-user-upload/d3cab33c9fb9-20220618.png)

SYSではHeaderに `X-Access-Token` のkeyにEnvironmentのユーザー用トークン `SYS_ACCESS_TOKEN` の値を参照してセットしています
![](https://storage.googleapis.com/zenn-user-upload/cd343971a628-20220618.png)

CLI配下のAPIのHeaderを見てみると、`X-Access-Token`  が自動追加されていることがわかります
![](https://storage.googleapis.com/zenn-user-upload/8ee9fcf85c69-20220618.png)

この構成によって、一度ログインを行ってアクセストークンをEnvironmentに保存してしまえば、あとは他のAPIは自由に叩くことができるようになります


# リクエストのTestsでレスポンスの値を取得する

Postmanではリクエストの前後にスクリプトを差し込む事ができ、スクリプトを使ってEnvironmentの値を取得・更新することができます

## スクリプトの種類

![](https://storage.googleapis.com/zenn-user-upload/4fb384ce1733-20220618.png)

* Pre-request: リクエストの送信前に書かれたスクリプトを実行する
* Tests: リクエスト完了後に書かれたスクリプトを実行する

## Testsにアクセストークンを更新するスクリプトを追加する

Environmentでトークンを管理するだけでも多少便利にはなりますが、
毎回ログインAPIを叩く→レスポンスのトークンをコピー→Environmentのトークンの値を更新
というような操作を毎回するのは面倒ですよね

なので、Testsの機能を使ってこの操作を自動化しちゃいます

### スクリプト
これをログイン用のリクエストのTestsに記述するだけ

```javascript
pm.test("set x_access_token", function () {
    const jsonData = pm.response.json();
    pm.environment.set("SYS_X_ACCESS_TOKEN", jsonData.x_access_token);
});
```

![](https://storage.googleapis.com/zenn-user-upload/375f9e07b283-20220618.png)

### 解説

今回のログインAPIは以下のJsonを返してきます

```json
{
    "emails": [],
    "id": "1234567890",
    "name": "postman",
    "login_id": "postman",
    "is_administrator": true,
    "x_access_token": "54dc23ed5350b18f622214abf58998c5617e9a1656cc19673b46ef97a0f1b60c"
}
```

ここで必要なのは `x_access_token` なので、この値を取得してEnvirionmentの `SYS_ACCESS_TOKEN`を更新する処理をかけばよい


```javascript
// pm.testはPostmanの関数
pm.test("set x_access_token", function () {
    // pm.response.json() でレスポンスのJSONを取得できる
    const jsonData = pm.response.json();

    // Environmentの変数を指定して更新する
    // pm.environment.set("Environmentの変数名", セットする値)
    pm.environment.set("SYS_X_ACCESS_TOKEN", jsonData.x_access_token);
});
```

これでさいきょうのPostman環境のできあがり！

# 余談

PostmanのWorkspaceはプロジェクトごとに作成すると、コレクションのリストやEnvironmentがごちゃつかないのでいいですよ〜