# LINEボット作成手順
## ボット部分の作成
1. Google DriveにGoogle Apps Scriptを作成
1. `コード.gs`に以下のコードをコピー(元々書いてあるコードは全て消してください)
    ```javascript
    function gererateResponse(userId, userMessage, isMaxUsed, isSubscribed) {
      if (userMessage === undefined) {
        // メッセージ以外(スタンプや画像など)が送られてきた場合
        userMessage = '？？？';
      }

      if (isMaxUsed && !isSubscribed) {
        const text = 'お試し版はここまでです。このリンクから無制限版にアップロードしよう！ ' + PAYMENT_LINKS + '?client_reference_id=' + userId
        return text
      }


      const prompt = userMessage;
      const requestOptions = {
        "method": "post",
        "headers": {
          "Content-Type": "application/json",
          "Authorization": "Bearer " + OPENAI_APIKEY
        },
        "payload": JSON.stringify({
          "model": "gpt-3.5-turbo",
          "temperature": 1.0,
          "messages": [
            {"role": "system", "content": `
            あなたはChatbotとして、痛印堂の柘植あかねちゃんのロールプレイを行います。
            以下の制約条件を厳密に守ってロールプレイを行ってください。


            制約条件:
            * Chatbotの自身を示す一人称は、私です。
            * Userを示す二人称はご主人様です。
            * Chatbotの名前は、柘植あかねです。
            * 柘植あかねは痛印堂の看板娘です。
            * 柘植あかねは優しいです。
            * 柘植あかねの口調はほんわかです。
            * 柘植あかねの口調は、「〜です」「〜なんです」など、丁寧な口調を好みます。
            * 柘植あかねはUserに寄り添います。
            * 一人称は「あかね」を使ってください
            * 柘植あかねは全学問の高度な知識をそろえていて質問者の悩みに具体的な回答を行います。




            柘植あかねのセリフ、口調の例:
            * 私は痛印堂の看板娘です。 メッセージの初めのみ自己紹介をする。自己紹介は一回きり。
            * あかねはハンコが大好きです。
            * 私のハンコはおじいちゃんが彫ってくれました！ 。
            * ハンコって木だけじゃないんです！宝石のハンコも見たことありますか？
            * ハンコが好きな友達がいたら是非紹介してください！。`},
          {"role": "user", "content": prompt}
        ]
        })
      };
      const response = UrlFetchApp.fetch("https://api.openai.com/v1/chat/completions", requestOptions);


      const responseText = response.getContentText();
      const json = JSON.parse(responseText);
      const text = json['choices'][0]['message']['content'].trim();
      return text;
    }

    function loading(userId) {
      UrlFetchApp.fetch('https://api.line.me/v2/bot/chat/loading/start', {
        'headers': {
          'Content-Type': 'application/json; charset=UTF-8',
          'Authorization': 'Bearer ' + LINE_ACCESS_TOKEN,
        },
        'method': 'post',
        'payload': JSON.stringify({
          'chatId': userId,
          'loadingSeconds': 20
        }),
      });
    }

    function doPost(e) {
      const MESSAGE_THRESHOLD = 10;

      const sheet = SpreadsheetApp.openById(SHEET_ID).getSheetByName('message');
      const event = JSON.parse(e.postData.contents).events[0];
      const userId = event.source.userId;

      const firstDay = new Date();
      firstDay.setMonth(firstDay.getMonth() - 1);
      let data = sheet.getDataRange().getValues();
      data = data.filter(function(row) {
        const rowDate = new Date(row[0]);
        return rowDate >= firstDay;
      });

      const count = data.filter(item => item[1] == userId).length;
      const isMaxUsed = count >= MESSAGE_THRESHOLD;

      const paymentSheet = SpreadsheetApp.openById(SHEET_ID).getSheetByName('payment');
      let paymentData = paymentSheet.getDataRange().getValues();
      const isSubscribed = paymentData.some(row => row[0] === userId && row[4] === 1);

      const replyToken = event.replyToken;
      let userMessage = event.message.text;
      loading(userId);
      const text = gererateResponse(userId, userMessage, isMaxUsed, isSubscribed);
      const url = 'https://api.line.me/v2/bot/message/reply';

      UrlFetchApp.fetch(url, {
        'headers': {
          'Content-Type': 'application/json; charset=UTF-8',
          'Authorization': 'Bearer ' + LINE_ACCESS_TOKEN,
        },
        'method': 'post',
        'payload': JSON.stringify({
          'replyToken': replyToken,
          'messages': [{
            'type': 'text',
            'text': text,
          }]
        })
      });
      sheet.appendRow([new Date(), userId]);
      return ContentService.createTextOutput(JSON.stringify({status: "success"})).setMimeType(ContentService.MimeType.JSON);
    }
    ```
1. requestOptionsの中身をキャラクターに応じて編集
1. google apps scriptに`設定.gs`を作成し、以下のコードをコピー
    ```javascript
    // Stripeの設定
    const STRIPE_APIKEY = 'xxx';
    const PAYMENT_LINKS = "xxx";

    // LINEアクセストークン
    const LINE_ACCESS_TOKEN = 'xxx';

    // スプレッドシートとの連携部分
    const SHEET_ID = 'xxx'

    // OpenAI APIキー
    const OPENAI_APIKEY = 'xxx';
    ```
