# この記事で紹介すること

次の2つの内容を説明します．

1. LINE Bot でユーザーが送信した画像を取得する
2. Slack Bots で画像を投稿する

サーバー は heroku を使用します．

# できあがるとこんな感じ

[f:id:kuri_megane:20191102010121g:plain]

---

<b>目次</b>

[:contents]

---

<!-- more -->

# LINE & Slack 連携基礎編

この記事には次の記事の続きです．
LINE と Slack の連携について詳しく知りたいときは，これらの記事を参考にしてください．

1. [アカウント準備編](https://kuri-megane.hatenablog.jp/entry/2019/10/28/190000)
2. [LINE -> Slack の連携編](https://kuri-megane.hatenablog.jp/entry/2019/10/29/190000)

## Slack Bots のパーミッション追加

Slack で画像を投稿する場合，https://api.slack.com/apps より次の操作が必要です．

[f:id:kuri_megane:20191101232033p:plain]

Permissionsを選択し，

[f:id:kuri_megane:20191101232131p:plain]

Scopes より次のパーミッションを追加します．

* files:write:user
* chat:write:user
* chat:write:bot


# ソースコード

以下で紹介するソースコードはGithubで公開しています．

[https://github.com/kuri-megane/line-slack-connector-line2slack/tree/release/blog:embed:cite]


## ディレクトリ構成

次のようになっています．

```
line2slack
├── Procfile
├── main.py
├── requirements.txt
└── runtime.txt
```

## 環境構築

pip や pipenv などで次を使えるよう準備してください．

```text
Flask
line-bot-sdk
freeze
slackweb
```

以下は ubuntu/debian, pip3 の例を紹介します．

```bash
$ sudo apt install python3-pip
$ pip3 install Flask
$ pip3 install line-bot-sdk
$ pip3 install freeze
$ pip3 install slackweb
```

## 処理の概要

LINEで送信された画像を取得し，Slackに投稿する処理は次のとおりです．

```python
@handler.add(MessageEvent, message=ImageMessage)
def handle_image_message(event):
    """
    Image Message の処理
    """

    # トーク情報の取得
    user_id, user_name, msg_type, room_id = get_event_info(event)

    # LINEで送信された画像の取得
    message_id = event.message.id
    message_content = line_bot_api.get_message_content(message_id)
    img = message_content.content

    # slack側に投稿するメッセージの加工
    send_msg = "[bot-line] {user_name}さんが画像を送信しました．\n".format(user_name=user_name) \
               + "---\n" \
               + "送信元: {msg_type} ( {room_id} )\n".format(msg_type=msg_type, room_id=room_id) \
               + "送信者: {user_name} ( {user_id} )".format(user_name=user_name, user_id=user_id)

    file_name = "send_image_{message_id}".format(message_id=message_id)

    # 画像送信
    files = {'file': img}
    param = {
        'token': USER_OATH,
        'channels': POST_CHANEL_ID,
        'filename': file_name,
        'initial_comment': send_msg,
        'title': file_name
    }
    response = requests.post(url="https://slack.com/api/files.upload", params=param, files=files)

```

USER_OATH は，Slack の Bots OAuth トークンです．

POST_CHANEL_ID は投稿したい Slack チャンネルのIDです．

get_event_info() で送信されたトークの情報を取得しています．

```python
def get_event_info(event):
    """
    トーク情報の取得
    :param event: LINE メッセージイベント
    :return: ユーザID, ユーザ表示名, 送信元トークルームの種別, ルームID
    :rtype: str, str, str, str
    """

    # LINEユーザー名の取得
    user_id = event.source.user_id
    try:
        user_name = line_bot_api.get_profile(user_id).display_name
    except LineBotApiError as e:
        user_name = "Unknown"

    # トーク情報の取得
    if event.source.type == "user":
        msg_type = "個別"
        room_id = None
        return user_id, user_name, msg_type, room_id

    if event.source.type == "group":
        msg_type = "グループ"
        room_id = event.source.group_id
        return user_id, user_name, msg_type, room_id

    if event.source.type == "room":
        msg_type = "複数トーク"
        room_id = event.source.room_id
        return user_id, user_name, msg_type, room_id
```

トークの種類は，

* Botとの1対1トークの「個別」
* グループトーク
* グループではない複数人でのトークグループを「複数トーク」

としています．

## ソースコードすべて

- main.py 

```python
import os

import requests
import slackweb
from flask import Flask, request, abort
from linebot import LineBotApi, WebhookHandler
from linebot.exceptions import InvalidSignatureError, LineBotApiError
from linebot.models import MessageEvent, TextMessage, ImageMessage

app = Flask(__name__)

# 認証情報の取得
CHANNEL_ACCESS_TOKEN = os.environ["LINE_CHANNEL_ACCESS_TOKEN"]
CHANNEL_SECRET = os.environ["LINE_CHANNEL_SECRET"]
WEB_HOOK_LINKS = os.environ["SLACK_WEB_HOOKS_URL"]
BOT_OAUTH = os.environ["SLACK_BOT_OAUTH"]
POST_CHANEL_ID = os.environ["SLACK_POST_CHANEL_ID"]

line_bot_api = LineBotApi(CHANNEL_ACCESS_TOKEN)
handler = WebhookHandler(CHANNEL_SECRET)


@app.route("/callback", methods=['POST'])
def callback():
    # get X-Line-Signature header value
    signature = request.headers['X-Line-Signature']

    # get request body as text
    body = request.get_data(as_text=True)
    app.logger.info("Request body: " + body)

    # handle web hook body
    try:
        handler.handle(body, signature)
    except InvalidSignatureError:
        abort(400)

    return 'OK'


def get_event_info(event):
    """
    トーク情報の取得
    :param event: LINE メッセージイベント
    :return: ユーザID, ユーザ表示名, 送信元トークルームの種別, ルームID
    :rtype: str, str, str, str
    """

    # LINEユーザー名の取得
    user_id = event.source.user_id
    try:
        user_name = line_bot_api.get_profile(user_id).display_name
    except LineBotApiError as e:
        user_name = "Unknown"

    # トーク情報の取得
    if event.source.type == "user":
        msg_type = "個別"
        room_id = None
        return user_id, user_name, msg_type, room_id

    if event.source.type == "group":
        msg_type = "グループ"
        room_id = event.source.group_id
        return user_id, user_name, msg_type, room_id

    if event.source.type == "room":
        msg_type = "複数トーク"
        room_id = event.source.room_id
        return user_id, user_name, msg_type, room_id


@handler.add(MessageEvent, message=TextMessage)
def handle_text_message(event):
    """
    Text Message の処理
    """
    
    slack_info = slackweb.Slack(url=WEB_HOOK_LINKS)

    # トーク情報の取得
    user_id, user_name, msg_type, room_id = get_event_info(event)

    # slack側に投稿するメッセージの加工
    send_msg = "[bot-line] {user_name}さん\n".format(user_name=user_name) \
               + "{msg}\n".format(msg=event.message.text) \
               + "---\n" \
               + "送信元: {msg_type} ( {room_id} )\n".format(msg_type=msg_type, room_id=room_id) \
               + "送信者: {user_name} ( {user_id} )".format(user_name=user_name, user_id=user_id)

    # メッセージの送信
    slack_info.notify(text=send_msg)


@handler.add(MessageEvent, message=ImageMessage)
def handle_image_message(event):
    """
    Image Message の処理
    """

    # トーク情報の取得
    user_id, user_name, msg_type, room_id = get_event_info(event)

    # LINEで送信された画像の取得
    message_id = event.message.id
    message_content = line_bot_api.get_message_content(message_id)
    img = message_content.content

    # slack側に投稿するメッセージの加工
    send_msg = "[bot-line] {user_name}さんが画像を送信しました．\n".format(user_name=user_name) \
               + "---\n" \
               + "送信元: {msg_type} ( {room_id} )\n".format(msg_type=msg_type, room_id=room_id) \
               + "送信者: {user_name} ( {user_id} )".format(user_name=user_name, user_id=user_id)

    file_name = "send_image_{message_id}".format(message_id=message_id)

    # 画像送信
    files = {'file': img}
    param = {
        'token': BOT_OAUTH,
        'channels': POST_CHANEL_ID,
        'filename': file_name,
        'initial_comment': send_msg,
        'title': file_name
    }
    response = requests.post(url="https://slack.com/api/files.upload", params=param, files=files)



if __name__ == "__main__":
    port = int(os.getenv("PORT", 5000))
    app.run(host="0.0.0.0", port=port)

```

認証情報の取得では，環境変数に [1.アカウント準備編](https://kuri-megane.hatenablog.jp/entry/2019/10/28/190000) でメモした値を設定する必要があります．

herokuで動かすために更にコードを追加します．

- Procfile

``` 
web: python main.py
```

- requirements.txt

```
Flask
line-bot-sdk
freeze
slackweb
```

- runtime.txt

```
python-3.6.8
```

# デプロイ

## Gitに追加する

heroku にデプロイするため，Gitレポジトリを作成しコミットしておきます．
以下はその例です．

```bash
$ git init
$ git add . 
$ git commit -m "first commit"
```

## heroku にデプロイ

heroku にアプリを作成し，環境変数を設定します．

heroku CLI を使うので，アカウントの準備ができていない場合は [1.アカウント準備編](https://kuri-megane.hatenablog.jp/entry/2019/10/28/190000) を参考にしてください．

ここでは，アプリ名を line2slack としてデプロイしています．

```bash
$ heroku create line2slack
Creating ⬢ line2slack... done
https://line2slack.herokuapp.com/ | https://git.heroku.com/line2slack.git

$ git push heroku master
$ heroku config:set LINE_CHANNEL_ACCESS_TOKEN=[LINE Bot のアクセストークン (ロングターム)] --app line2slack
$ heroku config:set LINE_CHANNEL_SECRET=[LINE Bot の Channel Secret] --app line2slack
$ heroku config:set SLACK_WEB_HOOKS_URL=[Slack Bots の Incomming Webhook URL] --app line2slack

$ heroku config:set SLACK_BOT_OAUTH=[Slack Bots の Bots OAuth Token] --app line2slack
$ heroku config:set SLACK_POST_CHANEL_ID=[投稿したい Slack のチャンネルID] --app line2slack
```


# 動け！

ここまで来ると，LINEに投稿したメッセージが動くと思います．

[f:id:kuri_megane:20191102010121g:plain]

# トラブルシューティング

うまく動かないときは，

```bash
$ heroku logs --tail --app line2slack
```

で確認してみてください．

また，アプリのURL (上の例では https://line2slack.herokuapp.com/ ) にアクセスできるかも確認してみてください．


# 参考記事

[https://qiita.com/_kazuya/items/78961aef30c192a8bd96:embed:cite]

[https://developers.line.biz/ja/docs/messaging-api/:embed:cite]

[https://qiita.com/namutaka/items/233a83100c94af033575:embed:cite]

