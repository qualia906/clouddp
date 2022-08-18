## 使用するサービス

-   Cloud Pub/Sub
    -   アプリケーションやサービスの間でイベントデータを非同期で交換するためのメッセージング サービスです。
    

-   Cloud Run
    -   Docker コンテナ アプリケーションをサーバーレスで実行することができるサービスです。

    -   このラボでは、メッセージのパブリッシャーである Lab Report Service とサブスクライバーである Email Service、SMS Service の 3 つのアプリケーションをコンテナ化して Cloud Run で実行します。
    

-   Cloud Build
    -   継続的インテグレーション (CI) のサービスです。
    
    -   コードからコンテナイメージをビルド (作成) し、Google Container Registry に自動で保存します。
    

-   Google Container Registry
    -   コンテナ レジストリ (コンテナ イメージの格納庫) のサービスです。
    
    -   ここに保存したコンテナ イメージから、Cloud Run や Google Kubernetes Engine (GKE) にコンテナ アプリケーションをデプロイして実行することができます。
    
<br />

## アーキテクチャ

次の図にアーキテクチャの概要を示します。

-   Lab Report Service は HTTPS POST リクエストを受け取ると、Cloud Pub/Sub のトピックにメッセージをパブリッシュします。
    
-   Cloud Pub/Sub にはトピックと、Email Service 用、SMS Service 用の 2 つのサブスクリプションを構成します。
    
-   それぞれのサブスクリプションは、Push 型で Email Service と SMS Service の 2 つのサブスクライバーにメッセージを送信します。
    