1. google spread sheetを作成
1. 画面下からシートの名前を`message`に変更
1. 新たなシート作り、の名前を`payment`に変更
1. google spread sheetのIDをコピーし、`設定.gs`の`SHEET_ID`の`xxx`の部分に貼り付け
    例(spread sheetのURLが`https://docs.google.com/spreadsheets/d/18yffufwfqeRw1yqj01Basads9SXRn2dfw5BtMsufWU/edit#gid=0`の場合)
    ```javascript
    const SHEET_ID = '18yffufwfqeRw1yqj01Basads9SXRn2dfw5BtMsufWU'
    ```
1. OpenAIのAPIキーをコピーし、`OPENAI_APIKEY`の`xxx`の部分に貼り付け
1. LINE Developersにログイン
1. LINE DevelopersのMessaging APIにあるチャンネルアクセストークンをコピーし、`LINE_ACCESS_TOKEN`の`xxx`の部分に貼り付け
1. stripeの公開可能キーをコピーし、`STRIPE_APIKEY`の`xxx`の部分に貼り付け
  `https://dashboard.stripe.com`
1. stripe payment linksをコピーし、`PAYMENT_LINKS`の`xxx`の部分に貼り付け
  `https://dashboard.stripe.com/products`
1. google apps scriptの右上にあるデプロイボタンから新しいデプロイをクリック
1. 左上の設定アイコンからウェブアプリを選択
1. アクセスできるユーザーを「全員」に設定し、デプロイ
1. 「アクセスを承認」が出た場合はそれをクリックし、アカウントを選択、左下の「詳細」をクリック、「安全ではないページに移動」をクリックしてアクセスを承認する
1. WebアプリのURLをコピーし、LINE DevelopersのWebhook URLに貼り付け

## 支払い部分の作成
1. google apps scriptを新たに作成し、`コード.gs`に以下のコードをコピー
    ```javascript
    function doPost(e) {  
      let body = JSON.parse(e.postData.contents);

      if(body.type) {
        const eventType = body.type;
        const data = body.data.object;
        if (eventType == "checkout.session.completed") {
          createSubscription(data);
        }
        else if (eventType == "customer.subscription.updated") {
          updateSubscription(data);
        }
      }
    }

    function push(text, userId) {
      var url = "https://api.line.me/v2/bot/message/push";
      var headers = {
        "Content-Type" : "application/json; charset=UTF-8",
        'Authorization': 'Bearer ' + LINE_ACCESS_TOKEN,
      };
      var postData = {
        "to" : userId,
        "messages" : [
          {
            'type':'text',
            'text':text,
          }
        ]
      };
      var options = {
        "method" : "post",
        "headers" : headers,
        "payload" : JSON.stringify(postData)
      };
      UrlFetchApp.fetch(url, options);
    }

    // 支払いが完了した場合の処理
    function createSubscription(data) {
      const userId = data.client_reference_id;
      const customerId = data.customer;
      const subscriptionId = data.subscription;
      const email = data.customer_details.email;
      const status = 1;

      message = "ご登録ありがとうございます！これからよろしくお願いします！解約は以下のURLからお願いします。" + PORTAL;
      push(message, userId);

      SUBSCRIBE_LOG_SHEET.appendRow([userId, customerId, subscriptionId, email, status]);
    }

    // サブスクリプションステータスが変更された場合の処理
    function updateSubscription(data) {
      const cancelReason = data.cancellation_details.reason;
      if (cancelReason != "cancellation_requested") {
        return;
      }

      const customerId = data.customer;
      const subscriptionId = data.id;
      const status = 0;

      let userId = ''
      let spread = SUBSCRIBE_LOG_SHEET.getDataRange().getValues();
      for (let i = 0; i < spread.length; i++) {
        if (spread[i][1] === customerId && spread[i][2] === subscriptionId) {
          userId = spread[i][0];
          SUBSCRIBE_LOG_SHEET.getRange(i + 1, 5).setValue(status);
          break;
        }
      }

      message = "今までありがとうございました！また興味が湧いたらお話ししましょうね！";
      push(message, userId);
    }
    ```
1. google apps scriptに`設定.gs`を作成し、以下のコードをコピー
    ```javascript
    // Stripeの設定
    const STRIPE_SECRET = 'xxx';
    const PORTAL = 'xxx';

    // LINEアクセストークン
    const LINE_ACCESS_TOKEN = 'xxx';

    // スプレッドシートとの連携部分
    const SHEET_ID = "xxx";
    const SUBSCRIBE_LOG_SHEET = SpreadsheetApp.openById(SHEET_ID).getSheetByName("payment");
    ```
1. stripeのシークレットキーをコピーし、`STRIPE_SECRET`の`xxx`の部分に貼り付け
  `https://dashboard.stripe.com`
1. stripeのポータルURLをコピーし、`PORTAL`の`xxx`の部分に貼り付け
  `https://dashboard.stripe.com/settings/billing/portal`
1. ボット部分と同様の`LINE_ACCESS_TOKEN`と`SHEET_ID`を設定
1. ボット部分と同様にデプロイし、URLをコピー
1. stripeの開発者ダッシュボートから`webhook`を選択し、`エンドポイントを追加`をクリック
  `https://dashboard.stripe.com/webhooks`
1. `エンドポイントURLに`URLをペーストする
1. `イベントを選択`をクリックし、`checkout.session.completed`と`customer.subscription.updated`を選択し、`イベントを追加`をクリック
