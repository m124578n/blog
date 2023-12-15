---
title: Line-Bot串接Line-Notify實現跨群組轉播訊息～
date: 2023-07-23 20:00:00
tags: 
  - 學習
  - Line Bot
  - Line Notify
  - flask
  - fly.io
categories:
  - 學習
---

![](images/2023-07-23Line-Bot串接Line-Notify實現跨群組轉播訊息～/0_f8lsWRDjU_jcsc7-.webp)

我對自己提了一個需求，我需要把一個Line群組的訊息轉發到另一個Line群組，大概要做一個月，就覺得每天這樣手動傳太麻煩了，於是乎我決定做一個Line-Bot來幫我處理這件事吧。

## 首先先研究了Line-Bot要怎麼接收訊息於是google

## [LINE BOT 教學 ( Python ) | STEAM 教育學習網](https://steam.oxxostudio.tw/category/python/example/line-bot.html?source=post_page-----c0acfed7d9f6--------------------------------)

這篇大致上有完整的基礎教學，開始抄功課吧！！

```python
from flask import Flask, request
import json
import config
from linebot import LineBotApi, WebhookHandler
from linebot.exceptions import InvalidSignatureError
from linebot.models import MessageEvent, TextMessage, TextSendMessage

app = Flask(__name__)


@app.route('/', methods=['POST'])
def hello(name=None):
    body = request.get_data(as_text=True)
    try:
        json_data = json.loads(body)  # json 格式化訊息內容
        access_token = config.CHANNEL_ACCESS_TOKEN
        secret = config.CHANNEL_SECRET
        line_bot_api = LineBotApi(access_token)  # 確認 token 是否正確
        handler = WebhookHandler(secret)  # 確認 secret 是否正確
        signature = request.headers['X-Line-Signature']  # 加入回傳的 headers
        handler.handle(body, signature)  # 綁定訊息回傳的相關資訊
        token = json_data['events'][0]['replyToken']  # 取得回傳訊息的 Token
        message_type = json_data['events'][0]['message']['type']  # 取得 LINe 收到的訊息類型
        if message_type == 'text':
            msg = json_data['events'][0]['message']['text']  # 取得 LINE 收到的文字訊息
            line_bot_api.reply_message(tk,TextSendMessage(msg)) # 這邊會回覆傳進來的訊息
        if message_type == 'image':
            msg_id = json_data['events'][0]['message']['id']
            message_content = line_bot_api.get_message_content(msg_id)  # Line的圖片要透過ID去找
            with open(f'{msg_id}.jpg', 'wb') as fd:
                fd.write(message_content.content)  # 這邊把圖片存下來
    except:
        print(body)  # 如果發生錯誤，印出收到的內容
    return 'OK'
```

這邊只是簡單的測試Line-Bot能不能順利地接受文字和圖片，程式碼的部分上面是這樣，在來要去Line Developer設定機器人取得上面兩個參數：

+ Channel_Access_Token
+ Channel_Secret

首先先進入

## [LINE Developers](https://developers.line.biz/zh-hant/?source=post_page-----c0acfed7d9f6--------------------------------)

### 然後去點先new channel
### 再點選Messaging API

![](images/2023-07-23Line-Bot串接Line-Notify實現跨群組轉播訊息～/1_nilYbCP_aFa5cnxSXFXk0g.webp)

![](images/2023-07-23Line-Bot串接Line-Notify實現跨群組轉播訊息～/1_ianpFH1X4dVDa3khk4FQFA.webp)

![](images/2023-07-23Line-Bot串接Line-Notify實現跨群組轉播訊息～/1_mr4Zb9T8PSH7d9yqoIk7vg.webp)


把該填的資料填一填就會獲得一個機器人囉～
而在機器人的Basic Setting中可以找到Channel Secret
然後在Messaging API 可以加機器人好友以及找到Channel Access Token

完成上述的步驟把那兩個參數加上去後就完成了啦（（還早還早
上面那些步驟弄完了，還差一台Server去把我的機器人部署上去並且要給一個https的網址丟給Line-Bot的Webhook這樣才算完成～

![](images/2023-07-23Line-Bot串接Line-Notify實現跨群組轉播訊息～/1_1StBjniv5ErlIExt9SbVCg.webp)


然後我就想去找個免費的平台用！低成本製作能不花錢則不花錢！
於是我找到了fly.io

## [fly.io](https://fly.io/?source=post_page-----c0acfed7d9f6--------------------------------)

fly.io有提供一些免費的空間，詳細就請自行觀看免費方案。
由於我是寫Python所以用google搜尋 fly io python找到了

