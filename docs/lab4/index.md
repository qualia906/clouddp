# BokehとBigQueryによる  カスタム対話型ダッシュボードの作成


このラボでは、公開されている [Google BigQuery](https://cloud.google.com/bigquery/) データセットのデータを、Bokeh ライブラリを使用して視覚化して表示するインタラクティブ ダッシュボード アプリケーションを構築します。

  
<br />  

## 使用するサービス

- Google Kubernetes Engine（GKE）
    
    -  コンテナ オーケストレーターである Kubernetes のマネージドサービスです。
    
    -  仮想マシンをベースにした Kubernetes のマネージド クラスタを構成でき、クラスタ上で Docker コンテナのアプリケーションを実行することができます。
    
    -  このラボでは、ダッシュボード アプリケーションのコンテナと Memcache コンテナを GKE 上で実行します。
    

- Cloud Storage
    
    - フルマネージド/サーバーレスのオブジェクトストレージです。
    
    - データをファイル単位で保存することができます。
    
 - HTTP(S) ロードバランサ
    
    - L7 のロードバランサーです。
    
    - GKE 上のコンテナ アプリケーションや Cloud Storage をバックエンドにすることができます。
    

- Cloud CDN
    
    - CDN（Content Delivery Networking）のサービスです。
    
    - このラボでは、アプリケーションで使用する静的コンテンツを Cloud Storage に格納し、それらを Cloud CDN を使用して世界中のエッジロケーションにキャッシュして配信するように構成します。
    

- BigQuery
    
    - フルマネージド/サーバーレスのデータウェアハウス（分析処理に特化したデータベース）のサービスです。
    
    - このラボでは、ダッシュボード アプリケーションが BigQuery にクエリを実行し、BigQuery に保存されたデータを取得してダッシュボードに描画します。
    
<br />
  
 
## デモアプリケーションの概要

デモアプリケーションには、米国のすべての 50 州について、公開されているデータを処理して視覚化するダッシュボードが表示されます。

ユーザーは、ドロップダウン ウィジェットから 50 個の状態のいずれかを選択して、その状態に関する情報をドリルダウンして表示できます。ダッシュボードには 4 つのモジュールがあり、各モジュールにはさまざまな種類の情報が表示されます。

  

- 線図は、過去数十年にわたる様々な大気汚染物質レベルの進化を示しています。データは、[EPAの歴史的大気質データセットから得られたもの](https://cloud.google.com/bigquery/public-data/epa)です。
    
- 棒グラフは、2016 年の降水量を表示します。データは、[NOAAの世界歴史気候ネットワーク天気データセットから得られ](https://cloud.google.com/bigquery/public-data/noaa-ghcn)ます。
    
- 線図は、2016 年の気温と平均気温を示しています。[日気象データセットのNOAAグローバルサーフェスサマリからのデータ](https://cloud.google.com/bigquery/public-data/noaa-gsod)。
    
- 表には、人口が最も多い郵便番号の上位 100 項目がリストされています。以下からのデータ[、米国国勢調査局のデータセット](https://cloud.google.com/bigquery/public-data/us-census)。


実行中のデモのスクリーンショットは次のとおりです。

![](https://lh6.googleusercontent.com/HHvPA74OuPbBkU-lB-9-4Hqt64TT3S3HQ78ytwFCATtYEjs_rDJKVP-Y6urkD56u30wwirJ04onMi8fm4cX1W2u0ZmysN2lP9AcRv3nkgtXbJzGq81NdsYtyJwCaoaG09d14wSulOX9k9qRWJE7dG44)

<br />  

### 全体的なアーキテクチャ

アプリケーションのアーキテクチャは 6 つの主要な要素で構成されています。次の図は、これらのコンポーネントの相互作用を示しています。

![](https://lh5.googleusercontent.com/PNYmR_vENG5DS9VdfCuTj27vQT7vwaKKysrO8JTM6ovCTV2oeoz--vU7WHQsAPCbtNtGnTH97qBrLjGGsODx_E2IvVwn92XTL1aSd6xLJM1F8qGbrq7G2PB3u1R3Uk8SHvXV8UmAkX-vd6deJlaoy7o)


- ダッシュボード ユーザーのデスクトップ マシンとモバイル デバイス上のクライアント Web ブラウザ。
    
- [Cloud Identity Aware Proxy](https://cloud.google.com/iap/) によって管理される認証プロキシで、アプリケーションへのアクセスを制御します。※ このラボでは構成しません
    
- 着信した Web トラフィックを適切なバックエンドに効果的にルーティングし、システムに出入りするデータの SSL 復号/暗号化を処理する [HTTPS ロードバランサ](https://cloud.google.com/compute/docs/load-balancing/http/)。
    
- 動的コンテンツ（HTML およびプロットデータ）の処理を担当するBokeh ダッシュボード アプリケーションが実行される、アプリケーション バックエンド。
    
- 静的アセット バックエンドは、クライアント Web ブラウザ レンダリングに必要な静的 JavaScript ファイルと CSS ファイルを提供します。
    
<br />
  

### アプリケーションバックエンド

アプリケーション バックエンドは、ロードバランサを介してリクエストを受け取ります。そして、BigQuery から適切なデータを取り出して変換し、要求された動的コンテンツを WebSocket を介して返します。

次の図は、アーキテクチャを示しています。![](https://lh3.googleusercontent.com/fW1Ec9qyuHeXzmhcRsdxTKsOIVK6slQ6qcTSGHQHZNpi71ootLua7GCbbrmX1dPQFgySXRkZN0cOqXHL5jA3_q-V6n4mXcKIJdJC8tTyuPGWUSXpYfuNHLgCsTBD-C03Zin8ehOsFs4vdhjEbhqmBE4)

アプリケーションバックエンドには、2 つの主要なサブコンポーネントが含まれています。

- Bokeh サービス。このサービスには、Bokeh というライブラリを使用したダッシュボード アプリケーションを実行するポッド(コンテナ)が含まれており、BigQuery からデータをクエリし、クライアントの Web ブラウザで表示される動的コンテンツを生成します。
    
- Memcached サービス。頻繁に要求されるデータをキャッシュする Memcached サーバーを実行するポッド(コンテナ)を含みます。
    
これらのサブコンポーネントは Docker コンテナのアプリケーションであり、[Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) のクラスタ上にデプロイされ実行されます。

 <br /> 

## ダッシュボード アプリケーションのデプロイ  

### 環境の初期化

1. Cloud Console で Cloud Shell を開いてください。  
    Cloud Shell で次のコマンドを実行して、Git リポジトリから必要なコードをクローンします。
    
   ```
   git clone https://github.com/GoogleCloudPlatform/bigquery-bokeh-dashboard  
   cd bigquery-bokeh-dashboard
   ```

   <br />
  

2. デフォルトで使用する Google Cloud のゾーン (可用性ゾーン) を設定します。  
    ここでは、us-east1 リージョンに含まれる us-east1-c ゾーンを使用する設定にします。
    
   ```
   export ZONE=us-east1-c  
   gcloud config set compute/zone ${ZONE}
   ```
  
   <br />  

3. デモアプリケーションが BigQuery にアクセスできるようにするために、次のコマンドを実行してサービスアカウントキー (アプリケーションが使用するキー) を作成します。
    
   ※コマンドを実行するとポップアップが表示されることがありますが、「承認」をクリックしてください

   ```
   export PROJECT_ID=$(gcloud info --format='value(config.project)')
   export SERVICE_ACCOUNT_NAME="dashboard-demo"
   ```
  
   ```
   gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME --display-name="Bokeh/BigQuery Dashboard Demo"
   ```

   ```  
   gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" --role roles/artifactregistry.reader
   ```
  
    ```
   gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" --role roles/bigquery.user
   ```
  
 <br />  

### Bokeh サービスのデプロイ

ここからアプリケーションのデプロイに進みます。

まず、Docker コンテナのアプリケーションをデプロイする Kubernetes Engine のクラスタを作成します。

続いて、Bokeh を使用したダッシュボード アプリケーションを Kubernetes Engine クラスタにデプロイして実行します。

<br />
 
1. Cloud Shell で次のコマンドを実行して、2 つのノードから構成される Kubernetes Engine クラスタを作成します。
    
   ```
   gcloud container clusters create dashboard-demo-cluster \  
     --tags dashboard-demo-node \
     --service-account ${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \  
     --num-nodes 2 \  
     --zone ${ZONE}
   ```
   
   完了までに 3～4 分かかります。

   <br />

2. 後の手順でロードバランサーに割り当てる、静的 IP アドレスを作成します。
    
    ```
   gcloud compute addresses create dashboard-demo-static-ip --global
   ```
  
     <br />

3. 無料の DNS サービスである [nip.io](http://nip.io/) を使用してドメインを定義します。このサービスは、IP アドレスを含むドメインを自動的に同じ IP アドレスに解決します。
    
   ```
   export STATIC_IP=$(gcloud compute addresses describe dashboard-demo-static-ip --global --format="value(address)")  
   ```
   ```
   export DOMAIN="${STATIC_IP}.nip.io"  
   echo "My domain: ${DOMAIN}"
   ```
   ```
   kubectl create secret generic domain --from-literal domain=$DOMAIN
   ```
  
   <br />  

4. コンテナイメージの格納庫のサービスである Artifact Registry で Docker コンテナ用のリポジトリを作成します。
    
   ```  
   gcloud artifacts repositories create dashboard-demo \
     --location=us-central1 \
     --repository-format=docker
   ```
  
   <br />  

5. CI (継続的インテグレーション) のサービスである Cloud Build を使用して、Bokeh アプリケーションの Docker イメージをビルドし、上で作成した Artifact Registry にイメージを格納します。  
   (cloudbuild.yaml は Cloud Build の設定ファイルです)  
   ※ pip に関する Warning は無視してください。  
      
   ```   
   gcloud builds submit --config app/cloudbuild.yaml
   ```
  
     <br />

6. 下記のコマンドを実行して kubernetes/bokeh.yaml 内の [PROJECT_ID] を実際のプロジェクト ID の値に置き換えます。
    
   ```
   sed -i "s/\[PROJECT_ID\]/$PROJECT_ID/" kubernetes/bokeh.yaml
   ```
  
    <br /> 

7. Bokeh アプリケーションのポッド (コンテナ) を Kubernetes Engine 上にデプロイします。  
   kubectl は Kubernetes を操作するためのコマンドです。  
   (Kubernetes では、構成を定義した YAML 形式のマニフェストファイルを指定してコマンドを実行することでコンテナアプリケーションのデプロイやその他様々なオブジェクトを作成することができます)  
   ※ kubectl コマンド実行時に表示される Warning は無視してください
    
   ```
   kubectl create -f kubernetes/bokeh.yaml
   ```
  
   このコマンド実行によって、2 つのポッド (コンテナ) で冗長化して Bokeh アプリケーションがデプロイされました。（bokeh.yaml のなかでポッド数が指定されていました）

  
  

8. 2 つの Bokeh ポッドが起動し準備が整っていることを確認します。
    
   ```
   kubectl get pods
   ```
  

   次のような2つのポッドが表示されます(ポッドの NAME は環境ごとに異なります)。
   ```
   NAME READY STATUS RESTARTS AGE  
   bokeh-3952007524-9gc0l 1/1 Running 0 1m  
   bokeh-3952007524-vzn8q 1/1 Running 0 1m
   ```
  
   ポッドが 1/1 READY になるまで、最後のコマンドを繰り返します。

<br />
  
### Memcached サービスのデプロイ

1. 続いて、Bokeh アプリケーションと同様に、Memcached のポッド (コンテナ) を Kubernetes Engine クラスタ上にデプロイします。
    
   ```
   kubectl create -f kubernetes/memcached.yaml
   ```
  
   <br />

2. 2 つの Memcached ポッドが起動し準備ができていることを確認します。
    
   ```
   kubectl get pods
   ```
  
   2つのポッドが一覧表示され、次のようになります (NAME の値は環境ごとに異なります)。
   ```
   NAME READY STATUS RESTARTS AGE  
   bokeh-3952007524-9gc0l 1/1 Running 0 4m  
   bokeh-3952007524-vzn8q 1/1 Running 0 4m  
   memcached-3872116640-3bwcj 1/1 Running 0 1m  
   memcached-3872116640-t25zb 1/1 Running 0 1m
   ```
  
<br /> 

## ロードバランサの追加

上で構成したダッシュボード アプリケーションにエンドユーザーがアクセスできるように、アプリケーションをインターネットに公開するために [HTTP(S) ロードバランサ](https://cloud.google.com/compute/docs/load-balancing/http/)を構成します。  
これは3つのステップで行います


1. SSL証明書を作成します。
    
2. HTTP(S) ロードバランサを作成します。
    
3. 接続タイムアウトを延長します。
    
<br />
  

### SSL証明書の作成

1. 下記のコマンドを実行して SSL 証明書を作成します。  
   プロダクション環境では、公式[認証局](https://wikipedia.org/wiki/Certificate_authority)を通じて証明書を発行する必要があります。  
   このラボでは、次のコマンドを実行して[自己署名証明書を](https://wikipedia.org/wiki/Self-signed_certificate)発行できます。
    
   ```
   export STATIC_IP=$(gcloud compute addresses describe \  
   dashboard-demo-static-ip --global --format="value(address)")  
   export DOMAIN="${STATIC_IP}.nip.io"  
   mkdir /tmp/dashboard-demo-ssl  
   cd /tmp/dashboard-demo-ssl  
   openssl genrsa -out ssl.key 2048  
   openssl req -new -key ssl.key -out ssl.csr -subj "/CN=${DOMAIN}"  
   openssl x509 -req -days 365 -in ssl.csr -signkey ssl.key -out ssl.crt
   ```

<br />  
  

### HTTPSロードバランサの作成

1. 下記のコマンドでシェルスクリプトを実行して HTTP(S) ロードバランサを構成します。  
   (ロードバランサの構成を理解することがこのラボの目的ではないため、スクリプトを使って構成を行います)  
   ※ロードバランサの作成が完全に完了するまでには最大10分かかる場合があります。
    
   ```
   cd ~/bigquery-bokeh-dashboard/  
   ./create-lb.sh
   ```
  
   <br />  

2. 以下のコマンドを実行し、返された URL にブラウザでアクセスして、ダッシュボード アプリケーションにアクセスできることを確認します。
    
   ```
   echo https://${STATIC_IP}.nip.io/dashboard
   ```
  
   注：自己署名証明書を使用しているため、ブラウザーに警告が表示されます。この警告を無視して、ページの読み込みに進むことができます。

   ブラウザが 502 コード等でエラーを返す場合、ロードバランサのデプロイはまだ完了していません。完了するまでに数分〜十数分程度かかる場合があります。しばらく待ってから、ページを更新してください。ページが機能し、ダッシュボードが表示されるまでこれを試してください。

   SSL で暗号化されたトラフィックは HTTP(S) ロードバランサによって終端されます。  
   システムに送受信されるトラフィックは HTTPS と Secure WebSockets を使用し、ロードバランサから後ろのシステム内のトラフィックは通常の HTTP とWebSocket を使用し暗号化はされません。ほとんどのシナリオではこのレベルの暗号化で十分です。

<br />  


### 接続タイムアウトの延長

1. デフォルトでは、HTTPS ロードバランサは 30 秒以上開いているすべての接続を閉じます。ダッシュボードの WebSocket 接続を長時間開いたままにするには、接続タイムアウトを長くする必要があります。次のコマンドを実行して、タイムアウト期限を 1 日(86,400秒) に設定します。
    
   ```
   gcloud compute backend-services update dashboard-demo-service --global --timeout=86400
   ```
  
<br />  
  

## 静的コンテンツ配信のリダイレクト

Bokeh アプリケーションのポッド (コンテナアプリケーション) は、ダッシュボードのレンダリングに必要なすべての静的な JavaScript ファイルと CSS ファイルを直接処理できます。しかし、これらのポッドにこのタスクを処理させることには多くの欠点があります。 

-  Bokeh ポッドに不必要な負荷がかかり、動的プロットデータなど、他のタイプのリソースを処理する能力が潜在的に低下します。
    
-  クライアントが地理的にどこに位置しているかにかかわらず、クライアントは同じオリジンサーバーからアセットをダウンロードしなければなりません。
 

これらの欠点を軽減するには、静的アセットを効率的に処理するための専用のバックエンドを作成します。

この静的アセットバックエンドは、次の2つの主要コンポーネントを使用します。  

-  Cloud Storage のバケットで、ソース JavaScript と CSS アセットを保持します。
    
-  Cloud CDN  [は](https://cloud.google.com/cdn/)、これらのアセットを世界中のエッジロケーションに配布してキャッシュします。これにより、エンドユーザからのリクエストに対するレイテンシを短縮し、ページの読み込みを高速化します。
    
<br />

次の図は、アーキテクチャを示しています。

![](https://lh4.googleusercontent.com/-g6tM7LRg31ikiztmXWFzJCZJNV_HYeUMF-2Jqp0ECztFbDb8vLHwL-tpTa0ohaRUOJterGR9xo3PgeGOjOlZ88lcyTVF1vPv0kSDEq_sMiwiBWjDpkYY5SxHwL0Ndc_JBI8hkQWx5pgPxsh6WHiV_U)

 <br /> 
  

### 静的アセット バックエンドの作成

1. 次のコマンドを実行して、静的アセットを Bokeh コンテナーから抽出し、Cloud Shell 上の一時ディレクトリに保存します。  
   (Cloud Shell のローカル環境上で、Bokehアプリケーションのコンテナを実行し、その中の静的アセットをローカルに保存します)
    
   ```
   gcloud auth configure-docker us-central1-docker.pkg.dev --quiet
   ```
   ```
   export DOCKER_IMAGE="us-central1-docker.pkg.dev/$PROJECT_ID/dashboard-demo/dashboard-demo:latest"
   export STATIC_PATH=$(docker run $DOCKER_IMAGE bokeh info --static)  
   export TMP_PATH="/tmp/dashboard-demo-static"  
   mkdir $TMP_PATH  
   ```
   ```
   docker run $DOCKER_IMAGE tar Ccf $(dirname $STATIC_PATH) - static | tar Cxf $TMP_PATH -
   ```
  
   <br />  

2. `gsuti` コマンドで Cloud Storage のバケットを作成し、そのバケットにすべてのアセットをアップロードします。  
   (`gsutil` は Cloud Storage を操作するためのコマンドです)
    
   ```
   export BUCKET="${PROJECT_ID}-dashboard-demo"  
   ```
   ``` 
   # バケットの作成  
   gsutil mb gs://$BUCKET
   ```
   ```
   # 静的アセットファイルのアップロード
   gsutil -m cp -z js,css -a public-read -r $TMP_PATH/static gs://$BUCKET
   ```
  

   `-z` オプションを使用すると、ファイル拡張子が js と css のファイルに gzip エンコードが適用され、Cloud Storage のネットワーク帯域幅とスペースが節約されます。これにより、ストレージコストが削減され、ページの読み込み速度が向上します。  
   `-a` オプションは、すべてのクライアントがアップロードしたアセットに一般にアクセスできるように public-read のアクセス権を設定します。

   <br />  

3. Cloud CDN を有効化する `--enable-cdn` オプションを使用して下記のコマンドを実行し、前の手順でファイルを保存した Cloud Storage のバケットを HTTP(S) ロードバランサのバックエンド バケットとして割り当てます。
    
   ```
   gcloud compute backend-buckets create dashboard-demo-backend-bucket --gcs-bucket-name=$BUCKET --enable-cdn
   ```
  
<br />  

### 静的コンテンツをリダイレクトするようにロードバランサを構成する

1. `/static/*` ディレクトリ内の静的アセットをリクエストするすべてのトラフィックを Cloud Storage のバックエンド バケットにルーティングするように HTTP(S) ロードバランサを設定します。
    
   ```
   gcloud compute url-maps add-path-matcher dashboard-demo-urlmap \  
     --default-service dashboard-demo-service \  
     --path-matcher-name bucket-matcher \  
     --backend-bucket-path-rules "/static/*=dashboard-demo-backend-bucket"
   ```
  
   <br />  

2. 静的アセットの応答ヘッダーに x-goog-* パラメータが含まれていることを確認して、CDN が機能していることを確認します。
    
   ```
   curl -Iks https://${DOMAIN}/static/js/bokeh.min.js | grep x-goog
   ```
  

   応答が得られない場合は、数分待ち、再度コマンドを実行してください。設定の変更には時間がかかることがあります（5分程度かかることがあります）。

   CDN が動作したら、次のような結果が表示されます。
   ```
   x-goog-generation: 1517995207579164  
   x-goog-metageneration: 1  
   x-goog-stored-content-encoding: gzip  
   x-goog-stored-content-length: 149673  
   x-goog-hash: crc32c=oIwVmQ==  
   x-goog-hash: md5=LDj7Jj8zw8kWdnQsOShEFg==  
   x-goog-storage-class: STANDARD
   ```
  
   <br />  

3. ブラウザから Bokeh アプリケーションでいろいろな州の情報を表示してみましょう。  
   新しい州の情報を表示する場合と、一度表示したことがある州の情報を再度表示する場合とでレイテンシに違いがありますか。
    
<br />

## お疲れ様でした
