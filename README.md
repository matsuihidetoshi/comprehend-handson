# Lambda + Comprehend で問合せした人の感情を判定するAPIを作る

こんにちは、スタテク開発部の松井です。  
近年、機械学習やAIに対する注目が集まっています。しかし、何らかの機械学習を活用した仕組みを1人で0から作るのはなかなかハードルが高く、機械学習で重要になってくるビッグデータの収集も大きな問題になってきます。  
AWSでは、こうした問題を解決してくれる、機械学習を誰でも簡単に活用できるサービスが多数提供されています。  
今回はその中でも、機械学習を使用してテキスト内でインサイトや関係性を検出する自然言語処理 (NLP) サービス[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を活用して、 **問合せした人の感情を判定してくれるAPI** を作ってみようと思います。

- 機械学習、AI活用といっても、どこから手をつけたらいいか分からない
- 日常の生活や業務でどの様に役に立てれば良いか分からない

という方も多くいらっしゃると思いますが、[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を活用したこちらのハンズオンが一助となれば幸いです。

## 前提環境

今回のハンズオンは下記の環境にて検証しています。 ※AWSマネジメントコンソールにアクセスして操作できればPCやOSは問いません

- macOS Big Sur version 11.2.1

## 今回のアーキテクチャ

今回のハンズオンで作成するシステムの構成図です。

![architecture](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/architecture.png)

- [Amazon API Gateway](https://aws.amazon.com/jp/api-gateway/)がリクエストを受け取り、[AWS Lambda](https://aws.amazon.com/jp/lambda/)へプロキシします。
- [AWS Lambda](https://aws.amazon.com/jp/lambda/)では[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を使って送信内容を解析し、文章から感情を判定します。
- 上記の判定内容をメールのテキストに追記して、[Amazon SES](https://aws.amazon.com/jp/ses/)へ管理者へのメール送信をリクエストします。


## 1.SESの設定

[Amazon SES](https://aws.amazon.com/jp/ses/)を使って管理者（自分）にメールを送れるように設定を行います。

***

![ses_select](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/ses_select.png)

- [AWSマネジメントコンソール](https://console.aws.amazon.com/)にアクセスし、「ses」と入力し、検索結果（プルダウン）の「Simple Email Service」をクリックします。

***

![verify_email](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/verify_email.png)

- 左ペインの「Email Address」をクリックし、「Verify a New Mail Address」ボタンをクリックします。

***

![input_email](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/input_email.png)

- フォームにメールアドレスを入力し、「Verify This Email Address」ボタンをクリックします。※こちらの見本アドレスはテスト用です。ご自身の実在するメールアドレスを使用してください。

***

![click_link](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/click_link.png)

- 入力したメールアドレス宛に上記のようなメールが届くので、リンクをクリックしてメールアドレスの検証を完了させます。

## 2.Lambda関数の作成

[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を使って送られてきた問合せ内容を解析して感情判定し、判定結果を加えた上で[Amazon SES](https://aws.amazon.com/jp/ses/)を使って管理者にメールを送信する関数を作成します。

***

![lambda_select](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/lambda_select.png)

- [AWSマネジメントコンソール](https://console.aws.amazon.com/)にアクセスし、「lambda」と入力し、検索結果（プルダウン）の「Lambda」をクリックします。

***

![lambda_create](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/lambda_create.png)

- 「関数の作成」をクリックします。

***

![lambda_config](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/lambda_config.png)

- 「関数名」に「contactFunction」と入力します（命名は何でも構いません）。
- 「ランタイム」に、今回は「Python 3.7」を選択します。
- 「実行ロールの選択または作成」の項目を展開し、「基本的なLambdaアクセス権限で新しいロールを作成」を選択します。
- 最後に、「関数の作成」をクリックします。

***

![lambda_role](https://user-images.githubusercontent.com/13535662/115181448-f1043480-a112-11eb-97b0-2378058028c0.png)

- 関数が正常に作成できたら、「設定」タブの「アクセス権限」をクリックして、「ロール名」に表示されるロールをクリックします。

***

![policy_attach](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/policy_attach.png)

- 「ポリシーをアタッチします」ボタンをクリックします。

***

![comprehend_policy](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/comprehend_policy.png)

- 検索フォームに「comprehend」と入力します。
- 「ComprehendFullAccess」にチェックを入れます。
- 「ポリシーのアタッチ」ボタンをクリックします。

***

![ses_policy](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/ses_policy.png)

- 同じ要領で再度「ポリシーをアタッチします」ボタンを押してから、検索フォームに「ses」と入力します。
- 「AmazonSESFullAccess」にチェックを入れます。
- 「ポリシーのアタッチ」ボタンをクリックします。

***

![function_code](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/function_code.png)

- 作成したLambda関数「contactFunction」に戻り、「コード」タブの「ソースコード」のエディタでコードを記述していきます。

lambda_function.py

```
import json
import boto3

REGION = 'ap-northeast-1'
SRC_MAIL = "test@example.com"#SESに登録したメールアドレス
DST_MAIL = "test@example.com"#SESに登録したメールアドレス
SENTIMENTS = {
    "NEUTRAL": "中立的",
    "POSITIVE": "肯定的",
    "NEGATIVE": "否定的",
    "MIXED": "混在"
}
LANGUAGE_CODE = 'ja'

def detect_text_sentiment(text):#Comprehendを使って感情を検知するメソッド
    comprehend = boto3.client('comprehend', region_name=REGION)
    response = comprehend.detect_sentiment(Text=text, LanguageCode=LANGUAGE_CODE)
    return response

def send_email(source, to, subject, body):#SESでメールを送るメソッド
    ses = boto3.client('ses', region_name=REGION)
 
    response = ses.send_email(
        Source=source,
        Destination={
            'ToAddresses': [
                to,
            ]
        },
        Message={
            'Subject': {
                'Data': subject,
            },
            'Body': {
                'Text': {
                    'Data': body,
                },
            }
        }
    )
     
    return response

def lambda_handler(event, context):
    sentiment = detect_text_sentiment(event['content'])['Sentiment']
    body = "件名:\n" + event['title'] + "\n" + "\n連絡先:\n" + event['contact'] + "\n" + "\n感情:\n" + SENTIMENTS[sentiment] + "\n" + "\n内容:\n" + event['content']
    mail = send_email(SRC_MAIL, DST_MAIL, 'お問い合わせがありました', body)
    
    return {
        'statusCode': 200,
        'body': {
            'sentiment': sentiment,
            'mail': mail
        }
    }
```

- 上記の通りのコードを記述します。

***

![create_test](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/create_test.png)

- 関数のテストイベントを作成して、動作確認します。
- 上記の「テスト」ボタンをクリックします。

***

![configure_test](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/configure_test.png)

- 上記の通り、「イベント名」（任意ですのでなんでも構いません）とテスト用のパラメータを記述して、「作成」ボタンをクリックします。

***

![create_test](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/deploy_function.png)

- 関数をデプロイし、もう一度「テスト」ボタンをクリックすると、テストが実行されます。

***

![mail_result](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/mail_result.png)

- 登録したアドレスに、上記の通りのメールが送られていれば成功です。
- 「感情」という項目に、感情の判定結果が表示されているかと思います。

## 3.API Gatewayの設定

作成したLambda関数をAPIとして呼び出しできる様に、[Amazon API Gateway](https://aws.amazon.com/jp/api-gateway/)を設定していきます。

***

![add_trigger](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/add_trigger.png)

- 引き続き作成したLambda関数「contactFunction」の画面で、「＋トリガーを追加」ボタンをクリックします。

***

![configure_trigger](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/configure_trigger.png)

- 選択肢はそれぞれ「API Gateway」「Create an API」「REST API」「オープン」を選択し、「追加」ボタンをクリックします。

***

![select_trigger](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/select_trigger.png)

- Lambda関数の画面に戻るので、「contactFunction-API」をクリックします。

***

![create_method](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/create_method.png)

- 「アクション」をクリックし、プルダウンメニューから「メソッドの作成」をクリックします。

***

![select_post](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/select_post.png)

![method_check](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/method_check.png)

- 表示されるプルダウンメニューの「POST」をクリックして、右側のチェックマークをクリックします。

***

![method_setup](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/method_setup.png)

- その後表示されるメソッドのセットアップ画面で、Lambda関数の項目で「contactFunction(作成したLambda関数名)」を入力（入力中に候補が表示されるので、クリックでOKです）します。
- 「保存」ボタンをクリックします。

***

![method_setup_ok](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/method_setup_ok.png)

- 「OK」ボタンをクリックします。

***

![integrated_request](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/integrated_request.png)

- 続いて、「統合リクエスト」をクリックします。

***

![add_mapping_template](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/add_mapping_template.png)

- 「マッピングテンプレート」をクリックし、「テンプレートが定義されていない場合 (推奨) 」を選択し、「マッピングテンプレートの追加」をクリックします。

***

![open_mapping_template](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/open_mapping_template.png)

- 「application/json」と入力し、チェックマークをクリックします。

***

![enter_mapping_template](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/enter_mapping_template.png)

- 下にスクロールして、マッピングテンプレートの入力フィールドをクリックします。

```
{
  "contact": $input.json("$.contact"),
  "title": $input.json("$.title"),
  "content": $input.json("$.content")
}
```

- 上記の通りのテンプレートを入力して、「保存」をクリックします。

***

![select_cors](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/select_cors.png)

- 「アクション」から「CORS の有効化」をクリックします。

***

![activate_cors](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/activate_cors.png)

- 「CORS を有効にして既存の CORS ヘッダーを置換」ボタンをクリックします。

***

![confirm_cors](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/confirm_cors.png)

- 「はい、既存の値を置き換えます」ボタンをクリックします。

***

![select_api_deploy](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/select_api_deploy.png)

- 「アクション」から「API のデプロイ」をクリックします。

***

![confirm_api_deploy](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/confirm_api_deploy.png)

- 「デプロイされるステージ」に「default」を選択し、「デプロイ」をクリックします。

***

![test_post](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/test_post.png)

- 次のページの上記に表示されるURLの末尾に「/contactFunction」を追加したURLに対して、**curl**でPOSTメソッドのリクエストを送信して、動作確認しましょう。

Terminal

```
curl -X POST -H "Content-Type: application/json" -d '{"title":"curl", "contact":"curl@example.com", "content": "I am so happy."}' https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/default/contactFunction
```

- ターミナルにて、上記コマンドを実行して、登録したアドレスに正常にメールが送信されればOKです。
- 「**xxxxxxxxxx**」の部分は、実際に生成されたURLの「https://」に続く部分を記述してください。
- この「**xxxxxxxxxx**」の部分は後から使用しますので、コピーして撮っておいてください（忘れてしまった場合は再度マネジメントコンソールから**Lambda関数を表示して、API Gateway > 詳細 > API エンドポイント**で確認していただけます）。

***

## APIの削除

デプロイしたAPIは、そのままにしておくとオンデマンドで費用が発生します。  
料金をゼロにするためには、以下の手順にて関連ソースの削除を行います。

#### API Gatewayの削除

まずはLambda関数を呼び出すためのAPI Gatewayを削除します。

***

![select_api_gateway](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/select_api_gateway.png)

- [AWSマネジメントコンソール](https://console.aws.amazon.com/)にアクセスし、「api」と入力し、検索結果（プルダウン）の「API Gateway」をクリックします。

***

![delete_api_gateway](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/delete_api_gateway.png)

- **contactFunction-API**のラジオボタンをクリックして、**Actions**のプルダンメニューの**Delete**をクリックします。

***

![confirm_delete_api_gateway](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/confirm_delete_api_gateway.png)

- **削除**をクリックすれば、API Gatewayの削除完了です。

***

#### Lambda関数の削除

続いて、Lambda関数を削除していきます。

***

![lambda_select](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/lambda_select.png)

- [AWSマネジメントコンソール](https://console.aws.amazon.com/)にアクセスし、「lambda」と入力し、検索結果（プルダウン）の「Lambda」をクリックします。

***

![delete_lambda](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/delete_lambda.png)

- **contactFunction**のラジオボタンをクリックして、**アクション**のプルダンメニューの**削除**をクリックします。

***

![confirm_delete_lambda](https://github.com/matsuihidetoshi/comprehend-handson/blob/main/images/confirm_delete_lambda.png)

- **削除**をクリックすれば、Lambda関数の削除完了です。

***

これで、今回のハンズオンで作成したAPIの全てのリソースが削除されます。    
以上で終了です。お疲れ様でした！
