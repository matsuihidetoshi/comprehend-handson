# Amplify + Vue.js + Lambda + Comprehend で問合せした人の感情を判定するフォームを作る

こんにちは、JAWS-UG 浜松支部の松井です。  
近年、機械学習、AIに対する注目が集まっています。しかし、何らかの機械学習を活用した仕組みを1人で0から作るのはなかなかハードルが高く、機械学習で重要になってくるビッグデータの収集も大きな問題になってきます。  
AWSでは、こうした問題を解決してくれる、機械学習を誰でも簡単に活用できるサービスが多数提供されています。  
今回はその中でも、機械学習を使用してテキスト内でインサイトや関係性を検出する自然言語処理 (NLP) サービス[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を活用して、 **問合せした人の感情を判定してくれるフォーム** を作ってみようと思います。

- 機械学習、AI活用といっても、どこから手をつけたらいいか分からない
- 日常の生活や業務でどの様に役に立てれば良いか分からない

という方も多くいらっしゃると思いますが、[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を活用したこちらのハンズオンが一助となれば幸いです。

## 前提環境

今回のハンズオンは下記の環境&ツールにて検証しています。 ※必要なツールが揃っていればPCやOSは問いません

- OS: Mac OS X
- node: v14.6.0
- npm: 6.14.7
- vue: @vue/cli 4.4.6
- amplify: 4.27.3

また、[こちら](https://aws.amazon.com/jp/builders-flash/202008/amplify-crud-app/)をご参考に、上記の必要なツールを揃えた環境を[AWS Cloud9](https://aws.amazon.com/jp/cloud9/)上に構築していただくこともできます。

## 今回のアーキテクチャ

今回のハンズオンで作成するシステムの構成図です。

![aws_top](https://github.com/matsuihidetoshi/contact/blob/master/images/architecture.png)

- 動作の起点となるお問い合わせフォームは **Vue.js** を使って作成し、[AWS Amplify](https://aws.amazon.com/jp/amplify/)を使って[Amazon S3](https://aws.amazon.com/jp/s3/)バケット上にデプロイします。煩雑な環境構築のステップを省略し、証明書付きのURLを自動生成してくれるのでとても便利です。
- お問い合わせフォームの入力内容はAjaxを使ってPOSTリクエストを送信し、[Amazon API Gateway](https://aws.amazon.com/jp/api-gateway/)が受け取り、[AWS Lambda](https://aws.amazon.com/jp/lambda/)へプロキシします。
- [AWS Lambda](https://aws.amazon.com/jp/lambda/)では[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を使って問合せ内容を解析し、文章から感情を判定します。その判定内容を添えて、[Amazon SES](https://aws.amazon.com/jp/ses/)へ管理者へのメール送信をリクエストします。


## 1.SESの設定

[Amazon SES](https://aws.amazon.com/jp/ses/)を使って管理者（自分）にメールを送れるように設定を行います。

***

![ses_select](https://github.com/matsuihidetoshi/contact/blob/master/images/ses_select.png)

- [AWSマネジメントコンソール](https://console.aws.amazon.com/)にアクセスし、「ses」と入力し、検索結果（プルダウン）の「Simple Email Service」をクリックします。

***

![verify_email](https://github.com/matsuihidetoshi/contact/blob/master/images/verify_email.png)

- 左ペインの「Email Address」をクリックし、「Verify a New Mail Address」ボタンをクリックします。

***

![input_email](https://github.com/matsuihidetoshi/contact/blob/master/images/input_email.png)

- フォームにメールアドレスを入力し、「Verify This Email Address」ボタンをクリックします。※こちらの見本アドレスはテスト用です。ご自身の実在するメールアドレスを使用してください。

***

![click_link](https://github.com/matsuihidetoshi/contact/blob/master/images/click_link.png)

- 入力したメールアドレス宛に上記のようなメールが届くので、リンクをクリックしてメールアドレスの検証を完了させます。

## 2.Lambda関数の作成

[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を使って送られてきた問合せ内容を解析して感情判定し、判定結果を加えた上で[Amazon SES](https://aws.amazon.com/jp/ses/)を使って管理者にメールを送信する関数を作成します。

***

![lambda_select](https://github.com/matsuihidetoshi/contact/blob/master/images/lambda_select.png)

- [AWSマネジメントコンソール](https://console.aws.amazon.com/)にアクセスし、「lambda」と入力し、検索結果（プルダウン）の「Lambda」をクリックします。

***

![lambda_create](https://github.com/matsuihidetoshi/contact/blob/master/images/lambda_create.png)

- 「関数の作成」をクリックします。

***

![lambda_config](https://github.com/matsuihidetoshi/contact/blob/master/images/lambda_config.png)

- 「関数名」に「contactFunction」と入力します（命名は何でも構いません）。
- 「ランタイム」に、今回は「Python 3.7」を選択します。
- 「実行ロールの選択または作成」の項目を展開し、「基本的なLambdaアクセス権限で新しいロールを作成」を選択します。
- 最後に、「関数の作成」をクリックします。

***

![lambda_role](https://github.com/matsuihidetoshi/contact/blob/master/images/lambda_role.png)

- 関数が正常に作成できたら、「アクセス権限」タブをクリックして、「ロール名」に表示されるロールをクリックします。

***

![policy_attach](https://github.com/matsuihidetoshi/contact/blob/master/images/policy_attach.png)

- 「ポリシーをアタッチします」ボタンをクリックします。

***

![comprehend_policy](https://github.com/matsuihidetoshi/contact/blob/master/images/comprehend_policy.png)

- 検索フォームに「comprehend」と入力します。
- 「ComprehendFullAccess」にチェックを入れます。
- 「ポリシーのアタッチ」ボタンをクリックします。

***

![ses_policy](https://github.com/matsuihidetoshi/contact/blob/master/images/ses_policy.png)

- 同じ要領で再度「ポリシーをアタッチします」ボタンを押してから、検索フォームに「ses」と入力します。
- 「AmazonSESFullAccess」にチェックを入れます。
- 「ポリシーのアタッチ」ボタンをクリックします。

***

![function_code](https://github.com/matsuihidetoshi/contact/blob/master/images/function_code.png)

- 作成したLambda関数「contactFunction」に戻り、「設定」タブの「関数コード」のエディタでコードを記述していきます。

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

![create_test](https://github.com/matsuihidetoshi/contact/blob/master/images/create_test.png)

- 関数のテストイベントを作成して、動作確認します。
- 上記の「テスト」ボタンをクリックします。

***

![configure_test](https://github.com/matsuihidetoshi/contact/blob/master/images/configure_test.png)

- 上記の通り、「イベント名」（任意ですのでなんでも構いません）とテスト用のパラメータを記述して、「作成」ボタンをクリックします。

***

![create_test](https://github.com/matsuihidetoshi/contact/blob/master/images/create_test.png)

- もう一度「テスト」ボタンをクリックすると、テストが実行されます。

***

![mail_result](https://github.com/matsuihidetoshi/contact/blob/master/images/mail_result.png)

- 登録したアドレスに、上記の通りのメールが送られていれば成功です。
- 「感情」という項目に、感情の判定結果が表示されているかと思います。

## 3.API Gatewayの設定

作成したLambda関数をAPIとして呼び出しできる様に、[Amazon API Gateway](https://aws.amazon.com/jp/api-gateway/)を設定していきます。

***

![add_trigger](https://github.com/matsuihidetoshi/contact/blob/master/images/add_trigger.png)

- 引き続き作成したLambda関数「contactFunction」の画面で、「＋トリガーを追加」ボタンをクリックします。

***

![configure_trigger](https://github.com/matsuihidetoshi/contact/blob/master/images/configure_trigger.png)

- 選択肢はそれぞれ「API Gateway」「Create an API」「REST API」「オープン」を選択し、「追加」ボタンをクリックします。

***

![select_trigger](https://github.com/matsuihidetoshi/contact/blob/master/images/select_trigger.png)

- Lambda関数の画面に戻るので、「contactFunction-API」をクリックします。

***

![create_method](https://github.com/matsuihidetoshi/contact/blob/master/images/create_method.png)

- 「アクション」をクリックし、プルダウンメニューから「メソッドの作成」をクリックします。

***

![select_post](https://github.com/matsuihidetoshi/contact/blob/master/images/select_post.png)

![method_check](https://github.com/matsuihidetoshi/contact/blob/master/images/method_check.png)

- 表示されるプルダウンメニューの「POST」をクリックして、右側のチェックマークをクリックします。

***

![method_setup](https://github.com/matsuihidetoshi/contact/blob/master/images/method_setup.png)

- その後表示されるメソッドのセットアップ画面で、Lambda関数の項目で「contactFunction(作成したLambda関数名)」を入力（入力中に候補が表示されるので、クリックでOKです）します。
- 「保存」ボタンをクリックします。

***

![method_setup_ok](https://github.com/matsuihidetoshi/contact/blob/master/images/method_setup_ok.png)

- 「OK」ボタンをクリックします。

***

![integrated_request](https://github.com/matsuihidetoshi/contact/blob/master/images/integrated_request.png)

- 続いて、「統合リクエスト」をクリックします。

***

![add_mapping_template](https://github.com/matsuihidetoshi/contact/blob/master/images/add_mapping_template.png)

- 「マッピングテンプレート」をクリックし、「テンプレートが定義されていない場合 (推奨) 」を選択し、「マッピングテンプレートの追加」をクリックします。

***

![open_mapping_template](https://github.com/matsuihidetoshi/contact/blob/master/images/open_mapping_template.png)

- 「application/json」と入力し、チェックマークをクリックします。

***

![enter_mapping_template](https://github.com/matsuihidetoshi/contact/blob/master/images/enter_mapping_template.png)

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

![select_cors](https://github.com/matsuihidetoshi/contact/blob/master/images/select_cors.png)

- 「アクション」から「CORS の有効化」をクリックします。

***

![activate_cors](https://github.com/matsuihidetoshi/contact/blob/master/images/activate_cors.png)

- 「CORS を有効にして既存の CORS ヘッダーを置換」ボタンをクリックします。

***

![confirm_cors](https://github.com/matsuihidetoshi/contact/blob/master/images/confirm_cors.png)

- 「はい、既存の値を置き換えます」ボタンをクリックします。

***

![select_api_deploy](https://github.com/matsuihidetoshi/contact/blob/master/images/select_api_deploy.png)

- 「アクション」から「API のデプロイ」をクリックします。

***

![confirm_api_deploy](https://github.com/matsuihidetoshi/contact/blob/master/images/confirm_api_deploy.png)

- 「デプロイされるステージ」に「default」を選択し、「デプロイ」をクリックします。

***

![test_post](https://github.com/matsuihidetoshi/contact/blob/master/images/test_post.png)

- 次のページの上記に表示されるURLの末尾に「/contactFunction」を追加したURLに対して、**curl**でPOSTメソッドのリクエストを送信して、動作確認しましょう。

Terminal

```
curl -X POST -H "Content-Type: application/json" -d '{"title":"curl", "contact":"curl@example.com", "content": "I am so happy."}' https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/default/contactFunction
```

- ターミナルにて、上記コマンドを実行して、登録したアドレスに正常にメールが送信されればOKです。
- 「**xxxxxxxxxx**」の部分は、実際に生成されたURLの「https://」に続く部分を記述してください。
- この「**xxxxxxxxxx**」の部分は後から使用しますので、コピーして撮っておいてください（忘れてしまった場合は再度マネジメントコンソールから**Lambda関数を表示して、API Gateway > 詳細 > API エンドポイント**で確認していただけます）。

***

## 3.Vue.jsプロジェクトを作成

Vue CLIを使って、必要な設定を行いながらVue.jsプロジェクトを作成していきます。

#### vue create

contactという名前でプロジェクトを作成

```
vue create contact
```

#### npmリポジトリの設定(y or nを入力しEnter)

nを入力しEnter

```
?  Your connection to the default npm registry seems to be slow.
   Use https://registry.npm.taobao.org for faster installation? (Y/n) n
```

#### 設定方法の選択(↑↓でカーソル移動、Enterで決定)

Manually select featuresを選択しEnter

```
Vue CLI v4.1.2
? Please pick a preset: 
  default (babel, eslint) 
❯ Manually select features 
```

#### 追加パッケージを選択(↑↓でカーソル移動、Spaceで選択、Enterで決定)

下記の通り、Babel, Progressive Web App (PWA) Support, Router, Linter / Formatterを選択

```
? Check the features needed for your project: 
 ◉ Babel
 ◯ TypeScript
 ◉ Progressive Web App (PWA) Support
❯◉ Router
 ◯ Vuex
 ◯ CSS Pre-processors
 ◉ Linter / Formatter
 ◯ Unit Testing
 ◯ E2E Testing
 ```
 
 #### Historyモードを選択(y or nを入力しEnter)
 
 yを入力しEnter
 
 ```
 ? Use history mode for router? (Requires proper server setup for index fallback in production) (Y/n) y
 ```
 
 #### ESLintの設定を選択(↑↓でカーソル移動、Enterで決定)
 
 デフォルトのESLint with error prevention onlyを選択しEnter
 
 ```
 ? Pick a linter / formatter config: (Use arrow keys)
❯ ESLint with error prevention only 
  ESLint + Airbnb config 
  ESLint + Standard config 
  ESLint + Prettier 
```

#### Lintのタイミングの設定(↑↓でカーソル移動、Enterで決定)

デフォルトのLint on saveを選択しEnter

```
? Pick additional lint features: (Press <space> to select, <a> to toggle all, <i> to invert selection)
❯◉ Lint on save
 ◯ Lint and fix on commit
 ```
 
#### Configファイルの設定(↑↓でカーソル移動、Enterで決定)

デフォルトのIn dedicated config filesを選択しEnter

```
? Where do you prefer placing config for Babel, ESLint, etc.? (Use arrow keys)
❯ In dedicated config files 
  In package.json
```

#### 今回の設定を保存指定次回以降に転用するかの選択(y or nを入力しEnter)

任意だが、今回はnを入力しEnter

```
? Save this as a preset for future projects? (y/N) n
```

#### Vue.jsプロジェクトにCloud9向けの設定を追加 ※Cloud9の場合

Cloud9環境をお使いの方は、Cloud9の仕組み上起動した開発環境のサーバーをうまくプレビューできないため  
設定を追加します。

- [参考リポジトリのcontact/vue.config.js](https://github.com/matsuihidetoshi/contact/blob/master/vue.config.js)

を、作成しているプロジェクトで同じ階層になる様に

- contact/vue.config.js

の様にコピーします
#### テスト起動

開発環境のローカルサーバーを起動してプレビューし、問題なくVue.jsプロジェクトが  
立ち上がることを確認します。

#### Vue.jsプロジェクトフォルダ(contact)に移動

```
cd contact
```

#### サーバー起動

```
npm run serve
```

ローカルのPCの場合はブラウザにてlocalhost:8080にアクセス
Cloud9の場合、画面の上部のPreview→Preview Running Applicationをクリックし、  
Vue.jsのデフォルト画面が表示されればOK  
*そのままサーバーは起動したまま、別ターミナルを開き、contactディレクトリに移動して以降の作業を実施

## 4.フロントエンド側のコードを作成

ここまでで、APIの作成とVue.jsのプロジェクトの作成ができました。  
続いて、ユーザーが実際に問合せできる様にフォームを作成していきます。

#### Vuetifyのインストール

今回は簡単な問合せフォームを作成するだけですが、手軽にマテリアルデザインを導入できるデザインフレームワークの**Vuetify**を導入します。  
これにより、優れたUIのフォームを簡単に作成することができます。

```
vue add vuetify
```

- 以上を実行

```
? Choose a preset: (Use arrow keys)
❯ Default (recommended) 
  Prototype (rapid development) 
  Configure (advanced)
```

- Default (recommended)のまま、Enter

#### axiosのインストール

作成したAPIへリクエストを送信するためにライブラリ**axios**をインストールします。

```
npm i axios
```

- 以上を実行



#### Contact.vue

続いて、必要なコードを[参考リポジトリ](https://github.com/matsuihidetoshi/contact)からコピーしていきます。  
まずはお問い合わせフォームページのビューをコピーします。

- https://github.com/matsuihidetoshi/contact/blob/master/src/views/Contact.vue

を

- contact/src/views/Contact.vue

となる様にコピーします

また、169行目に先ほど**3.API Gatewayの設定**の項目の最後に生成された**API Gatewayのエンドポイント**を設定されます。

- contact/src/views/Contact.vue#169

```
- baseURL: 'https://example_url.execute-api.ap-northeast-1.amazonaws.com'
+ baseURL: 'https://xxxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com' //3.API Gatewayの設定で生成されたエンドポイント
```

#### router/index.js

ルーティングの設定を記述するファイルです。  
ログイン状態を管理してページのアクセス制御をします。

- https://github.com/matsuihidetoshi/contact/blob/master/src/router/index.js

の内容を

- contact/src/router/index.js

に上書き

#### App.vue

アプリケーションの共通レイアウトのコンポーネントです。  
各ページへのナビゲーションを追加します。

- https://github.com/matsuihidetoshi/contact/blob/master/src/App.vue

の内容を

- contact/src/App.vue

に上書き

#### 動作確認

すでにアプリケーションがローカルで起動している状態でしたら、一旦**ctrl + c**でサーバーを停止してから、

```
npm run serve
```

を実行してアプリケーションを起動し、プレビューしてみてください。  
**Contact**ページからお問い合わせを送信し、設定したメールアドレスに感情解析の結果とともにお問い合わせ内容が届けばOKです。

## フロントエンドのデプロイ

ここまでで、

- バックエンドの本番環境

- フロントエンドのコード

が完成しました。あとは、フロントエンドの本番環境のホスティングができれば完成です。

#### GitHubにコードをプッシュ

まず、今までの変更をGitでコミットしていきます。

```
git add .
git commit -m 'built application'
```

その後、GitHubにサインインしてください。

***

![github_signin](https://github.com/matsuihidetoshi/contact/blob/master/images/github_signin.png)

- ユーザー名またはメールアドレスとパスワードを入力し、「Sign in」をクリックします。

***

![new_repository](https://github.com/matsuihidetoshi/contact/blob/master/images/new_repository.png)

- 上部ナビゲーションバー右上の`+`をクリックし、`New repository`をクリックしてください。

***

![create_repository](https://github.com/matsuihidetoshi/contact/blob/master/images/create_repository.png)

- `Repository name`に、`contact`と入力し、それ以外はデフォルトのまま`Create repository`をクリックしてください。

***

![copy_push_command](https://github.com/matsuihidetoshi/contact/blob/master/images/copy_push_command.png)

- `…or push an existing repository from the command line`という項目をコピーし、ターミナルでプロジェクトフォルダにて実行してください。

***

```
git remote add origin https://github.com/ユーザー名/contact.git
git push -u origin master
```

下記のようにユーザー名、パスワードを求められるので、GitHubの`ユーザー名` `パスワード`を入力してください。

```
Username for 'https://github.com/matsuihidetoshi/contact.git':ユーザー名
Password for 'https://matsuihidetoshi@github.com/matsuihidetoshi/contact.git':パスワード
```

これで、GitHubにコードがプッシュされ、デプロイの準備ができました。

#### Amplifyコンソールからデプロイ

通常、Webページをホスティングするには、サーバー構築・ネットワーク設定・ミドルウェアのインストール及び設定等が必要ですが、  
Amplifyコンソールを使用するとWebインターフェースから少ないステップで簡単にデプロイできます。

#### Amplifyコンソールを開く

![choose_amplify](https://github.com/matsuihidetoshi/contact/blob/master/images/choose_amplify.png)

- AWSマネジメントコンソールから、`Amplify`を検索し、選択します。  

***

![select_contact](https://github.com/matsuihidetoshi/contact/blob/master/images/select_contact.png)

- すでに`contact`というアプリケーションの項目が作成されているはずですので、それをクリックしてください。  

***

#### デプロイ - Githubの連携

![select_github](https://github.com/matsuihidetoshi/contact/blob/master/images/select_github.png)

- `GitHub`を選択し、`Connect branch`をクリックしてください。  

***

![oauth](https://github.com/matsuihidetoshi/contact/blob/master/images/oauth.png)

- GitHubの認証ページが開く場合は、`ユーザー名またはメールアドレス` と `パスワード`を入力してログインしてください。  
- OAuthによるアクセス許可の画面が表示されますので、`Authorize aws-amplify-console`をクリックしてください。

***

![select_repository](https://github.com/matsuihidetoshi/contact/blob/master/images/select_repository.png)

- `リポジトリ`は`GitHubユーザー名/contact`を選択し、ブランチは`master`を選択し、`次へ`をクリックしてください。  

***

#### デプロイ - ビルド設定

![build_config](https://github.com/matsuihidetoshi/contact/blob/master/images/build_config.png)

- `ビルド設定の構成`画面が開くので、`Select a backend environment`で`default`を選択してください。
- `Select an existing service role or create a new one so Amplify Console may access your resources.`という項目で、`Create new role`をクリックしてください。  

***

#### デプロイ - ロールの作成

![role_service](https://github.com/matsuihidetoshi/contact/blob/master/images/role_service.png)

- `ロールの作成`画面が表示されますので、デフォルトのまま`次のステップ: アクセス権限`をクリックしてください。

***

![role_policy](https://github.com/matsuihidetoshi/contact/blob/master/images/role_policy.png)

- 次の画面で`Attached アクセス権限ポリシー`という項目などが表示されますが、こちらもデフォルトのまま`次のステップ: タグ`をクリックしてください。  

***

![role_tag.png](https://github.com/matsuihidetoshi/contact/blob/master/images/role_tag.png)

- 次の画面で`タグの追加（オプション）`という項目が表示されますが、こちらもデフォルトのまま`次のステップ: 確認`をクリックしてください。

***

![role_confirm](https://github.com/matsuihidetoshi/contact/blob/master/images/role_confirm.png)

- 確認画面が開きますが、そのまま`ロールの作成`をクリックしてください。
- 画面が遷移したら、そのページは閉じてしまって構いません。

***

#### デプロイ- ビルド設定2

![role_select](https://github.com/matsuihidetoshi/contact/blob/master/images/role_select.png)

- 先ほど開いていたAmplifyコンソールに戻ってください。
- `Select an existing service role or create a new one so Amplify Console may access your resources.`の項目のプルダウンの横の🔄マークをクリックして下さい。
- 先ほど作成した`amplifyconsole-backend-role`を選択してください。  
- その後、`次へ`をクリックしてください。  

***

![deploy_confirm](https://github.com/matsuihidetoshi/contact/blob/master/images/deploy_confirm.png)

- `確認`画面が表示されますが、`保存してデプロイ`をクリックしてください。  

***

![deploy_progress](https://github.com/matsuihidetoshi/contact/blob/master/images/deploy_progress.png)
  
- ここから少し時間がかかりますが（10分ほど）、デプロイのフローが終わるまでお待ちください。
- デプロイが完了してからURLのリンクをクリックすると、アプリケーションが開きます。 
- 動作確認し、問題なければ完成です。おめでとうございます！

***

## アプリケーションの削除

デプロイしたアプリケーションは、そのままにしておくとオンデマンドで**課金**されます。  
よほどアクセスが集中しない限り、ほとんど無視できるレベルの金額しか課金されませんが、  
全てのリソースを削除しておけば課金されることはありません。  
以下にアプリケーションリソース全体の削除方法を記載します。

#### CLIによる削除

まずはバックエンドのリソースを削除します。  
プロジェクトフォルダにて、

```
amplify delete
```

を実行してください。  
すると下記の通り確認が表示されますので、承諾してください(yを入力しEnter)。

```
? Are you sure you want to continue? (This would delete all the environments of 
the project from the cloud and wipe out all the local amplify resource files) (Y/n) y
```

問題なく削除されれば、下記の通り表示されます。

```
✔ Project deleted in the cloud
Project deleted locally.
```

#### Amplifyコンソールから削除

S3にデプロイされたアプリケーションのフロントエンドも削除する必要があります。

***

![delete_select_contact](https://github.com/matsuihidetoshi/contact/blob/master/images/delete_select_contact.png)

- **Amplifyコンソール**を開き、**全てのアプリ**→**contact**を選択してください。  

***

![delete](https://github.com/matsuihidetoshi/contact/blob/master/images/delete.png)

- 画面右上の**アクション**から、**アプリの削除**を選択してください。  

***

![delete_confirm](https://github.com/matsuihidetoshi/contact/blob/master/images/delete_confirm.png)

- 確認用ダイアログが表示されるので、フォームに**delete**を入力し、**Delete**を押下してください。

***

これで、今回のハンズオンで作成したアプリケーションの全てのリソースが削除されます。  
**今回使用したCloud9環境を使用しない場合、別途削除していただく必要があります  
(使用する分だけEC2インスタンスタイプに応じた課金が発生します)。**

  
以上で終了です。お疲れ様でした！