![650_arch.png](https://lh3.googleusercontent.com/EGGYFx1X9zMhaIJ4p1fxYG0Jrb7_M0Pa2wAzOnUnQzXNYIcpkJoY6KX8JwMD8iEylZIxrH6xW44UVZYle9NFHAcxY68qb2JnspEAIvH3cWsSa7hHSq7T8dpDWA2vzf3BAYqJ9UsvI9cCVj0IdWFqZA)

## Pub/Sub トピックの作成

まず new-lab-report という Pub/Sub トピックを作成します。

![650-system-arch.png](https://lh6.googleusercontent.com/I5eqUVRDlA1zfI1XqYGmQN0Lg7PtNnqdOJ7qTbOT1FsBIvvmYPb6JRGCFmX5kziHKn5S_zIgFXz2jhD0N3PkwOnRtImwJsFHf3FZbtKy0IQvD8XF-1Bu7Yy520BW43W0lrQ92mgLkrVIqKnRgZpg4Q)

<br />

1. Cloud Console で Cloud Shell を起動します。

2. 次のコマンドを実行して、Pub/Sub トピックを作成します。

   ```
   gcloud pubsub topics create new-lab-report
   ```

3. Cloud Run を使用できるようにサービスを有効化します。

   ```
   gcloud services enable run.googleapis.com
   ```
<br />
 
## Lab Report Serive

次に Pub/Sub を介してメッセージを送信するパブリッシャーである Lab Report Service を作成します。

![9cd5510d5a234950a1b7c6cb1c00e327.png](https://lh6.googleusercontent.com/THDxSAiYGa1xlRpHflPwC-HgsAf9zFRHVPhCUoRCuLlmIYV608-EtVsRthhhLbvNsf7jBgRZHB6Xg2ynaHCihn5raq8x07NY9m_s5L3tSWTJgXXePtxkAMIemRMpYGIp_XCHAS5ASQ5QzUjU2ciNXA)

この Lab Report Service サービスは 2 つの機能を持ちます。

-   HTTPS POST でリクエストを受信する
    
-   Pub/Sub にメッセージをパブリッシュする
    

ここから、まずアプリケーションのコード (Node.js) を用意します。

そしてそのコードから Cloud Build というサービスを使って Docker コンテナのイメージを作成し Google Container Registry にイメージを保存します。

イメージを保存したら、Cloud Run というサービスで、イメージから起動したコンテナ アプリケーションを実行します。
<br />

1. Cloud Shell に戻り、このラボに必要な Git リポジトリのクローンを作成します。  
   そして、 lab-servicer ディレクトリに移動します。
   
   ```
   git clone https://github.com/rosera/pet-theory.git
   cd pet-theory/lab05/lab-service
   ```
   <br />

2. HTTPS リクエストを受信し、Pub/Sub にパブリッシュするのに必要な以下のパッケージをインストールします。

   ```    
   npm install express
   npm install body-parser
   npm install @google-cloud/pubsub
   ```

   これらのコマンドは package.json ファイルを更新し、このサービスに必要な依存関係を指定します。
   
   <br />

3. Cloud Run にコードをどのように開始するかを指示できるよう、package.json ファイルを編集します。  
   nano エディタで package.json ファイルを開きます。 

   ```
   nano package.json
   ```
   
   <br />

4. コード内の scripts セクションに以下のように "start": "node index.js" の行を追加して、ファイルを保存します。  
   ※ 行頭が適切にインデントされていることを確認してください。
   ※ インデントを修正する場合、全角スペースを使わないように注意してください。 
    
   ```
   "scripts": {
       "start": "node index.js",
       "test": "echo \"Error: no test specified\" && exit 1"
     },
   ```
   編集が完了したら、Ctrl + O、Enter、Ctrl + X で nano エディタを終了します。
   
   <br />

5. index.js という名前の新しいファイルを作成して、コードを追加します。  
   編集が完了したら、Ctrl + O、Enter、Ctrl + X で nano エディタを終了します。  
   ※ 行頭を適切にインデントされていることを確認してください。  
   ※ インデントを修正する場合、全角スペースを使用しないように注意してください。
    
   ```
   nano index.js
   
   # 下記のコードをコピー & ペーストで追加する
   const {PubSub} = require('@google-cloud/pubsub');
   const pubsub = new PubSub();
   const express = require('express');
   const app = express();
   const bodyParser = require('body-parser');
   app.use(bodyParser.json());
   const port = process.env.PORT || 8080;
   app.listen(port, () => {
     console.log('Listening on port', port);
   });
   app.post('/', async (req, res) => {
     try {
       const labReport = req.body;
       await publishPubSubMessage(labReport);
       res.status(204).send();
     }
     catch (ex) {
       console.log(ex);
       res.status(500).send(ex);
     }
   })
   async function publishPubSubMessage(labReport) {
     const buffer = Buffer.from(JSON.stringify(labReport));
     await pubsub.topic('new-lab-report').publish(buffer);
   }
   ```
  
   このコードの中核となるのは末尾の以下の部分です。

   ```
   const labReport = req.body;
   await publishPubSubMessage(labReport);
   ```

   これら 2 つの行で、サービスの主な処理が以下のとおり行われます。

   -   受け取った POST リクエストからメッセージ データを抽出します。
   -   メッセージ データを Cloud Pub/Sub のトピックにパブリッシュします。

<br />

6. 次に、Dockerfile という名前のファイルを作成して、以下のコードを追加します。  
   編集が完了したら、Ctrl + O、Enter、Ctrl + X で nano エディタを終了します。
    
   ```
   nano Dockerfile

   # 下記のコードを追加
   FROM node:10
   WORKDIR /usr/src/app
   COPY package.json package*.json ./
   RUN npm install --only=production
   COPY . .
   CMD [ "npm", "start" ]
   ```

   <br />

7. Cloud Build を使用してコンテナ イメージのビルド (作成) と Google Contrainer Registry への保存を実行します。

   ```
   gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service
   ```
   gcr.io は Google Container Registry のドメインを表しており、上記のコマンドを実行することで、ビルドされた lab-report-service という名前のコンテナ イメージが Google Container Registry に保存されます。

<br />

8. 上でビルド・保存したイメージから、Cloud Run に lab-report-service という名前でコンテナ アプリケーションをデプロイし実行します。

   ```
   gcloud run deploy lab-report-service \
     --image gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service \
     --platform managed \
     --region us-central1 \
     --allow-unauthenticated \
     --max-instances=1
   ```

   デプロイが完了したら以下のようなメッセージが表示されます
   
   ```
   Service [lab-report-service] revision [lab-report-service-00001] has been deployed and is serving traffic at https://lab-report-service-[hash].a.run.app
   ```

   -  \-\-image オプションでは、デプロイするコンテナ アプリケーションのイメージ (上でビルド・保存したイメージ) を指定しています。

   - \-\-region オプションでは、コンテナ アプリケーションをデプロイして実行するリージョンを指定しています。ここでは us-central1 リージョンを指定しています。

   - \-\-allow-unauthenticated オプションでは、コンテナ アプリケーションに対する未認証のユーザーによるリクエストを許可しています。

   - \-\-max-instances オプションでは、アプリケーションを実行するコンテナ インスタンスの最大数を指定しています。ここでは 1 インスタンスのみで実行するように指定しています。

### Lab Report Service のテスト

Lab Report Service の動作をテストするために、 3 件の HTTPS POST をシミュレートします。これら 3 件の POST には、それぞれ 1 つのサンプル メッセージ (ID 番号) が含まれています。
<br />

1. Lab Report Service の URL を環境変数に設定して使用しやすくします。
    
   ```
   export LAB_REPORT_SERVICE_URL=$(gcloud run services describe lab-report-service --platform managed --region us-central1 --format="value(status.address.url)")
   ```
   
   <br />

2. LAB_REPORT_SERVICE_URL が正しく取得されたことを確認します。
    
   ```
   echo $LAB_REPORT_SERVICE_URL
   ```
   
   <br />

3. 3 件の HTTPS POST リクエストを実行するシェルスクリプトを作成します。  
   post-reports.sh という名前で新しいファイルを作成して、コードを追加します。  
   編集が完了したら、Ctrl + O、Enter、Ctrl + X で nano エディタを終了します。
    
   ```
   nano post-reports.sh
   ```
   
   以下のコードをコピー & ペーストで追記します。

   ```
   curl -X POST \
     -H "Content-Type: application/json" \
     -d "{\"id\": 12}" \
     $LAB_REPORT_SERVICE_URL &
   curl -X POST \
     -H "Content-Type: application/json" \
     -d "{\"id\": 34}" \
     $LAB_REPORT_SERVICE_URL &
   curl -X POST \
     -H "Content-Type: application/json" \
     -d "{\"id\": 56}" \
     $LAB_REPORT_SERVICE_URL &
   ```

   上記のスクリプトでは、curl コマンドを使用して、Lab Report Service の URL に 3 つの異なる ID を POST で送信します。各コマンドはバックグラウンドで個別に実行されます。
   
   <br />

4. post-reports.sh スクリプトを実行可能にし、スクリプトを実行して 3 件の POST リクエストを送信します。
    
   ```
   chmod u+x post-reports.sh
   ./post-reports.sh
   ```
   
   <br />

5. 上のスクリプトによって、Lab Report Service に 3 件の POST リクエストが送信されました。ログで結果を確認します。  
   ナビゲーション メニューで [Cloud Run] をクリックします。
    
   ![650_cloudrun.png](https://lh6.googleusercontent.com/l5afE2FnxOgilYTs5nbb57o4tKK3eE99YMv4At-QiFRCk3PlHqAp_DB5HIOThslH770OnLo7u8AzPACdtCbyAEvsv3Jhj8dYaL1qj8kSDwr46BdFLSWkWRsy8ayHDm9J6T0KFKbnf9RFFgw-XMtpvw)
   
   <br />

6. 新しくデプロイした lab-report-service が表示されたら、これをクリックします。
    
   ![650_lab-report.png](https://lh4.googleusercontent.com/FuHRclwHdpvRHs7L90EDb7h5iF6BO0npwRAIa_vjHxAHfvVLZW8E47dMnHDuzCHeqDwr7wudKO-ob9vSMHKisBXJEjwJP_9PUCvR753H5k8klz9UFwQC7_vhhyt7tyDxYcvh-6ilF39Qfpek5OvsDQ)
   
   <br />

7. 次のページに、lab-report-service の詳細が表示されます。[ログ] タブをクリックします。
    
   ![650_logs.png](https://lh5.googleusercontent.com/awudO9vTJN9QqLxhqbH7fdgc7TA2pY3nT4XcBijElepI_OxbJh0P5zUO4xoR1XrKkDWxvsN95bzM1WsX2rdm_5ybGIXzt__5drjnm56QhEVOGMitroXxqJ-FGDFuuQ4qwIUYuNe0FgeMlnn00Gp5hQ)

   [ログ] ページには、先ほどスクリプトを使用して送信した 3 件の POST リクエストの結果が表示されます。  
   以下のように、OK を意味する HTTP コード 204（No Content）が返されると成功です。  
   エントリが表示されない場合は、右側にあるスクロールバーを上下に動かします。これにより、ログが再読み込みされます。

   ![650_logs-show.png](https://lh4.googleusercontent.com/IbXBuGZF_kGDKGj4EOwd7rYNkxfWJHHqJ9SaSL5synpDaHWhHBAr3GoR59iYeat8mLi3hjHDzBMlYD_VKaOwDt8XH9UWMs2I3fiLwya0ZV9-1XcN3-FCjsEifFgMSxswvNfC2z8ApUZI02BbRCGJJg)

次のセクションから、Pub/Sub を介してメッセージを受け取るサブスクライバーである SMS Service と Email Service を作成します。  
Lab Report Service が「new-lab-report」トピックにメッセージをパブリッシュすると、Pub/Sub のサブスクリプションが Push 型 (webhook) でこれらのサブスクライバー アプリケーションにメッセージを送信して実行をトリガーします。

<br />

## Email Service

ここでは、Lab Report Service から Pub/Sub を介してメッセージを受け取ってメール配信を行う Email Service を作成します。  
(ただし、このラボでは実際のメール配信は実装も実行もしません)

![650_email-svs.png](https://lh3.googleusercontent.com/5MPW7WCZ0B5opG2AakjzM_kIDS3HV1y9o3X7mu9RF4iKUkC7GrwDR6qt4EJRi3s3wBcAaGwyMqCIkoLrLNVbWemYXKilQaPpCcK6_tpfPst3UAEP7c7_tDhiuC2YHB7Lk1mLqyMEsjBm70nUo_QQEg)

<br />

### メールサービス用のコードの追加

1. メールサービスのコードが用意されたディレクトリに移動し、Pub/Sub から受信する HTTPS リクエストをコードで処理できるように必要なパッケージをインストールします。
    
   ```
   cd ~/pet-theory/lab05/email-service
   npm install express
   npm install body-parser
   ```
   
   <br />

2. nano エディタで、アプリケーションとその依存関係を記述する package.json ファイルを更新します。start 命令を追加して Cloud Run にコードの実行方法を指示します。  
   package.json ファイルを開き、scripts セクションに以下のように `"start": "node index.js"` の行を追加します。  
   編集が完了したら、Ctrl + O、Enter、Ctrl + X で nano エディタを終了します。
    
   ```
   nano package.json
   ```
   
   scripts セクションに以下のように 1 行追記する。

   ```
   "scripts": {
       "start": "node index.js",
       "test": "echo \"Error: no test specified\" && exit 1"
     },
   ```
   
   <br />  

3. nano エディタで index.js という名前の新しいファイルを作成して、以下のコードを追加します。
    
   ```
   nano index.js
   ```
   
   以下のコードをコピー & ペーストで追記する。

   ```
   const express = require('express');
   const app = express();
   const bodyParser = require('body-parser');
   app.use(bodyParser.json());
   const port = process.env.PORT || 8080;
   app.listen(port, () => {
     console.log('Listening on port', port);
   });
   app.post('/', async (req, res) => {
     const labReport = decodeBase64Json(req.body.message.data);
     try {
       console.log(`Email Service: Report ${labReport.id} trying...`);
       sendEmail();
       console.log(`Email Service: Report ${labReport.id} success :-)`);
       res.status(204).send();
     }
     catch (ex) {
       console.log(`Email Service: Report ${labReport.id} failure: ${ex}`);
       res.status(500).send();
     }
   })
   function decodeBase64Json(data) {
     return JSON.parse(Buffer.from(data, 'base64').toString());
   }
   function sendEmail() {
     console.log('Sending email');
   }
   ```

   このコードは、Pub/Sub がメッセージをサービスにパブリッシュしたときに実行され、以下の処理を行います。

   -   Pub/Sub メッセージを解読し、sendEmail() 関数の呼び出しを試みます。  
    (このラボでは実際にはメール送信せず、メール送信した旨のログ生成のみ)
    
   -   Pub/Subからのメッセージ送信によるトリガーが成功し、例外がスローされなければ、ステータス コード 204 を返し、メッセージが処理されたことを Pub/Sub に通知します。
    
   -   例外が発生した場合、サービスはステータス コード 500 を返し、メッセージが処理されなかったことを Pub/Sub に通知します。Pub/Sub は、後でサービスに再度メッセージを配信します。
   
   <br />

4. 次に、Dockerfile という名前のファイルを作成して、以下のコードを追加します。
    
   ```
   nano Dockerfile
   ```
   
   以下のコードをコピー & ペーストで追記する。

   ```
   FROM node:10
   WORKDIR /usr/src/app
   COPY package.json package*.json ./
   RUN npm install --only=production
   COPY . .
   CMD [ "npm", "start" ]
   ```
   
   <br />  

### メールサービスのビルドとデプロイ

1. Cloud Build を使用して Email Service のコンテナ イメージをビルドし、Google Container Registry に保存します。
    
   ```
   gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/email-service
   ```
   
   <br />

2. Cloud Run で Emal Service のコンテナ アプリケーションを実行します。

   ```
   gcloud run deploy email-service \
     --image gcr.io/$GOOGLE_CLOUD_PROJECT/email-service \
     --platform managed \
     --region us-central1 \
     --no-allow-unauthenticated \
     --max-instances=1
   ```
  
   デプロイが完了したら、以下のようなメッセージが表示されます。
   ```
   Service [email-service] revision [email-service-00001] has been deployed and is serving traffic at https://email-service-[hash].a.run.app
   ```

Emal Service が正常にデプロイされました。  
次に、Pub/Sub からメッセージが配信されたときに Email Service が正しくトリガーされることを確認する必要があります。

<br />

### Pub/Sub にメールサービスをトリガーさせるための設定

Pub/Sub の「new-lab-report」トピックにメッセージがパブリッシュされたときに、Email Serive をトリガーする必要があります。このためには、Cloud Run 上の Email Service メッセージを送信して実行をトリガーする権限を Pub/Sub に付与する必要があります。  
そのため、ここでは Pub/Sub に持たせるアカウントであるサービス アカウントの設定と、そのサービス アカウントへの権限付与の設定を行います (詳細は理解しなくて構いません)

![650_email-trigger.png](https://lh3.googleusercontent.com/3A5PhRwyxkqzMgDUSvfEptB7pxPvjqazeFZIo_v0-IisgVRIhIlZN-tfGQqd-K7s-KhCuvJgWZFh0kAz6fD1FTh46XXnG8c0JRmgbcT16x9eEvPfJvJIcQZaVwEWXCmZDBqq_5RxhimSmItT-xvo7Q)
<br />

1. pubsub-cloud-run-invoker という名前の新しいサービス アカウントを作成します。

   ```
   gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"
   ```
   
   <br />

2. 作成したサービス アカウントに、Email Service を呼び出す権限を付与します。

   ```    
   gcloud run services add-iam-policy-binding email-service --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --region us-central1 --platform managed
   ```

<br />

3. 次に、以下の一連のコマンドを実行して、上記で権限が付与されたサービス アカウントの認証トークンを使用して Pub/Sub のサブスクリプションが Email Service にメッセージ送信してトリガーできるように構成します。

   ```
   PROJECT_NUMBER=$(gcloud projects list --filter="qwiklabs-gcp" --format='value(PROJECT_NUMBER)')
   ```
   ```
   gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com --role=roles/iam.serviceAccountTokenCreator
   ```
   ```
   EMAIL_SERVICE_URL=$(gcloud run services describe email-service --platform managed --region us-central1 --format="value(status.address.url)")
   ```
   ```
   echo $EMAIL_SERVICE_URL
   ```
   ```
   gcloud pubsub subscriptions create email-service-sub --topic new-lab-report --push-endpoint=$EMAIL_SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
   ```  

上記最後のコマンドで、Email Service にメッセージを送信するための Cloud Pub/Sub のサブスクリプションが作成されました。

<br />

### Lab Report Service と Email Service のテスト

1. 先ほど作成したシェルスクリプトを使用して、再びメッセージを POST で送信します。

   ```
   ~/pet-theory/lab05/lab-service/post-reports.sh
   ```

   <br />

2. Cloud Console で ナビゲーション メニュー > [Cloud Run] をクリックします。
    
   ![650-email-logs.png](https://lh4.googleusercontent.com/RaqrICHrncXRfD85R2W8NdLJJJEK3ldN5eF7hGoBUqA1NzjyqoMlcO8yKKsiyibkzHvLF9CDWwkE8G2CfszKk9OFCx9x7DJ430iogMzwKbjO_HxDMyKHqCy0J_jHKq1JE_9-5XcMUyG6C0_VIE-lVw)

   リストから [email-service] > [ログ] の順にクリックします。  
   このサービスが Pub/Sub によってトリガーされた結果が表示されます。  
  
   Pub/Sub サブスクリプションからのメッセージ送信 POST リクエストに対して応答コード 204 を返すログや、”Sending email” というログが確認できます。  
   目的のメッセージが表示されない場合は、スクロールバーを上下に動かしてログを更新してください。

   ![650_log-fails-ok.png](https://lh4.googleusercontent.com/6lUt8Hx5H0LhoeSvsxyiDevnMtomUI7K-jJxrW4JtF0X5fzzX2IDEYiE5WhtJ1nyh16qi2075c20L_yIW7dPj_f2G1LqBibcHnO3Ioc6JI7NVuh4RiGswnK_pxCHQQLHs4bEr3MCYdmQxdevNxULAw)

これで、Email Service が Cloud Pub/Sub を介して Lab Report Service からメッセージを非同期で受け取り、処理を実行することができるようになりました。

<br />
<br />

## SMS Service

続いて、Lab Report Service からメッセージを受け取って Email Service とは異なる処理を並列的に実行する SMS Service を作成します。

![650_sms_arch.png](https://lh5.googleusercontent.com/8xsJsCpTNn5Yj_nGqLSSu859exUzEz3AXeYmLxfKKELb2BX7SnsdLePaNUc6wESPNlStPFHqBShSixb6sWbJu8b_mewkpQpxOl3wLU8a3MwAYD7xAHWQIv2lEFCH0shD3IFxHnp_xqvBvji9sRWF0g)

<br />

### SMS サービス用のコードの追加

1. SMS サービス用のコードが用意されたディレクトリに移動します。

   ```
   cd ~/pet-theory/lab05/sms-service
   ```
   
   <br />

2. HTTPS リクエストを受信するのに必要なパッケージをインストールします。

   ```
   npm install express
   npm install body-parser
   ```
   
   <br />

3. nano エディタで package.json ファイルを編集し、scripts セクションに以下のように `"start": "node index.js"` の行を追加します。  
   編集が完了したら、Ctrl + O、Enter、Ctrl + X で nano エディタを終了します。

   ```
   nano package.json
   ```
   
   下記のように Scripts セクションにコードを 1 行追記する。

   ```
   "scripts": {
       "start": "node index.js",
       "test": "echo \"Error: no test specified\" && exit 1"
     },
   ```
   
   <br />

4. nano エディタで index.js という名前の新しいファイルを作成して、以下のコードを追加します。  
   編集が完了したら、Ctrl + O、Enter、Ctrl + X で nano エディタを終了します。

   ```
   nano index.js
   ```
   
   下記のコードを追記する。

   ```
   const express = require('express');
   const app = express();
   const bodyParser = require('body-parser');
   app.use(bodyParser.json());
   const port = process.env.PORT || 8080;
   app.listen(port, () => {
     console.log('Listening on port', port);
   });
   app.post('/', async (req, res) => {
     const labReport = decodeBase64Json(req.body.message.data);
     try {
       console.log(`SMS Service: Report ${labReport.id} trying...`);
       sendSms();
       console.log(`SMS Service: Report ${labReport.id} success :-)`);    
       res.status(204).send();
     }
     catch (ex) {
       console.log(`SMS Service: Report ${labReport.id} failure: ${ex}`);
       res.status(500).send();
     }
   })
   function decodeBase64Json(data) {
     return JSON.parse(Buffer.from(data, 'base64').toString());
   }
   function sendSms() {
     console.log('Sending SMS');
   }
   ```
   
   <br />  

5. 次に、Dockerfile という名前のファイルを作成して、以下のコードを追加します。

   ```
   nano Dockerfile
   ```
   
   下記のコードを追記する。

   ```
   FROM node:10
   WORKDIR /usr/src/app
   COPY package.json package*.json ./
   RUN npm install --only=production
   COPY . .
   CMD [ "npm", "start" ]
   ```
   
   <br />  

### SMS サービスのデプロイ

1. Cloud Build を使用して、SMS Service のコンテナイメージをビルドし、Google Container Registry に保存します。

   ```
   gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/sms-service
   ```
   
   <br />

2. SMS Service を Cloud Run にデプロイして実行します。

   ```
   gcloud run deploy sms-service \
     --image gcr.io/$GOOGLE_CLOUD_PROJECT/sms-service \
     --platform managed \
     --region us-central1 \
     --no-allow-unauthenticated \
     --max-instances=1
   ```

   デプロイが完了したら、以下のようなメッセージが表示されます。
   ```
   Service [sms-service] revision [sms-service-00001] has been deployed and is serving traffic at https://sms-service-[hash].a.run.app
   ```

   <br />

### Cloud Pub/Sub に SMS サービスをトリガーさせるための設定

SMS Service が正常にデプロイされましたが、まだ Cloud Pub/Sub にリンクされていません。  
ここでは、先程の Email Service の時と同様に、Pub/Sub のサブスクリプションが SMS Service にメッセージを送信して実行をトリガーできるように構成します。

Lab Report Service が トピックにパブリッシュしたメッセージに対して、Email Service と SMS Service をそれぞれ並列に実行させるため、SMS Service にメッセージを送信するためのサブスクリプションを適切な権限付与をして別途構成します。

![650_sms_trigger.png](https://lh4.googleusercontent.com/Apxos5yWUhRhXnRdrn8JEsjcaLG3AlXuJFti3lBaprcoLvA6906lVD3XTdAkdnYi-21vJ7xp21C7sYofy8g2knX8t8mDkT1kUPPYsA8ScmtvWZcrUZ6_yvnAgxQBXNwrLSBczr3kR7Bde4eM2ZwYOw)
<br />

1. 下記の一連のコマンドを実行して、SMS Service をトリガーする権限を設定し、サブスクリプションを作成します。  
   権限付与の手順についてここで理解する必要はありません。  
   最後のコマンドで sms-service-sub という名前のサブスクリプションを作成します。
    
   ```
   gcloud run services add-iam-policy-binding sms-service --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --region us-central1 --platform managed
   ```
   ```
   SMS_SERVICE_URL=$(gcloud run services describe sms-service --platform managed --region us-central1 --format="value(status.address.url)")
   echo $SMS_SERVICE_URL
   ```
   ```
   gcloud pubsub subscriptions create sms-service-sub --topic new-lab-report --push-endpoint=$SMS_SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
   ```
   
   <br />

2. 再度シェルスクリプトを実行して、3 件の POST リクエストを Lab Report Service に送信します。  
   これにより、Lab Report Service は Pub/Sub のトピックにメッセージを送信し、そのトピックに紐づけられた 2 つのサブスクリプションからそれぞれ Email Service と SMS Service にメッセージが送信されます。

   ```
   ~/pet-theory/lab05/lab-service/post-reports.sh
   ```
   
   <br />

3. Cloud Console で ナビゲーション メニュー > [Cloud Run] をクリックします。  
   リストから [sms-service] > [ログ] の順にクリックします。SMS Service が Pub/Sub によってトリガーされた結果が表示されます。
    
   ![650_sms_logs.png](https://lh5.googleusercontent.com/4i-T530vnoodA-e_E8w_1sNa5vuOjJQKUztnRuOys9jTIhJ_1qJUat6JkoLSLUTA6CEAifkugo7AMSIbyklfa46N2CI-RlOre0zG90TWiAGVReVetOagf9OOkPw6Vcklo2UBD1tJBtMgV7A953ufRA)

これで、プロトタイプ システムが作成され、テストが正常に終わりました。

<br />
<br />

## システムの復元性のテスト

最後に、サブスクライバーのアプリケーションが一時的な障害でメッセージの処理に失敗しても、Cloud Pub/Sub がメッセージを保持し続け、正常に処理されるまでメッセージが失われないことを確認します。
<br />

1. Email Service のコードが用意されたディレクトリに戻ります。

   ```
   cd ~/pet-theory/lab05/email-service
   ```
   
   <br />

2. 一時的な障害をシミュレートするため、エラーを発生させるように正しくないテキストをEmail Service のコードに追加します。以下のように、index.js を編集して `sendEmail()` 関数に `throw` の行を追加します。これにより、メールサーバーが停止している場合と同様に例外がスローされるようになります。  
   編集が完了したら、Ctrl + O、Enter、Ctrl + X で nano エディタを終了します。

   ```
   nano index.js
   ```
   
   以下のようにコードを変更します。

   ```
   function sendEmail() {
     throw 'Email server is down';
     console.log('Sending email');
   }
   ```

   このコードを追加すると、サービスが呼び出されたときにクラッシュします。

   <br />

3. この不正なバージョンのメールサービスをデプロイします。  
   Cloud Build を使用して Email Service のコンテナ イメージをビルドし、Google Container Registry に保存します。

   ```
   gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/email-service
   ```
   
   <br />

4. Cloud Run で Emal Service のコンテナ アプリケーションを実行します。

   ```
   gcloud run deploy email-service \
     --image gcr.io/$GOOGLE_CLOUD_PROJECT/email-service \
     --platform managed \
     --region us-central1 \
     --no-allow-unauthenticated \
     --max-instances=1
   ```

   デプロイが完了したら、以下のようなメッセージが表示されます。
   ```
   Service [email-service] revision [email-service-00001] has been deployed and is serving traffic at https://email-service-[hash].a.run.app
   ```
   
   <br />

5. メールサービスのデプロイが正常に完了したら、Lab Report Service に再度データを POST で送信します。

   ```    
   ~/pet-theory/lab05/lab-service/post-reports.sh
   ```
   
   <br />

6. email-service のログの状況を詳しく調べます。  
   Cloud Console で ナビゲーション メニュー > [Cloud Run] をクリックします。  
   リストから [email-service] > [ログ] の順にクリックします。
    
   Email Service が呼び出されていますが、クラッシュを繰り返していることがわかります。ログを少し戻って確認すると、`Email server is down`と表示されているのがわかります。これが根本的な原因です。サービスがステータス コード 500 を返しており、Pub/Sub がサービスを繰り返し呼び出していることがわかります。

   ![650_throw](https://lh4.googleusercontent.com/l2KsH00tXiiTDjExHAZnxkplVakWRbYu8gQjrU7XgDC8yaEryP66DqQV_Jfshvmsa5__-LKNxDUX4GS0XlYIkzgkeU7cE-c1Bl44J4TTyKg2jGgDZvl_vvs_IlBK0Ivk9vVYocorM_8pjlnUUS9JtQ)

   SMS Service のログを見ると、こちらは正常に動作していることがわかります。

   <br />

7. Email Service のエラーを修正して、アプリケーションを復元します。  
   index.js ファイルを開き、先ほど入力した `throw` の行を削除してファイルを保存します。index.js の `sendEmail` 関数は以下のようになります。  
   編集が完了したら、Ctrl + O、Enter、Ctrl + X で nano エディタを終了します。

   ```
   nano index.js
   ```
   
   以下のようにコードを変更します。

   ```
   function sendEmail() {
     console.log('Sending email');
   }
   ```
   
   <br />

8. 修正したバージョンのメールサービスをデプロイします。  
   Cloud Build を使用して 修正した Email Service のコンテナ イメージをビルドし、Google Container Registry に保存します。

   ```
   gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/email-service
   ```
   
   <br />

9. Cloud Run で 修正した Emal Service のコンテナ アプリケーションを実行します。

   ```    
   gcloud run deploy email-service \
     --image gcr.io/$GOOGLE_CLOUD_PROJECT/email-service \
     --platform managed \
     --region us-central1 \
     --no-allow-unauthenticated \
     --max-instances=1
   ```

   デプロイが完了したら、以下のようなメッセージが表示されます。
   ```
   Service [email-service] revision [email-service-00001] has been deployed and is serving traffic at https://email-service-[hash].a.run.app
   ```
   
   <br />

10. デプロイが完了したら、Cloud Console で Emal Service の [ログ] 画面に戻り、画面右上隅の更新アイコンをクリックし、新しいログを確認します。
    
    ID 12、34、56 のメールが正常に送信され、Email Service がステータス コード 204 を返し、Pub/Sub がサービス呼び出しの再試行を停止したことがわかります。  
    Pub/Sub は正常にサービスを呼び出せるまで再試行を続けていたので、データは失われていません。これで堅牢なシステムの基盤ができあがりました。

    ![650-another.png](https://lh3.googleusercontent.com/grcILnoBbbA9f5IMikt_19kqOCYy-902-TxJg_wR5kxjwB0LF90O5YhE8hQTCt86oPvPjijugyAW8O2wRQUsquE1S1xMi4sD_yve85G-QzZsDplel2JIwCVt-zFDRQa_pjNCZGbUeCLCCKoeW5F5QQ)

    <br />

## 要点

1. 複数のサービスが直接お互いを呼び出すのではなく、Pub/Sub を介して相互に非同期通信を行うことで、システムの復元性を高めることができます。
    
2. Lab Report Service は、Pub/Sub のようなメッセージキューを使用することで、バックエンドの他サービスの状態を考慮せずにメッセージを送信できます。
    
3. 再試行の処理は Cloud Pub/Sub が担当するので、サービスのアプリケーションによる再試行は不要です。サービスは、成功または失敗を示すステータス コードを返すだけです。
    
4. サービスが停止した場合でも Pub/Sub が再試行を続けるので、サービスがオンラインに戻ったときにシステムを自動的に回復できます。
    
<br />

## お疲れさまでした