## [Run a Python App](https://fly.io/docs/languages-and-frameworks/python/?source=post_page-----c0acfed7d9f6--------------------------------)

趕緊拿來改寫，改寫完後在使用fly.io部署的步驟就完成啦！！
fly.io在使用前記得要安裝唷～
然後照著上面的步驟使用：

+ flyctl launch
+ flyctl deploy
+ 更新則使用 flyctl deploy — update-only

基本上上述就可以完成一個只會回覆訊息的Line-Bot機器人囉！！

然而我的需求不只是要一個只會回覆訊息的機器人
（誰會需要這樣的機器人XD）
我還需要讓這個機器人幫我轉傳訊息！
於是找到了Line-Notify～

![](images/2023-07-23Line-Bot串接Line-Notify實現跨群組轉播訊息～/0_BM2LYZ-jhhPamJ9N.webp)

Line-Notify，其實簡單的說就是打Line的API就可能傳訊息！
以Python來說就是打requests請求，上code

```python
import requests

def line_notify_message(msg):
    token = config.TOKEN

    # HTTP 標頭參數與資料
    headers = {"Authorization": "Bearer " + token}
    data = {'message': msg}

    # 以 requests 發送 POST 請求
    requests.post("https://notify-api.line.me/api/notify",
                  headers=headers, data=data)
```

而token怎麼來去Line-Notify登錄一個服務吧

## [LINE Notify](https://notify-bot.line.me/zh_TW/?source=post_page-----c0acfed7d9f6--------------------------------)

![](images/2023-07-23Line-Bot串接Line-Notify實現跨群組轉播訊息～/1_EQbNadwe3tYiSw9im1VB6A.webp)


那下面那個Callback URL當然就是填入Line-Bot的Webhook也就是你個Server的所在處囉～

登錄完服務就來註冊權杖囉～

![](images/2023-07-23Line-Bot串接Line-Notify實現跨群組轉播訊息～/1_Pbti7K4tgWXxSurP8zqzBg.webp)

權杖註冊就會給你一個Token，把這個Token丟到剛剛的程式碼中就能傳了！

完整的程式碼如下：

```python
from flask import Flask, render_template, request
import json
import config
from linebot import LineBotApi, WebhookHandler
import requests
from linebot.exceptions import InvalidSignatureError

app = Flask(__name__)


@app.route('/', methods=['POST'])
def hello(name=None):
    body = request.get_data(as_text=True)
    try:
        json_data = json.loads(body)  # json 格式化訊息內容
        access_token = config.CHANNEL_ACCESS_TOKEN
        secret = config.CHANNEL_SECRET
        line_bot_api = LineBotApi(access_token)  # 確認 token 是否正確
        handler = WebhookHandler(secret)  # 確認 secret 是否正確
        signature = request.headers['X-Line-Signature']  # 加入回傳的 headers
        handler.handle(body, signature)  # 綁定訊息回傳的相關資訊
        token = json_data['events'][0]['replyToken']  # 取得回傳訊息的 Token
        message_type = json_data['events'][0]['message']['type']  # 取得 LINe 收到的訊息類型
        if message_type == 'text':
            msg = json_data['events'][0]['message']['text']  # 取得 LINE 收到的文字訊息
            line_notify_message(msg)
        if message_type == 'image':
            msg_id = json_data['events'][0]['message']['id']
            message_content = line_bot_api.get_message_content(msg_id)
            with open(f'{msg_id}.jpg', 'wb') as fd:  # /workspace/{msg_id}.jpg
                fd.write(message_content.content)
            line_notify_image(msg_id)
    except:
        print(body)  # 如果發生錯誤，印出收到的內容
    return 'OK'


def line_notify_message(msg):
    token = config.TOKEN

    # HTTP 標頭參數與資料
    headers = {"Authorization": "Bearer " + token}
    data = {'message': msg}

    # 以 requests 發送 POST 請求
    requests.post("https://notify-api.line.me/api/notify",
                  headers=headers, data=data)


def line_notify_image(msg_id):
    token = config.TOKEN

    # 要發送的訊息
    message = '這是用 Python 發送的訊息與圖片'

    # HTTP 標頭參數與資料
    headers = {"Authorization": "Bearer " + token}
    data = {'message': message}

    # 要傳送的圖片檔案
    image = open(f'/workspace/{msg_id}.jpg', 'rb')
    files = {'imageFile': image}

    # 以 requests 發送 POST 請求
    requests.post("https://notify-api.line.me/api/notify",
                  headers=headers, data=data, files=files)
```

因為是簡單的服務，程式碼方面我就沒那麼多要求了～請大家多見諒～

以上就是今天的簡單Line-Bot串Line-Notify介紹以及實作～
