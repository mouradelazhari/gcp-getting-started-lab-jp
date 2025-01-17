# Google Kubernetes Engine (GKE) と Cloud Spanner ハンズオン

## Google Cloud プロジェクトの選択

ハンズオンを行う Google Cloud プロジェクトを作成し、 Google Cloud プロジェクトを選択して **Start/開始** をクリックしてください。

**なるべく新しいプロジェクトを作成してください。**

<walkthrough-project-setup>
</walkthrough-project-setup>

## [解説] ハンズオンの内容

### **内容と目的**

本ハンズオンでは、Google Kubernetes Engine を触ったことない方向けに、Kubernetes クラスタの作成から始め、コンテナのビルド・デプロイ・アクセスなどを行います。

Cloud Spanner にアクセスする Web アプリケーションを題材にして、Workload Identity を利用して、Service Account の鍵なしで Cloud Spanner にアクセスすることも試します。

本ハンズオンを通じて、 Google Kubernetes Engine を使ったアプリケーション開発における、最初の 1 歩目のイメージを掴んでもらうことが目的です。

次の図は、ハンズオンのシステム構成(最終構成)になります。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/0-1.png)

## Google API の有効化と Kubernetes クラスタの作成

現在 Cloud Shell と Editor の画面が開かれている状態だと思いますが、[Google Cloud のコンソール](https://console.cloud.google.com/) を開いていない場合は、コンソールの画面を開いてください。

### **利用プロジェクトを設定**
```bash
gcloud config set project {{project-id}}
```

GOOGLE_CLOUD_PROJECT にプロジェクト ID をセットします。

```bash
export GOOGLE_CLOUD_PROJECT=$(gcloud config list project --format "value(core.project)")
```

### **API の有効化**

次のコマンドで、ハンズオンで利用する Google API を有効化します。

```bash
gcloud services enable cloudbuild.googleapis.com \
  sourcerepo.googleapis.com \
  containerregistry.googleapis.com \
  cloudresourcemanager.googleapis.com \
  container.googleapis.com \
  stackdriver.googleapis.com \
  cloudtrace.googleapis.com \
  cloudprofiler.googleapis.com \
  logging.googleapis.com
```

### **Kubernetes クラスタの作成**

API を有効化したら、Kubernetes クラスタを作成します。
次のコマンドを実行してください。

```bash
gcloud container --project "$GOOGLE_CLOUD_PROJECT" clusters create "cluster-1" \
  --zone "asia-northeast1-a" \
  --enable-autoupgrade \
  --image-type "COS" \
  --enable-ip-alias \
  --workload-pool=$GOOGLE_CLOUD_PROJECT.svc.id.goog
```

Kubernetes クラスタの作成には数分かかります。

## ハンズオンで使用するスキーマの説明

今回のハンズオンでは以下のように、3 つのテーブルを利用します。これは、あるゲームの開発において、バックエンド データベースとして Cloud Spanner を使ったことを想定しており、ゲームのプレイヤー情報や、アイテム情報を管理するテーブルに相当するものを表現しています。

![スキーマ](https://storage.googleapis.com/egg-resources/egg3/public/1-1.png "今回利用するスキーマ")

このテーブルの DDL は以下のとおりです、実際にテーブルを CREATE する際に、この DDL は再度掲載します。

```sql
CREATE TABLE players (
player_id STRING(36) NOT NULL,
name STRING(MAX) NOT NULL,
level INT64 NOT NULL,
money INT64 NOT NULL,
) PRIMARY KEY(player_id);
```

```sql
CREATE TABLE items (
item_id INT64 NOT NULL,
name STRING(MAX) NOT NULL,
price INT64 NOT NULL,
) PRIMARY KEY(item_id);
```

```sql
CREATE TABLE player_items (
player_id STRING(36) NOT NULL,
item_id INT64 NOT NULL,
quantity INT64 NOT NULL,
FOREIGN KEY(item_id) REFERENCES items(item_id)
) PRIMARY KEY(player_id, item_id),
INTERLEAVE IN PARENT players ON DELETE CASCADE;
```


## Cloud Spanner インスタンスの作成

現在 Cloud Shell と Editor の画面が開かれている状態だと思いますが、[Google Cloud のコンソール](https://console.cloud.google.com/) を開いていない場合は、コンソールの画面を開いてください。


### **Cloud Spanner インスタンスの作成**

![](https://storage.googleapis.com/egg-resources/egg3/public/2-1.png)

1. ナビゲーションメニューから「Spanner」を選択

![](https://storage.cloud.google.com/egg-resources/egg3-2/public/2-2.png)

2. 「インスタンスを作成」を選択

### **情報の入力**

![](https://storage.googleapis.com/egg-resources/egg3/public/2-3.png)

以下の内容で設定して「作成」を選択します。
1. インスタンス名：dev-instance
2. インスタンスID：dev-instance
3. 「リージョン」を選択
4. 「asia-northeast1 (Tokyo) 」を選択
5. ノードの割り当て：1
6. 「作成」を選択

### **インスタンスの作成完了**
以下の画面に遷移し、作成完了です。
どのような情報が見られるか確認してみましょう。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/2-4.png)

## テーブルの作成

### **データベースの作成**

まだ Cloud Spanner のインスタンスしか作成していないので、データベース及びテーブルを作成していきます。

1つの Cloud Spanner インスタンスには、複数のデータベースを作成することができます。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/5-1.png)

![](https://storage.googleapis.com/egg-resources/egg3-2/public/5-2.png)

1. dev-instnace を選択すると画面が遷移します
2. データベースを作成を選択します

### **データベース名の入力**

![](https://storage.googleapis.com/egg-resources/egg3-2/public/5-3.png)
名前に「player-db」を入力します。


### **データベーススキーマの定義**

![](https://storage.googleapis.com/egg-resources/egg3-2/public/5-4.png)
スキーマを定義する画面に遷移します。

1. のエリアに、以下の DDL を直接貼り付けます。

```sql
CREATE TABLE players (
player_id STRING(36) NOT NULL,
name STRING(MAX) NOT NULL,
level INT64 NOT NULL,
money INT64 NOT NULL,
) PRIMARY KEY(player_id);

CREATE TABLE items (
item_id INT64 NOT NULL,
name STRING(MAX) NOT NULL,
price INT64 NOT NULL,
) PRIMARY KEY(item_id);

CREATE TABLE player_items (
player_id STRING(36) NOT NULL,
item_id INT64 NOT NULL,
quantity INT64 NOT NULL,
FOREIGN KEY(item_id) REFERENCES items(item_id)
) PRIMARY KEY(player_id, item_id),
INTERLEAVE IN PARENT players ON DELETE CASCADE;
```

2. の作成を選択すると、テーブル作成が開始します。

### **データベースの作成完了**

![](https://storage.googleapis.com/egg-resources/egg3-2/public/5-5.png)

うまくいくと、データベースが作成されると同時に 3 つのテーブルが生成されています。

## Cloud Spanner 接続クライアントの準備 

### **Cloud Spanner に書き込みをするアプリケーションのビルド**

まずはクライアント ライブラリを利用した Web アプリケーションを作成してみましょう。

Cloud Shell では、今回利用する `gke_spanner` のディレクトリにいると思います。
spanner というディレクトリがありますので、そちらに移動します。

```bash
cd spanner
```

ディレクトリの中身を確認してみましょう。

```bash
ls -la
```

`main.go` や `pkg/` という名前のファイルやディレクトリが見つかります。
これは Cloud Shell の Editor でも確認することができます。

`gke_spanner/spanner/main.go` を Editor から開いて中身を確認してみましょう。

```bash
cloudshell edit main.go
```

![](https://storage.googleapis.com/egg-resources/egg3-2/public/4-1.png)

このアプリケーションは、今回作成しているゲームで、新規ユーザーを登録するためのアプリケーションです。
実行すると Web サーバーが起動します。
Web サーバーに HTTP リクエストを送ると、自動的にユーザー ID が採番され、Cloud Spanner の players テーブルに新規ユーザー情報を書き込みます。

以下のコードが実際にその処理を行っている部分です。

```go
func (h *spanHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
        ...
		p := NewPlayers()
		// get player infor from POST request
		err := GetPlayerBody(r, p)
		if err != nil {
			LogErrorResponse(err, w)
			return
		}
		// use UUID for primary-key value
		randomId, _ := uuid.NewRandom()
		// insert a recode using mutation API
		m := []*spanner.Mutation{
			spanner.InsertOrUpdate("players", tblColumns, []interface{}{randomId.String(), p.Name, p.Level, p.Money}),
		}
		// apply mutation to cloud spanner instance
		_, err = h.client.Apply(r.Context(), m)
		if err != nil {
			LogErrorResponse(err, w)
			return
		}
		LogSuccessResponse(w, "A new Player with the ID %s has been added!\n", randomId.String())}
        ...
```

次にこの Go 言語で書かれたソースコードをビルドしてみましょう。

そして、次のコマンドでビルドをします。初回ビルド時は、依存ライブラリのダウンロードが行われるため、少し時間がかかります。
1分程度でダウンロード及びビルドが完了します。

```bash
go build -o player
```

ビルドされたバイナリがあるか確認してみましょう。
`player` というバイナリが作られているはずです。これで Cloud Spanner に接続して、書き込みを行うアプリケーションができました。

```bash
ls -la
```

**Appendix) バイナリをビルドせずに動かす方法**

次のコマンドで、バイナリをビルドせずにアプリケーションを動かすこともできます。

```bash
go run *.go
```



## データの書き込み：アプリケーション

### **Web アプリケーションから player データの追加**

設定していなければ、GOOGLE_CLOUD_PROJECT にプロジェクト ID をセットします。

```bash
export GOOGLE_CLOUD_PROJECT=$(gcloud config list project --format "value(core.project)")
```
そして、先程ビルドした `player` コマンドを実行します。

```bash
./player
```

以下の様なログが出力されれば、Web サーバーが起動しています。

```bash
2021/06/14 01:14:25 Defaulting to port 8080
2021/06/14 01:14:25 Listening on port 8080
```

次のようなログが出力された場合は `GOOGLE_CLOUD_PROJECT` の環境変数が設定されていません。

```bash
2021/06/14 18:05:47 'GOOGLE_CLOUD_PROJECT' is empty. Set 'GOOGLE_CLOUD_PROJECT' env by 'export GOOGLE_CLOUD_PROJECT=<gcp project id>'
```

環境変数を設定してから再度実行してください。
```bash
export GOOGLE_CLOUD_PROJECT=$(gcloud config list project --format "value(core.project)")
```

または
```bash
GOOGLE_CLOUD_PROJECT={{project-id}} ./player
```

この Web サーバーは、特定のパスに対して、HTTP リクエストを受け付けると新規プレイヤー情報を登録・更新・削除します。
それでは、Web サーバーに対して新規プレイヤー作成のリクエストを送ってみましょう。
`player` を起動しているコンソールとは別タブで、以下のコマンドによる HTTP POST リクエストを送ります。

```bash
curl -X POST -d '{"name": "testPlayer1", "level": 1, "money": 100}' localhost:8080/players
```

`curl` コマンドを送ると、次のような結果が返ってくるはずです。

```bash
A new Player with the ID 78120943-5b8e-4049-acf3-b6e070d017ea has been added!
```

もし **`invalid character '\\' looking for beginning of value`** というエラーが出た場合は、curl コマンド実行時に、バックスラッシュ(\\)文字を削除して改行せずに実行してみてください。

この ID(`78120943-5b8e-4049-acf3-b6e070d017ea`) はアプリケーションによって自動生成されたユーザー ID で、データベースの観点では、player テーブルの主キーになります。
以降の演習でも利用しますので、手元で生成された ID をメモなどに控えておきましょう。

## GKE クラスタの作成が完了しているかの確認

### GUI で確認

Kubernetes クラスタが作成されると、[管理コンソール] - [Kubernetes Engine] - [クラスタ] から確認できます。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/1-1.png)

Kubernetes クラスタを作成したことで、現在のシステム構成は次のようになりました。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/1-2.png)

### **コマンド設定**

次のコマンドを実行して、コマンドの実行環境を設定します。

```bash
gcloud config set project {{project-id}}
gcloud config set compute/zone asia-northeast1-a
gcloud config set container/cluster cluster-1
gcloud container clusters get-credentials cluster-1
```

上記コマンドのうち次のコマンドで、Kubernetes クラスタの操作に必要な認証情報をローカル(Cloud Shell)に持ってきています。

```bash
gcloud container clusters get-credentials cluster-1
```

次のコマンドを実行して、Kubernetes Cluster との疎通確認、バージョン確認を行います。

```bash
kubectl version
```

次のような結果が出れば、Kubernetes Cluster と疎通できています(Client と Server それぞれでバージョンが出力されている)。

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.1", GitCommit:"5e58841cce77d4bc13713ad2b91fa0d961e69192", GitTreeState:"clean", BuildDate:"2021-05-12T14:18:45Z", GoVersion:"go1.16.4", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"19+", GitVersion:"v1.19.9-gke.1900", GitCommit:"008fd38bf3dc201bebdd4fe26edf9bf87478309a", GitTreeState:"clean", BuildDate:"2021-04-14T09:22:08Z", GoVersion:"go1.15.8b5", Compiler:"gc", Platform:"linux/amd64"}
```

コマンド設定をしたことで、現在のシステム構成は次のようになりました。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/1-3.png)

## Docker コンテナイメージの作成

### **Docker コンテナイメージのビルド**

Docker コンテナイメージを作るときは、Dockerfile を用意します。
Dockerfile は spanner ディレクトリに格納されています。

```bash
cd ~/cloudshell_open/gcp-getting-started-lab-jp/fundamental/gke_spanner
```

ファイルの内容は次のコマンドで、Cloud Shell Editor で確認できます。

```bash
cloudshell edit spanner/Dockerfile
```

ターミナルに戻って、次のコマンドで Docker コンテナイメージのビルドを開始します。

```bash
docker build -t asia.gcr.io/${GOOGLE_CLOUD_PROJECT}/spanner-app:v1 ./spanner
```

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/2-1.png)

Docker コンテナイメージのビルドには数分かかります。

次のコマンドでローカル(Cloud Shell)上のコメントイメージを確認できます。

```bash
docker image ls
```

次のように、コンテナイメージが出力されているはずです。

```
REPOSITORY                                       TAG  IMAGE ID       CREATED         SIZE
asia.gcr.io/<GOOGLE_CLOUD_PROJECT>/spanner-app   v1   8952a9a242f5   5 minutes ago   23.2MB
```

### **Docker コンテナイメージを Push**

コンテナイメージのビルドができたら、ビルドしたイメージを Google Container Registry にアップロードします。

次のコマンドで、`docker push` 時に `gcloud` の認証情報を使うように設定します。

```bash
gcloud auth configure-docker
```

次のコマンドで、Google Container Registry に Push します。
(初回は時間がかかります)

```bash
docker push asia.gcr.io/${GOOGLE_CLOUD_PROJECT}/spanner-app:v1
```

次のコマンドで Google Container Registry に Push されたことを確認します。

```bash
gcloud container images list-tags asia.gcr.io/${GOOGLE_CLOUD_PROJECT}/spanner-app
```

管理コンソール > [ツール] > [Container Registry] からも確認できます。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/2-2.png)

Google Container Registry にコンテナイメージを追加したことで、現在のシステム構成は次のようになりました。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/2-3.png)

## Workload Identity 設定

### **Workload Identity が有効化されていることを確認**

GKE クラスタで Workload Identity が有効になっていることを確認します。
次のコマンドを実行してください。

```bash
gcloud container clusters describe cluster-1 \
  --format="value(workloadIdentityConfig.workloadPool)" --zone asia-northeast1-a
```

`<GOOGLE_CLOUD_PROJECT>.svc.id.goog` と出力されていれば、正しく設定されています。

またクラスタとは別に、NodePool で Workload Identity が有効になっていることを確認します。
次のコマンドを実行してください。

```bash
gcloud container node-pools describe default-pool --cluster=cluster-1 \
  --format="value(config.workloadMetadataConfig.mode)" --zone asia-northeast1-a
```

`GKE_METADATA` と出力されていれば、正しく設定されています。

### **Service Account 作成**

Workload Identity を利用する Service Account を作成します。
Kubernetes Service Account(KSA) と Google Service Account(GSA) を作成します。

次のコマンドで、Kubernetes クラスタ上に Kubernetes 用の Service Account(KSA) を作成します。

```bash
kubectl create serviceaccount spanner-app
```

次のコマンドで、Google Service Account(GSA)を作成します。

```bash
gcloud iam service-accounts create spanner-app
```

今回利用する Spanner App は Cloud Spanner にアクセスする Web アプリケーションです。
Cloud Spanner の Database に対してデータの操作(追加・更新・削除など)を行えるように、`roles/spanner.databaseUser` の権限を付与する必要があります。
次のコマンドで、Service Account に Cloud Spanner Database に対するデータ操作が行える権限を追加します。

```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --role "roles/spanner.databaseUser" \
  --member serviceAccount:spanner-app@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

この時点で、KSAとGSAはそれぞれ別のものとして作成だけします。KSAとGSAの紐付け作業は次の作業で行います。

### **Service Account の紐付け**

Kubernetes Service Account が Google Service Account の権限を借用できるように、紐付けを行います。
次のコマンドで紐付けを行います。

```bash
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:$GOOGLE_CLOUD_PROJECT.svc.id.goog[default/spanner-app]" \
  spanner-app@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

次のコマンドで、Google Service Account の情報を Kubernetes Service Account のアノテーションに追加します。

```bash
kubectl annotate serviceaccount spanner-app \
  iam.gke.io/gcp-service-account=spanner-app@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com
```

## Kubernetes Deployment の作成

ここからは Kubernetes クラスタ上で動かすリソースを作成していきます。

### **Deployment の Manifest ファイルを編集する** 

Manifest ファイルは k8s ディレクトリに格納されています。
ファイルの内容は次のコマンドで、Cloud Shell Editor で確認できます。

```bash
cloudshell edit k8s/spanner-app-deployment.yaml
```

**ファイル中の project id を自分の {{project-id}} に変更します。**

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/3-1.png)


### **Deployment を作成する**

次のコマンドで、Kubernetes クラスタ上に Deployment を作成します。

```bash
kubectl apply -f k8s/spanner-app-deployment.yaml
```

作成された Deployment、Pod を次のコマンドで確認します。

```bash
kubectl get deployments
```

```bash
kubectl get pods -o wide
```

Deployment を作成したことで、現在のシステム構成は次のようになりました。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/4-1.png)

**Appendix) kubectl コマンドリファレンス その１**

 * Pod を一覧で確認したいとき
```bash
kubectl get pods
```

 * Pod の IP アドレスや実行 Node を知りたいとき
```bash
kubectl get pods -o wide
```

 * Pod の詳細を見たいとき
```bash
kubectl describe pods <pod name>
```

 * Pod の定義を YAML で取得したいとき
```bash
kubectl get pods <pod name> -o yaml
```


## Kubernetes Service (Discovery) の作成

### **Service を作成する**

Manifest ファイルは k8s ディレクトリに格納されています。
ファイルの内容は次のコマンドで、Cloud Shell Editor で確認できます。

```bash
cloudshell edit k8s/spanner-app-service.yaml
```

次のコマンドで、Kubernetes クラスタ上に Service を作成します。

```bash
kubectl apply -f k8s/spanner-app-service.yaml
```

次のコマンドで、作成された Service を確認します。

```bash
kubectl get svc
```

ネットワークロードバランサー(TCP)を作成するため、外部 IP アドレス(EXTERNAL-IP)が割り当てられるまでにしばらく(数分)時間がかかります。
次のコマンドで、-w フラグを付けると変更を常に watch します(中断するときは Ctrl + C)

```bash
kubectl get svc -w
```


**Appendix) kubectl コマンドリファレンス その2**

 * Service を一覧で確認したいとき
```bash
kubectl get services
```

 * Service が通信をルーティングしている Pod(のIP) を知りたいとき
 * endpointsリソースはServiceリソースによって自動管理されます
```bash
kubectl get endpoints
```

 * Service の詳細を見たいとき
```bash
kubectl describe svc <service name>
```

 * Service の定義を YAML で取得したいとき
```bash
kubectl get svc <service name> -o yaml
```

### **Service にアクセスする**

`Type: LoadBalancer` の Service を作成すると、ネットワークロードバランサーが払い出されます。
[管理コンソール] - [ネットワーキング] - [ネットワークサービス] - [負荷分散] から確認できます。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/5-1.png)

ロードバランサーの外部 IP アドレスを指定して、実際にアクセスしてみましょう。
次のコマンドで、サービスの外部IPアドレスを確認します。

```bash
kubectl get svc
```

次の出力のうち、`EXTERNAL-IP` が外部 IP アドレスになります。
環境ごとに異なるため、自分の環境では読み替えてください。

```
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)          AGE
spanner-app  LoadBalancer   10.0.12.198   203.0.113.245  8080:30189/TCP   58s
kubernetes   ClusterIP      10.56.0.1     <none>         443/TCP          47m
```

次の curl コマンドでアクセスして、Players 情報取得します。
<EXTERNAL IP> の部分は上記で確認した IP アドレスに変更してください。

```bash
curl <EXTERNAL IP>:8080/players
```

curl コマンドの結果、アクセスできれば、Service リソースと Deployment(そしてPod)は正しく機能しています。

また、Workload Identity の機能を使うことで、Service Account の鍵なしで Cloud Spanner にアクセスできることも確認できました。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/5-2.png)

### **Spanner App を操作する**

 * Player 新規追加(playerId はこの後、自動で採番される)
```bash
curl -X POST -d '{"name": "testPlayer99", "level": 9, "money": 10000}' <EXTERNAL IP>:8080/players
```

もし **`invalid character '\\' looking for beginning of value`** というエラーが出た場合は、curl コマンド実行時に、バックスラッシュ(\\)文字を削除して改行せずに実行してみてください。

 * Player 一覧取得
```bash
curl <EXTERNAL IP>:8080/players
```

 * Player 更新(playerId は適宜変更すること)
```bash
curl -X PUT -d '{"playerId":"afceaaab-54b3-4546-baba-319fc7b2b5b0","name": "testPlayer1", "level": 2, "money": 200}' <EXTERNAL IP>:8080/players
```

 * Player 削除(playerId は適宜変更すること)
```bash
curl -X DELETE http://<EXTERNAL IP>:8080/players/afceaaab-54b3-4546-baba-319fc7b2b5b0
```

**Appendix) kubectl を使ったトラブルシューティング**

Kubernetes 上のリソースで問題が発生した場合は `kubectl describe` コマンドで確認します。
Kubernetes は リソースの作成・更新・削除などを行うと `Event` リソースを発行します。 `kubectl describe` コマンドで1時間以内の `Event` を確認できます。

```bash
kubectl describe <resource name> <object name>
# 例
kubectl describe deployments hello-node
```

また、アプリケーションでエラーが発生していないかは、アプリケーションのログから確認します。
アプリケーションのログを確認するには `kubectl logs` コマンドを使います。

```bash
kubectl logs -f <pod name>
```

## Cleanup

すべての演習が終わったら、リソースを削除します。

 1. ロードバランサーを削除します。Disk と LB は先に消さないと Kubernetes クラスタごと削除しても残り続けるため、先に削除します
```bash
kubectl delete svc spanner-app
```

 2. Container Registry に格納されている Docker イメージを削除します
```bash
gcloud container images delete asia.gcr.io/$PROJECT_ID/spanner-app:v1 --quiet
```

 3. Kubernetes クラスタを削除します
```bash
gcloud container clusters delete "cluster-1" --zone "asia-northeast1-a"
```

`Do you want to continue (Y/n)?` には `y` を入力します。

 4. Google Service Account を削除します
```bash
gcloud iam service-accounts delete spanner-app@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

`Do you want to continue (Y/n)?` には `y` を入力します。

 5. Cloud Spanner Instance を削除します。
```bash
gcloud spanner instances delete dev-instance
```

`Do you want to continue (Y/n)?` には `y` を入力します。

## **Thank You!**

以上で、今回の Google Kubernetes Engine と Cloud Spanner のハンズオンは完了です。