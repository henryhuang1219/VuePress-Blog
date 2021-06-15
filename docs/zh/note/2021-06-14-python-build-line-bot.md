---
title: Python - 建立Line Bot聊天機器人
date: 2021-06-14
tags:
 - Backend
 - Python
 - Line Bot
categories:
 - Note
---

::: tip
由於Line Bot一般的Messaging API在電腦版上無法正常顯示，改為請使用者至手機上查看之替代文字呈現，因此這次Line Bot開發選擇可自訂排版且能在電腦版顯示之Flex Message。
應用範例選擇使用大學專題所製作的WhatToEat APP，將少數與使用者關係較深且常用到之功能套用至Line Bot上。
:::

<!-- more -->

## 建立Line官方帳號
進入[LINE Developers](https://developers.line.biz/console/)，點擊Create按鈕新增Provider
![Create Line Provider](https://i.imgur.com/pV1fyVq.png)

進入Provider後，點擊Create a new channel，會跳出選擇channel type的視窗
![Create New Channel](https://i.imgur.com/1lSO1T6.png)

選擇Messaging API選項，將資料填寫完畢後點擊Create，便完成了當前步驟。


## 設定Django環境並與Line官方帳號連結
::: tip
以下將不會對Django建置多做敘述，如有需要請參考[官方文件](https://docs.djangoproject.com/en/2.1/intro/tutorial01/)
:::

安裝所需的套件`line-bot-sdk`
```bash
 pip3 install line-bot-sdk
```

進入剛新建的Channel中，點擊Messaging API
![Messaging API](https://i.imgur.com/stwHQaQ.png)

滑到底部後會看到Channel access token，點擊Issue(由於我已創建過token，所以顯示Reissue)
![Channel access token](https://i.imgur.com/nRyFDeW.png)

Channel Secret在Basic setting裡便可找到
![Basic settings](https://i.imgur.com/KMzSf1q.png)

取到Access token和Secret後，在views.py中加入Line Bot SDK所需之初始化宣告
```python
from linebot import LineBotApi, WebhookHandler

line_bot_api = LineBotApi('YOUR_CHANNEL_ACCESS_LONG_TOKEN')
web_hook_handler = WebhookHandler(LINE['YOUR_CHANNEL_SECRET'])
```

新增WebHook驗證功能
```python
from django.http import HttpResponse
from linebot.exceptions import LineBotApiError

def bot(request):
    try:
        # signature
        signature = request.headers['X-Line-Signature']
        body = request.body.decode()
        web_hook_handler.handle(body, signature)
    except LineBotApiError as e:
        print(str(e))
    return HttpResponse("Success")
```

至[LINE Official Account Manager](https://manager.line.biz/account/)選擇官方帳號，點擊聊天，選擇設定回應模式
![LOAM Dialog](https://i.imgur.com/BwBA5VC.png)

設定結果如下
![Response Configuration](https://i.imgur.com/tGOHjV6.png)

設定完後便能啟動Server，將產生的網址填入[LINE Developers](https://developers.line.biz/console/) Messaging API中的webhook URL並點擊Verify，如成功便會跳出Success視窗
![Success Dialog](https://i.imgur.com/7luO5nO.png)
::: tip
Django使用虛擬環境執行且無固定IP之問題，推薦使用[ngrok](https://ngrok.com)
:::

## 建立Flex Message模板
這邊選擇使用官方提供的[Flex Message Simulator](https://developers.line.biz/flex-simulator/)，中間可以進行元件的新增修改刪除，右邊則是可調整之元件屬性，而Showcase裡有官方所提供的數個模板可供套用，也可點擊View as JSON直接修改或取得模版之Json檔
![Flex Message Template](https://i.imgur.com/CBvsthB.png)


## 撰寫Line Bot API及結合Flex Message
::: tip
以下皆以WhatToEat之功能為範例來做說明
:::

目前只有用到其中幾種情況監聽一般訊息的`MessageEvent`、監聽追蹤的`FollowEvent`、監聽取消追蹤的`UnFollowEvent`及監聽按鈕回傳值的`PostbackEvent`

只要是出現在聊天室的訊息都會觸發到監聽一般訊息的`MessageEvent`，寫一個鸚鵡機器人來當作測試，確認能夠順利取得聊天室訊息
```python
@web_hook_handler.add(MessageEvent, message=TextMessage)
def echo(event):
    try:
        line_bot_api.reply_message(event.reply_token,
                TextSendMessage(text=event.message.text))
    except LineBotApiError as e:
        print(str(e))
    return HttpResponse("OK")
```

監聽追蹤的`FollowEvent`是在點擊追蹤或取消封鎖時會觸發，監聽取消追蹤的`UnFollowEvent`則是在封鎖時觸發，兩者內容其實差不多，可撰寫歡迎訊息來測試是否成功
```python
@web_hook_handler.add(FollowEvent)
def handle_follow(event):
    try:
        # send welcome message
        line_bot_api.reply_message(
            event.reply_token,
            TextSendMessage(
                text="Hello，您好，歡迎加入「飢不擇食」\U001000A4 \n\n"
                "目前功能尚未開發完成，期待能帶給使用者們更佳的操作體驗\U00100090"))
    except LineBotApiError as e:
        print(str(e))
    return HttpResponse("OK")


@web_hook_handler.add(UnfollowEvent)
def handle_un_follow(event):
    # do something
    
    return HttpResponse("OK")

```

再來，就是比較需要注意的監聽按鈕回傳值`PostbackEvent`，在這邊可先利用剛剛的鸚鵡機器人傳送一個有button且type為postback的Flex Message至聊天室中，以下直接取用Showcase中的模板進行測試

postback button中的action可自訂，但欄位不可作更改，alt_text欄位為聊天室之預覽訊息，將程式碼複製進前面所寫的`echo`功能中，並使之觸發
```python
contents = {
  "type": "bubble",
  "hero": {
    "type": "image",
    "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/01_1_cafe.png",
    "size": "full",
    "aspectRatio": "20:12",
    "aspectMode": "cover",
    "action": {
      "type": "uri",
      "uri": "http://linecorp.com/"
    }
  },
  "body": {
    "type": "box",
    "layout": "vertical",
    "contents": [
      {
        "type": "text",
        "text": "Store Name",
        "weight": "bold",
        "size": "xl"
      },
      {
        "type": "box",
        "layout": "baseline",
        "contents": [
          {
            "type": "icon",
            "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png"
          },
          {
            "type": "text",
            "text": "Star",
            "flex": 0,
            "margin": "md",
            "size": "sm",
            "color": "#999999"
          }
        ],
        "margin": "md"
      },
      {
        "type": "box",
        "layout": "vertical",
        "margin": "lg",
        "spacing": "sm",
        "contents": [
          {
            "type": "box",
            "layout": "baseline",
            "spacing": "sm",
            "contents": [
              {
                "type": "text",
                "text": "Place",
                "color": "#aaaaaa",
                "size": "sm",
                "flex": 1
              },
              {
                "type": "text",
                "text": "Address",
                "wrap": true,
                "color": "#666666",
                "size": "sm",
                "flex": 5
              }
            ]
          },
          {
            "type": "box",
            "layout": "baseline",
            "spacing": "sm",
            "contents": [
              {
                "type": "text",
                "text": "Time",
                "color": "#aaaaaa",
                "size": "sm",
                "flex": 1
              },
              {
                "type": "text",
                "text": "24 hour",
                "wrap": true,
                "color": "#666666",
                "size": "sm",
                "flex": 5
              }
            ]
          }
        ]
      }
    ]
  },
  "footer": {
    "type": "box",
    "layout": "vertical",
    "spacing": "sm",
    "contents": [
      {
        "type": "button",
        "style": "link",
        "height": "sm",
        "action": {
          "type": "postback",
          "label": "label",
          "data": "YOUR_POST_BACK_DATA",
          "displayText": "displayText"
        }
      }
    ],
    "flex": 0
  }
}

line_bot_api.reply_message(event.reply_token,
    FlexSendMessage(alt_text='YOUR_ALT_TEXT', contents=contents))
```
::: tip
如不清楚如何修改，可將以下Json值複製進[Flex Message Simulator](https://developers.line.biz/flex-simulator/)查看
:::

點擊Flex Message具有postback type的按鈕，就會觸發監聽按鈕回傳值`PostbackEvent`，可在功能裡加入判斷回傳資料之if或是直接將資料print出來在terminal，確認資料是否正確
```python
@web_hook_handler.add(PostbackEvent)
def post_back(event):
    try:
        if event.postback.data == 'YOUR_POST_BACK_DATA':
            print('Success')
    except Exception as e:
        print(str(e))
    return HttpResponse("OK")
```


## 總結
此篇文章只有大略介紹了一下常用的Line Bot API，可再自行作延伸，還有例如群發等API，詳細的資料可至[line-bot-sdk-python](https://github.com/line/line-bot-sdk-python)，裡面有各個API及資料內容的介紹，感謝您的閱讀。