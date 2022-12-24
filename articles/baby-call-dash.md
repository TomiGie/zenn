---
title: "AWS IoTボタンで陣痛のお知らせを手軽にできる陣痛Dashボタンを作ってみた"
emoji: "🐣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [AWSIoT, AWS]
published: true
---

# はじめに

この記事は、[身の回りの困りごとを楽しく解決！ by Works Human Intelligence](https://qiita.com/advent-calendar/2022/works-hi-1) Advent Calendar 2022 の25日目の記事です。


[株式会社dott](https://thedott.io)でバックエンドエンジニアをしているとみぎと申します。
この度、11月に第2子が産まれたのですが、第1子と同様にAWS IoTボタン(以降Dashボタン)を使って陣痛の発生を通知する仕組み、通称「陣痛Dashボタン」を作成し、実際に運用して便利だったので、この知見を残しておきたいと思い、こちらのAdvent Calendarに参加しました。


# 作ったもの

今回私が作成したものは、Dashボタンを押すとSlackの指定のチャンネルとLINEの通知グループに陣痛発生のメッセージを送信するというものです。

![](https://storage.googleapis.com/zenn-user-upload/91a366973170-20221225.png =250x)
*LINEのメッセージ画面*

![](https://storage.googleapis.com/zenn-user-upload/79c65775e0bc-20221225.png =250x)
*Slackのメッセージ画面*

ソースコードは以下にあります

https://github.com/TomiGie/LaborPainDash

:::message
## おすすめポイント

この陣痛Dashボタンのいいところは、`ボタンを押すだけで済む` ため、記録がすごく楽であるという所です。
アプリストアには、より便利な前駆陣痛・陣痛間隔を計算・記録するためのアプリがあり、機能面を比較するとこれらのアプリの方に軍配が上がるでしょう。

しかし、いざ前駆陣痛や陣痛が始まったとして、陣痛記録アプリを起動してタイマーを開始して記録するという操作を冷静にできるでしょうか？
夫がスタンバイをして計測する分には可能だと思いますが、妊婦さん本人が痛みの中でこの操作をする余裕って意外とないことも多いと思います。

そんなときに役に立つのがこの陣痛Dashボタンなんです。

**痛みが来たらボタンを押すだけ**
この1アクションだけでSlackとLINEに発生時間が通知され、グループに参加しているメンバーにも共有できますし、発生時間のログも残すことができます。

実際に使ってみましたが、結構便利でした。

:::

# 構成

* 利用サービス: AWS Lambda、AWS IoT 1-Click
* 言語: Go
* メッセージの受け取り: Slack, LINE

# 解説

## イベントトリガーの設定

Dashボタンのイベントトリガーの取得はAWS IoT 1-Clickで設定をしています
設定はAWSのコンソールもしくは[アプリ](https://docs.aws.amazon.com/ja_jp/iot-1-click/latest/developerguide/1click-mobile-app.html)から可能で、私はアプリから登録しました。

ここでリージョンの設定や、Dashボタンのデバイス情報の登録・Wi-Fiの設定を行ったり、ボタンを押したときのアクションを登録します。
今回、このアクションの部分を指定のLambda関数を呼び出すようにしています

![](https://storage.googleapis.com/zenn-user-upload/77707d43e174-20221225.png)
*設定した状態はこんな感じ*

:::message
ここでは先にイベントトリガーの設定を紹介していますが、アクションの登録の際は先にLambda関数を作成・デプロイしておく必要があります
:::

![](https://storage.googleapis.com/zenn-user-upload/7416969a1441-20221225.png)
*予めデプロイした関数*


## メッセージの送信

今回はGoを使って、SlackとLINEへ通知を飛ばすシンプルな処理を書きました
Slackは通知専用のチャンネルに対してWebhookを利用してメッセージを送信しています
LINEの送信先に関しては、なるべく実装を簡単にしたかったので個人宛てではなく、LINEのメッセージグループに対して[LINE Notify](https://notify-bot.line.me/ja/)経由でメッセージを送信する形を取りました

:::details 処理全体

```go:hello.go

package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
	"strings"
	"time"

	"github.com/aws/aws-lambda-go/lambda"
)

type MyEvent struct {
	Name string `json:"What is your name?"`
}

type DashEvent struct {
	DeviceEvent struct {
		ButtonClicked struct {
			ClickType    string
			ReportedTime string
		}
	}
}

type MyResponse struct {
	Message string `json:"Answer:"`
}

type SlackMessage struct {
	Text string `json:"text"`
}

func main() {
	lambda.Start(hello)
}

func hello(event DashEvent) (MyResponse, error) {
	url := "INCOMING_WEBHOOK_URL"
	SendSlackMessage(url, event.DeviceEvent.ButtonClicked.ClickType)
	return MyResponse{Message: fmt.Sprintf("Hello %s!!", "event.Name")}, nil
}

func SendSlackMessage(url string, clickType string) error {

	jstZone := time.FixedZone("Asia/Tokyo", 9*60*60)
	jst := time.Now().UTC().In(jstZone)
	timeFormat := "15時04分"
	nowDateTimeString := fmt.Sprint(jst.Format(timeFormat))

	msg := ""

	if clickType == "SINGLE" {
		msg = "前駆陣痛開始"
	} else {
		msg = "前駆陣痛終了"
	}

	message := SlackMessage{Text: fmt.Sprintf("*【%v】* `%v`", nowDateTimeString, msg)}

	SendLine(fmt.Sprintf("*【%v】* `%v`", nowDateTimeString, msg))

	requestParam, _ := json.Marshal(message)

	req, err := http.NewRequest(
		"POST",
		url,
		bytes.NewBuffer(requestParam),
	)
	if err != nil {
		return err
	}

	req.Header.Set("Content-Type", "application/json")
	client := &http.Client{}
	resp, err := client.Do(req)

	if err != nil {
		return err
	}

	defer resp.Body.Close()
	return nil
}

func SendLine(message string) {

	url := "https://notify-api.line.me/api/notify"
	method := "POST"

	payload := strings.NewReader("message=" + message)

	client := &http.Client{}
	req, err := http.NewRequest(method, url, payload)

	if err != nil {
		fmt.Println(err)
		return
	}
	req.Header.Add("Authorization", "Bearer {LINE_ACCESS_TOKEN}")
	req.Header.Add("Content-Type", "application/x-www-form-urlencoded")

	res, err := client.Do(req)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer res.Body.Close()

	body, err := ioutil.ReadAll(res.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(string(body))
}

// build command
// GOOS=linux GOARCH=amd64 go build -o hello
// zip handler.zip ./hello
```

:::



## LINE Notifyの設定

LINEグループへの通知は[LINE Notify](https://notify-bot.line.me/ja/)を利用します。
[![](https://storage.googleapis.com/zenn-user-upload/dfaf6befb5c9-20221225.png)
](https://notify-bot.line.me/ja/)

### ログイン
まず、LINE Notifyに自分のLINEアカウントでログインします。
ログインすると「LINE Notify」というLINEアカウントが友達に追加されます。
![](https://storage.googleapis.com/zenn-user-upload/525db158ee16-20221225.png)

### 通知用のLINEグループを作成
通知したい家族とLINE NotifyのLINEアカウントを入れたLINEグループを作成します。

### アクセストークンを作成
LINE Notifyのサイトの[マイページ](https://notify-bot.line.me/my/)にアクセスし、「トークンを発行する」(Generate token)ボタンよりアクセストークンを発行します。

![](https://storage.googleapis.com/zenn-user-upload/fe3b4f31ee0d-20221225.png)

ここで設定するトークン名はLINE Notifyからの通知の際に表示されます

## Slack(Incoming Webhooks)の設定

Slackのチャンネルの通知には[Incoming Webhooks](https://api.slack.com/messaging/webhooks)を利用しました。
これを使うことで、0からSlackbotを登録する手間がなくなりますし、生成されたWebhookのURLに必要なリクエストを送るだけで、予め登録されていたチャンネルにメッセージを送信することができます。

# 実装方法

ここからは実装するために必要な手順やデプロイの方法などを紹介していきます

## 作業の流れ

:::details 1. Slack(Incoming Webhooks)の準備
### 作業内容
   1. Slackチャンネルを作成(既存のチャンネルでも可)
   2. [Incoming Webhooks](https://api.slack.com/messaging/webhooks)を登録し、Webhookイベントで受け取ったメッセージを送信するチャンネルを設定する
   3. Webhook URLをメモっておく

:::

:::details 2. LINEグループの準備
### 作業内容
   1. 通知を受け取るためのLINEグループを作成する
   2. アクセストークンを発行する
   3. アクセストークンをメモっておく
:::

:::details 3. Lambdaの関数を登録する
### 作業内容
1. GoのテンプレートコードをDL
   1. [GitHub](https://github.com/TomiGie/LaborPainDash)
2. テンプレートコード(hello.go)の一部を書き換え
   1. SlackのIncoming WebhooksのURL
   2. LINEのアクセストークン
3. 書き換えたコードをzipにする
4. Lambdaのコンソールにアクセスし、関数を作成
5. 作成した関数に、2で書き出したhello.zipをアップロード

### 書き換え箇所

#### Slack

41行目 `{INCOMING_WEBHOOK_URL}` の部分に、取得したIncoming WebhookのURLを上書きします

#### LINE

102行目 `{LINE_ACCESS_TOKEN}` の部分に、LINE Notifyで発行したアクセストークンを上書きします

### zipにする方法

カレントディレクトリを、今回のディレクトリにした状態で以下のコマンドを実行します

```
bash build.sh
```
これを実行すると、同階層に `hello.zip` という名前のzipファイルが作成されます

もし、うまく行かなかった場合は以下のコマンドを1行ずつ実行してみてください

```
GOOS=linux GOARCH=amd64 go build -o hello
zip handler.zip ./hello
```

:::

:::details 4. AWS IoT 1-Clickでデバイス・アクションの登録
### 作業内容
   1. アプリもしくはコンソールからデバイス情報を登録
      1. デバイス情報
         1. シリアルコード
         2. Wi-Fi接続設定
      2. ボタン押下時のアクション
         1. デプロイしたLambda関数の指定
:::


# あとがき

残念ながらAWS IoTボタンは販売終了してしまったため、同じデバイスを新しく手に入れることは難しくなってしまいましたが、コード自体はほかのIoTデバイスからイベントを受け取った後の処理に流用できるかと思い、今回紹介させていただきました。

サンプルコードはまだまだリファクタリングの余地はあるかとは思いますが、私自身、育休を取得して絶賛子育て中でなかなか時間がとれないため、よしなに書き換えて頂ければと思います。

この知識が少しでもお役に立てれば幸いです。
それでは、よいクリスマスを!