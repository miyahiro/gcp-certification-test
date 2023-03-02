# Cloud Developer Course Qwiklab のメモ

## Google Cloud Fundamentals: Getting Started with GKE

環境変数にゾーンを設定

```sh
export MY_ZONE=us-central1-a
```

GKEクラスタを作成

```sh
gcloud container clusters create webfrontend --zone $MY_ZONE --num-nodes 2
```

Kubenetesのバージョンを確認

```sh
kubectl version
```

クラスタにDeploymentをイメージとしてnginx:1.17.10を指定して作成.

```sh
kubectl create deploy nginx -- image=nginx:1.17.10

kubectl get pods
```

作成したDeploymentへ80/tcpでアクセスできるようにLoadBalancerタイプのServiceを作成し、そのサービスのバックエンドとして作成したDeploymentを利用することを指定.

```sh
kubectl expose deployment nginx --port 80 --type LoadBalancer

kubectl get services
```

作成したDeploymentで動作させるPodの数を3に変更

```sh
kubectl scale deployment nginx --replicas 3

kubectl get pods
```

作成したServiceを確認し、LoadBalancerの外部IPアドレスを知る

```sh
kubectl get services
```

LoadBalancerの外部IPアドレスにアクセスすると、バックエンドのDeployment内のPodがレスポンスを返すことを確認.

## Hello Cloud Run APPRUN

gcloudコマンドの設定、環境変数の設定を行う

```sh
gcloud config set compute/region us-central1

LOCATION="us-central1"
```

Dockerイメージ作成に必要なファイルを作成

- package.json
- index.js
- Dockerfile

Cloud Buildを使ってDockerイメージを作成し、プロジェクトのArtifact Repositryに格納

```sh
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld

gcloud container images list
```

ローカルで確認

```sh
docker run -d -p 8080:8080 gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld
```

Artfact Repositryに格納されているDockerイメージを指定してCloud Runリソースを作成
`--allow-unauthenticated`としているため、認証されていなくてもアクセスできる

作成したCloud Runリソース用のURLを得られる

```sh
gcloud run deploy --image gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld --allow-unauthenticated --region=$LOCATION
```

Cloud Runリソース用のURLにアクセス
(`https://<cloud-run-service-name>-<project-id>-<location>.run.app`)

作成したCloud Runリソースを削除

```sh
gcloud container images delete gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld
```

Artifact Repositryから作成したDockerイメージを削除.

```sh
gcloud beta run services delete helloworld
```

## Automating the Deployment of Infrastructure Using Terraform

```sh
terraform --version

mkdir tfinfra

cd tfinfra
```

make terraform file named provider.tf

```tf
provider "google" {}
```

```sh
terraform init
```

make terraform file named mynetwork.tf

```tf
# Create the mynetwork network
resource "google_compute_network" "mynetwork" {
  name = "mynetwork"
  # RESOURCE properties go here
  auto_create_subnetworks = "true"
}
# Add a firewall rule to allow HTTP, SSH, RDP and ICMP traffic on mynetwork
resource "google_compute_firewall" "mynetwork-allow-http-ssh-rdp-icmp" {
  name = "mynetwork-allow-http-ssh-rdp-icmp"
  # RESOURCE properties go here
  network = google_compute_network.mynetwork.self_link
  allow {
    protocol = "tcp"
    ports    = ["22", "80", "3389"]
  }
  allow {
    protocol = "icmp"
  }
  source_ranges = ["0.0.0.0/0"]
}
# Create the mynet-us-vm instance
module "mynet-us-vm" {
  source           = "./instance"
  instance_name    = "mynet-us-vm"
  instance_zone    = "us-central1-a"
  instance_network = google_compute_network.mynetwork.self_link
}
# Create the mynet-eu-vm" instance
module "mynet-eu-vm" {
  source           = "./instance"
  instance_name    = "mynet-eu-vm"
  instance_zone    = "europe-west1-d"
  instance_network = google_compute_network.mynetwork.self_link
}
```

```sh
mkdir instance
cd instance
```

make terraform file named main.tf

```tf
resource "google_compute_instance" "vm_instance" {
  name = "${var.instance_name}"
  # RESOURCE properties go here
  zone         = "${var.instance_zone}"
  machine_type = "${var.instance_type}"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
      }
  }
    network_interface {
    network = "${var.instance_network}"
    access_config {
      # Allocate a one-to-one NAT IP to the instance
    }
  }
}
```

```sh
terraform fmt
```

## アプリ開発 - Kubernetes Engine へのアプリケーションのデプロイ: Node.js

Enable Kubenetes Engine API and Container Registry API.

Create K8S Cluster named quiz-cluster

`default-poll(node pool setting) => security => access scope`

"すべての Cloud API に完全アクセス権を許可]"

Create 2 Dockerfile

```sh
gcloud builds submit -t gcr.io/$DEVSHELL_PROJECT_ID/quiz-frontend ./frontend/

gcloud builds submit -t gcr.io/$DEVSHELL_PROJECT_ID/quiz-backend ./backend/
```

Craete 2 Deployment setting file(frontend-deployment.yaml, backend-deployment.yaml)

```sh
kubectl create -f ./frontend-deployment.yaml
kubectl create -f ./backend-deployment.yaml
```

```sh
# apply service setting(Loadbalancer)
kubectl create -f ./frontend-service.yaml
```

Access to created Service external ip via web brawser

## アプリ開発 - App Engine フレキシブル環境へのアプリケーションのデプロイ: Python

```sh
git clone https://github.com/GoogleCloudPlatform/training-data-analystp

ln -s ~/training-data-analyst/courses/developingapps/v1.3/python/appengine ~/appengine

cd ~/appengine/start

. prepare_environment.sh
```

create frontend/app.yaml

```yaml
runtime: python
env: flex
manual_scaling:
  instances: 1
entrypoint: gunicorn -b 0.0.0.0:8080 quiz:app
runtime_config:
  python_version: 3
env_variables:
  GCLOUD_BUCKET: "[GCLOUD_PROJECT]-media"
``

```sh
gcloud config set app/cloud_build_timeout 1800

gcloud app deploy ./frontend/app.yaml
```

modify file frontend/quiz/webapp/template/home.html

```sh
gcloud app deploy ./frontend/app.yaml --no-promote \
--no-stop-previous-version
```

See App Engine app Version.

2 Version exists.

Split App Engine app traffice 50% to old versoon app and 50% to new version app.

## ネットワーク ロードバランサと HTTP ロードバランサを設定する

```sh
gcloud config set compute/zone us-east1-d

gcloud config set compute/region us-east1
```

### ネットワークロードバランサ設定

```sh
gcloud compute instances create www1 \
    --zone=us-east1-d \
    --tags=network-lb-tag \
    --machine-type=e2-medium \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'

gcloud compute instances create www2 \
    --zone=us-east1-d \
    --tags=network-lb-tag \
    --machine-type=e2-medium \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www2</h3>" | tee /var/www/html/index.html'

gcloud compute instances create www3 \
    --zone=us-east1-d \
    --tags=network-lb-tag \
    --machine-type=e2-medium \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www3</h3>" | tee /var/www/html/index.html'
```

`network-lb-tag`という名前のタグが付けられたVMに対して`80/tcp`のアクセスを許可するfirewallルールを`www-firewall-network-lb`という名前で作成する.

```sh
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80
```

```sh
gcloud compute instances list
```

`network-lb-ip-1`という名前の静的なIPアドレスをリージョン`us-east1`に作成

```sh
gcloud compute addresses create network-lb-ip-1 \
    --region us-east1
```

`basic-check`という名前のhttp-health-chekcsヘルスチェックを作成

```sh
gcloud compute http-health-checks create basic-check
```

`www-pool`という名前のtarget-poolsを`us-east1`リージョンにヘルスチェックとして`basic-check`を指定して作成

```sh
gcloud compute target-pools create www-pool \
    --region us-east1 --http-health-check basic-check
```

`www-pool`という名前のtarget-poolsにターゲット仮想サーバとして`www1`,`www2`,`www3`を追加

```sh
gcloud compute target-pools add-instances www-pool \
    --instances www1,www2,www3
```

`wwww-rule`という名前のforwaring-rulesをリージョン`us-east1`にポート`80`、アドレス`network-lb-ip-1`、ターゲットプール`www-pool`を指定して作成する.

```sh
gcloud compute forwarding-rules create www-rule \
    --region  us-east1 \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool
```

これにより`http://<network-lb-ip-1>/`というURLでアクセスできるようになる.

```sh
gcloud compute forwarding-rules describe www-rule --region us-east1

IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region us-east1 --format="json" | jq -r .IPAddress)

echo $IPADDRESS

while true; do curl -m1 $IPADDRESS; done
```

### HTTP ロードバランサを作成する

`lb-ipv4-1`というIPアドレスリソースを作成
`--global`を指定しているので、作成したIPアドレスはエニーキャスト対応のIPアドレスとなる.
このIPアドレスはロードバランサーで使用する.

```sh
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global

# IPアドレスを確認
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global
```

`lb-backend-template`という名前のinstance-templatesを作成する

```sh
gcloud compute instance-templates create lb-backend-template \
   --region=us-east1 \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
```

`la-backend-group`という名前でinstance-groupsをインスタンステンプレートとして`lb-backend-templete`を指定して作成する.

```sh
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=us-east1-d
```

`fw-allow-health-ckeck`という名前のfirewall-rulesを作成する
このルールの`--source-range`で指定しているアドレスは、Google Cloudヘルスチェックシステムからのトラフィックの送信元である.

```sh
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```

`http-basic-check`という名前でhealth-checksを作成、チェックするポート番号は80/tcp.

```sh
gcloud compute health-checks create http http-basic-check \
  --port 80
```

`web-backend-service`という名前でbackend-servicesを作成する.
`--health-checks`でヘルスチェックとして`http-basic-check`という名前のhealth-checksの使用を指定
`--global`を指定することでグローバルで有効なバックエンドサービスとして作成することを指定

```sh
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global
```

バックエンドサービス`web-backend-service`にinstance-groupインスタンス`la-backend-group`を追加
`--instance-gorup-zone`でインスタンスグループのゾーンを指定
`--global`で何を指定しているかよく分からないが、どのリージョンからのトラフィックも`lb-backend-group`に流すこと死しているのかも

```sh
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=us-east1-d \
  --global
```

`web-map-http`とい名前のurl-mapsを作成
`--default-service`デフォルトで使用するバックエンドを指定

```sh
gcloud compute url-maps create web-map-http \
    --default-service web-backend-service
```

`http-lb-proxy`という名前のtarget-http-proxiesをurl-mapsとして`web-map-http`を指定して作成

```sh
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http
```

`http-content-rule`という名前のforwarding-rulesをtarget-http-proxiesインスタンス`web-lb-proxy`を指定して作成
`--address`で受け付けるIPアドレス
`--global`でグローバルリソースとして配置
`--ports`で受け付けるポートを指定

反映までに3-5分かかる

```sh
gcloud compute forwarding-rules create http-content-rule \
    --address=lb-ipv4-1\
    --global \
    --target-http-proxy=http-lb-proxy \
    --ports=80
```

この時点でLoad Balancerが作成されていることになる.
GCPでは明示的なLoad Blancerが存在しているわけではなく、適した`forwarding-rules`なのか？

## クラウドのリソースを作成、管理する: チャレンジラボ

### GKEクラスタを作成、Deployment作成、LoadBalancer Serviceを作成

```sh
CLUSTER_NAME=
ZONE=
```

GKEクラスタを`$CLUSTER_NAME`という名前でゾーン`$ZONE`に作成

```sh
gcloud container clusters create $CLUSTER_NAME --zone $ZONE --num-nodes 2
```

GKEクラスアにアクセスするceredentialを取得し保存

```sh
gcloud container clusters get-credentials --zone $ZONE $CLUSTER_NAME
```

Deploymentインスタンスを作成

```sh
kubectl create deployment nucleus-deployment-1 --image=gcr.io/google-samples/hello-app:2.0
```

Serviceインスタンスを作成

```sh
kubectl expose deployment nucleus-deployment-1 --type=LoadBalancer --port 8081
```

### グローバルhttp load balancerを作成

```sh
ZONE=
REGION=
FW_RULE_NAME=
```

インスタンステンプレートを作成

```sh
gcloud compute instance-templates create nucleus-nginx-instance-template \
   --region=$REGION \
   --network=default \
   --subnet=default \
   --tags=frontend-http-server \
   --machine-type=f1-micro \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- '\''s/nginx/Google Cloud Platform - '\''"\$HOSTNAME"'\''/'\'' /var/www/html/index.nginx-debian.html
'
```

マネージドインスタンスグループをインスタンステンプレートから作成

```sh
gcloud compute instance-groups managed create nucleus-nginx-instance-group \
   --template=nucleus-nginx-instance-template --size=2 --zone=$ZONE
```

特定のタグが付けられたインスタンスに80/tcpを通すようにFirewallのルールを作成

```sh
gcloud compute firewall-rules create $FW_RULE_NAME \
    --target-tags frontend-http-server --allow tcp:80
```

インスタンスがhttpサーバとして動作しているかを確認するためのヘルスチェックを作成

```sh
gcloud compute health-checks create http nucleus-health-checks \
  --port 80
```

http用のバックエンドサービスを作成
ヘルスチェックとして前のステップで作成したヘルスチェックを指定

```sh
gcloud compute backend-services create nucleus-backend-services \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=nucleus-health-checks \
  --global
```

バックエンドサービスに以前のステップで作成したマネージドインスタンスグループを追加

```sh
gcloud compute backend-services add-backend nucleus-backend-services \
  --instance-group=nucleus-nginx-instance-group \
  --instance-group-zone=$ZONE \
  --global
```

target-http-proxiesに指定するurl-mapsを作成
処理するバックエンドを作成したバックエンドとする

```sh
gcloud compute url-maps create nucleus-url-maps \
    --default-service nucleus-backend-services
```

前のステップで作成したurl-mapsを指定してtarget-http-proxiesを作成

```sh
gcloud compute target-http-proxies create nucleus-target-http-proxies \
    --url-map nucleus-url-maps
```

グローバルhttp load balancer用のIPアドレスを作成する
`--global`とすることでエニーキャストができるようになる？

```sh
gcloud compute addresses create nucleus-addresses \
  --ip-version=IPV4 \
  --global
```

以前のステップ作成したIPアドレス、target-http-proxiese、リッスンポート(80/tcp)を指定してforwarding-rulesを作成
`--global`としているのでグローバルリソースになる？

このルールを作成するとそのルールを実現するために必要なhttp load balancerが内部的に勝手に作られる？

```sh
gcloud compute forwarding-rules create nucleus-forwarding-rules \
    --address=nucleus-addresses \
    --global \
    --target-http-proxy=nucleus-target-http-proxies \
    --ports=80
```

## gcloud services enable run.googleapis.com

Cloud Run APIを有効化

```sh
gcloud services enable run.googleapis.com
```

gcloudのデフォルトのリージョンを設定、その他環境変数を設定

```sh
gcloud config set compute/region us-central1

LOCATION="us-central1"
```

Dockerイメージ作成用のファイルを作成

```sh
mkdir helloworld && cd helloworld
```

package.jsonを作成

```json
{
  "name": "helloworld",
  "description": "Simple hello world sample in Node",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "author": "Google LLC",
  "license": "Apache-2.0",
  "dependencies": {
    "express": "^4.17.1"
  }
}
```

index.jsを作成

```js
const express = require('express');
const app = express();
const port = process.env.PORT || 8080;
app.get('/', (req, res) => {
  const name = process.env.NAME || 'World';
  res.send(`Hello ${name}!`);
});
app.listen(port, () => {
  console.log(`helloworld: listening on port ${port}`);
});
```

Dockerfileを作成

```Dockerfile
# Node.js 12 の公式の軽量イメージを使用します。
# https://hub.docker.com/_/node
FROM node:12-slim
# app ディレクトリを作成してそのディレクトリに移動します。
WORKDIR /usr/src/app
# アプリケーション依存関係マニフェストをコンテナ イメージにコピーします。
# package.json と package-lock.json の両方がコピーされるようにワイルドカードを使用します（利用可能な場合）。
# これを最初にコピーしておくと、コードを変更するたびに npm install を再実行する必要がなくなります。
COPY package*.json ./
# 本番環境の依存関係をインストールします。
# package-lock.json を追加した場合、「npm ci」に切り替えることでビルドを高速化します。
# RUN npm ci --only=production
RUN npm install --only=production
# ローカルコードをコンテナ イメージにコピーします。
COPY . ./
# コンテナの起動時にウェブサービスを実行します。
CMD [ "npm", "start" ]
```

Dockerイメージを作成し、プロジェクト専用のArtifact Registryに格納

```sh
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld

gcloud container images list
```

ローカルで試す

```sh
docker run -d -p 8080:8080 gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld
```

Cloud Runリソースを作成

```sh
gcloud run deploy --image gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld --allow-unauthenticated --region=$LOCATION
```

デプロイされたときに提示されたURLにアクセス

## Creating Application Containers with Google Cloud Buildpacks [APPRUN]

Cloud Run APIを有効化する

```sh
gcloud services enable run.googleapis.com
```

gcloudコマンドの設定

```sh
gcloud config set compute/region us-central1
```

アプリケーションのソースコードをgit cloneする

```sh
git clone https://github.com/GoogleCloudPlatform/buildpack-samples.git

cd buildpack-samples/sample-node
```

GCPのWeb Console上にpackのCLIツールが既にインストールされているので、packコマンドでイメージを作成できる
Dockerfileを作らなくてもいい感じのDockerイメージを作ってくれる

```sh
pack build --builder=gcr.io/buildpacks/builder sample-node
```

ローカル環境で試す

```sh
docker run -it -e PORT=8080 -p 8080:8080 sample-node
```

Cloud Runにソースを元にDockerイメージを作成し、更にデプロイまでする
Artifact Registryにも内部的には格納されているのか？

```sh
gcloud beta run deploy --source .
```

この時点でCloud Runのリソースが出来上がるので、提示されるURLでアクセスできるようになる

## Traffic Management with Cloud Run [APPRUN]

Cloud Run APIを有効化する

```sh
gcloud services enable run.googleapis.com
```

gcloudコマンドの設定、環境変数設定

```sh
gcloud config set compute/region us-central1

LOCATION="us-central1"
```

既存のArtifact Repositry上のイメージを使用してCloud Runのリソース`product-service`を作成
`--tag test1`よりこのリビジョンのタグは`test1`

```sh
gcloud run deploy product-service \
   --image gcr.io/qwiklabs-resources/product-status:0.0.1 \
   --tag test1 \
   --region $LOCATION \
   --allow-unauthenticated

TEST1_PRODUCT_SERVICE_URL=$(gcloud run services describe product-service --platform managed --region us-central1 --format="value(status.address.url)")

curl $TEST1_PRODUCT_SERVICE_URL/help -w "\n"
```

Cloud Runリソース`product-service`の新しいリビジョンを作成
但し、`--no-traffic`としているのでこのリビジョンにはトラフィックは流れない
`--tag test2`よりこのリビジョンのタグは`test2`

```sh
gcloud run deploy product-service \
  --image gcr.io/qwiklabs-resources/product-status:0.0.2 \
  --no-traffic \
  --tag test2 \
  --region=$LOCATION \
  --allow-unauthenticated
```

しかし、このリビジョン専用のURLが既に出来上がっているので、それを使えばアクセス可能
(リビジョンごとのURLと、Cloud RunリソースのURLは別物)

```sh
TEST2_PRODUCT_STATUS_URL=$(gcloud run services describe product-service --platform managed --region=us-central1 --format="value(status.traffic[2].url)")

curl $TEST2_PRODUCT_STATUS_URL/help -w "\n"
```

Cloud Runリソース`product-service`のタグが`test2`のリビジョンのものにトラフィックを50%行くようにする

```sh
gcloud run services update-traffic product-service \
  --to-tags test2=50 \
  --region=$LOCATION
```

`test1`と`test2`で半々のトラフィックとなっている

```sh
for i in {1..10}; do curl $TEST1_PRODUCT_SERVICE_URL/help -w "\n"; done
```

タグが`test2`のリビジョンにトラフィックを行かないようにする
(あくまでCloud Runリソース`product-service`用のURLであってリビジョン専用のURLではないことに注意)

```sh
gcloud run services update-traffic product-service \
  --to-tags test2=0 \
  --region=$LOCATION
  
for i in {1..10}; do curl $TEST1_PRODUCT_SERVICE_URL/help -w "\n"; done
```

タグが`test3`のリビジョンを作成

```sh
gcloud run deploy product-service \
  --image gcr.io/qwiklabs-resources/product-status:0.0.3 \
  --no-traffic \
  --tag test3 \
  --region=$LOCATION \
  --allow-unauthenticated
```

タグが`test4`のリビジョンを作成

```sh
gcloud run deploy product-service \
  --image gcr.io/qwiklabs-resources/product-status:0.0.4 \
  --no-traffic \
  --tag test4 \
  --region=$LOCATION \
  --allow-unauthenticated
```
  
リビジョン一覧を確認

```sh
gcloud run services describe product-service \
  --region=$LOCATION \
  --format='value(status.traffic.revisionName)'
```

リビジョン指定でトラフィックを指定

```sh
LIST=$(gcloud run services describe product-service --platform=managed --region=$LOCATION --format='value[delimiter="=25,"](status.traffic.revisionName)')"=25"

gcloud run services update-traffic product-service \
  --to-revisions $LIST --region=$LOCATION

#4つのリビジョンに25%づつのトラフィックが分散されていることを確認

for i in {1..10}; do curl $TEST1_PRODUCT_SERVICE_URL/help -w "\n"; done
```
  
最新のリビジョンを使うように構成を変更

```sh
gcloud run services update-traffic product-service --to-latest --platform=managed --region=$LOCATION

#最終リビジョンのみが動く状態なっていることを確認
LATEST_PRODUCT_STATUS_URL=$(gcloud run services describe product-service --platform managed --region=$LOCATION --format="value(status.address.url)")
curl $LATEST_PRODUCT_STATUS_URL/help -w "\n"
```

## Implementing Least Privilege IAM Policy Bindings in Cloud Run [APPRUN]

本プロジェクトでCloud Run APIを有効化する

```sh
gcloud services enable run.googleapis.com
```

gcloudコマンドの設定、環境変数の設定

```sh
gcloud config set run/region us-central1

LOCATION="us-central1"
```

既存のイメージからCloud Runリソース`quickway-parking-billing-v1`を作成
`--allow-unauthenticated`なので、誰でもこのCloud RunをURLから呼び出せる
`--no-allow-unauthenticated`の場合は認証済みユーザのみしか呼び出せない(`--allow-unauthenticated`を付けなければこちらになる)

```sh
gcloud beta run deploy quickway-parking-billing-v1 \
  --image gcr.io/qwiklabs-resources/gsp723-parking-service \
  --region $LOCATION \
  --allow-unauthenticated
```

Cloud Runリソース`quickway-parking-billing-v1`のURLを取得

```sh
QUICKWAY_SERVICE=$(gcloud run services list \
  --format='value(URL)' \
  --filter="quickway")
```

作成したCloud Runリソースを消す

```sh
gcloud run services delete quickway-parking-billing-v1
```

既存のイメージからCloud Runリソース`quickway-parking-billing-v2`を作成
`--no-allow-unauthenticated`なので、認証済みユーザのみしか呼び出せない(`--allow-unauthenticated`を付けなければこちらになる)

```sh
gcloud run deploy quickway-parking-billing-v2 \
  --image gcr.io/qwiklabs-resources/gsp723-parking-service \
  --region $LOCATION \
  --no-allow-unauthenticated
```

サービスアカウントを作成し、そのサービスアカウントに`Cloud Run Invoker`ロールを付与

プロジェクトレベルで付与した場合、そのプロジェクト内のCloud Runリソースに対してCloud Runを起動する権限を持つ
Cloud Runリソース毎にアカウントに対して`Cloud Run Invoker`ロールを付与するということもできる

## Using a Global Load Balancer with Cloud Run [APPRUN]

本プロジェクトでCloud Run APIを有効化する

```sh
gcloud services enable run.googleapis.com
```

gcloudコマンドの設定、環境変数の設定

```sh
gcloud config set compute/region us-central1
gcloud config set run/region us-central1

LOCATION="us-central1"
```

Cloud Run上で動作させるイメージを作成するために必要なファイルを作成する

```sh
mkdir helloworld && cd helloworld
```

main.pyを作成

```py
import os
from flask import Flask
app = Flask(__name__)
@app.route("/")
def hello_world():
    name = os.environ.get("NAME", "World")
    return "Hello {}!".format(name)
if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

Dockerfileを作成

```Dockerfile
# Use the official lightweight Python image.
# https://hub.docker.com/_/python
FROM python:3.9-slim
# Allow statements and log messages to immediately appear in the Knative logs
    ENV PYTHONUNBUFFERED True
# Copy local code to the container image.
ENV APP_HOME /app
WORKDIR $APP_HOME
COPY . ./
# Install production dependencies.
RUN pip install Flask gunicorn
# Run the web service on container startup. Here we use the gunicorn
# webserver, with one worker process and 8 threads.
# For environments with multiple CPU cores, increase the number of workers
# to be equal to the cores available.
# Timeout is set to 0 to disable the timeouts of the workers to allow Cloud Run to handle instance scaling.
CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 main:app
```

Dockerイメージを作成してプロジェクト専用のArtifact Registryに格納

```sh
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld
```

Artifact Registry上のイメージを使ってCloud Runリソースをデプロイ

```sh
gcloud run deploy --image gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld
```

グローバルロードバランサーで使用するIPアドレスを確保する
`--global`とすることでエニーキャストを可能にしている?

```sh
gcloud compute addresses create example-ip \
    --ip-version=IPV4 \
    --global

# アドレスを確認
gcloud compute addresses describe example-ip \
    --format="get(address)" \
    --global
```

バックエンドリソースに指定するサーバレスnetwork-endpoint-groupsを`myneg`という名前で作成する
`--network-endpoint-type=serverless`としているのでサーバレスnetwork-endpoint-groupsとなる
`--cloud-run-service=helloworld`としているのでCloud Runリソースとしては`helloworld`を使う
network-endpoint-groupsはリージョンリソース？

```sh
gcloud compute network-endpoint-groups create myneg \
   --region=$LOCATION \
   --network-endpoint-type=serverless  \
   --cloud-run-service=helloworld
```

バックエンドサービスを`mybackendservice`という名前で作成する
`--global`としているのでグローバルリソースとして作成している？

```sh
gcloud compute backend-services create mybackendservice \
    --global
```

バックエンドサービスを`mybackendservice`に`myneg`という名前のサーバレスnetwork-endpoint-groupを追加
`--network-endpoint-group-region=$LOCATION`で`myneg`が存在するリージョンを指定している？
`--global`で`mybackendservice`自体のリソースがグローバルにあることを指定しているのか？

リージョンごとに作成したnetwork-endpoint-groupを追加していくとよしなにトラフィックを分割してくれる？

```sh
gcloud compute backend-services add-backend mybackendservice \
    --global \
    --network-endpoint-group=myneg \
    --network-endpoint-group-region=$LOCATION
```

target-http-proxiesに与えるurl-mapsを`myurlmap`という名前で作成

```sh
gcloud compute url-maps create myurlmap \
    --default-service mybackendservice
```

url-mapsとして`myurlmap`を指定して`mytargetproxy`という名前でtarget-http-proxiesを作成

```sh
gcloud compute target-http-proxies create mytargetproxy \
    --url-map=myurlmap
```

`myforwardingrule`というforwarding-rulesを作成する
`--address=example-ip`としているので、受け付けIPは`example-ip`という名前のIPアドレスリソースのIPアドレスを使用
`--target-http-proxy=mytargetproxy`としているので、`mytargetproxy`というtarget-http-proxiesを使うことを指定
`--ports=80`としているの待ち受けポートは`80`を使用
`--global`はグローバルリソースとして作成することを指定している？

これによりフォワードされるようにLoad Balancerが作成される

```sh
gcloud compute forwarding-rules create myforwardingrule \
    --address=example-ip \
    --target-http-proxy=mytargetproxy \
    --global \
    --ports=80
```

これでロードバランサー経由でアクセスできるようになった

## Configuring Egress from a Static Outbound IP Address [APPRUN]

gcloudコマンドの設定、環境編巣の設定

```sh
gcloud config set compute/region us-central1

LOCATION="us-central1"
```

DockerイメージをPack CLI(Cloud Consoleにはインストールされているっぽい)を使って作成し、Artifact Repositryに格納.

```sh
git clone https://github.com/GoogleCloudPlatform/buildpack-samples.git

cd buildpack-samples/sample-go
pack build --builder=gcr.io/buildpacks/builder sample-go

docker run -it -e PORT=8080 -p 8080:8080 sample-go

pack set-default-builder gcr.io/buildpacks/builder:v1

pack build --publish gcr.io/$GOOGLE_CLOUD_PROJECT/sample-go
```

Cloud RunがプロジェクトのVPCネットワークにアクセスするために使用するVPC Connector(vpc-access)が接続するVPCネットワーク上のサブネットを作成する.

```sh
gcloud compute networks list

gcloud compute networks subnets create mysubnet \
   --range=192.168.0.0/28 --network=default --region=$LOCATION
```

VPC Connectorを作成し、そのVPCコネクタが接続するサブネットとして先に作ったサブネットを指定する.
これにより、作成したVPC Connectorを使用するCloud Runリソースからのネットワークのアクセスは、VPC Connectorが所属するサブネットのIPアドレスをソースアドレスとしたアクセスとなる.

```sh
gcloud compute networks vpc-access connectors create myconnector \
  --region=$LOCATION \
  --subnet-project=$GOOGLE_CLOUD_PROJECT \
  --subnet=mysubnet
```

インターネットにアクセスするためのルーターを`myrouter`という名前で作成.

```sh
gcloud compute routers create myrouter \
  --network=default \
  --region=$LOCATION
```

natで使用するインターネットIPアドレスを確保する.

```sh
gcloud compute addresses create myoriginip --region=$LOCATION
```

ルータ`myrouter`にnat機能を設定する.
`--router`でどのルータに適用するかを指定.
`--nat-custom-subnet-ip-ranges`でVPCネットワーク内のどのサブネットのIPアドレスをnatの対象とするかを指定.
`--nat-external-ip-pool`でnatで使用する外部IPアドレス(インターネットIPアドレス)を指定する.

```sh
gcloud compute routers nats create mynat \
  --router=myrouter \
  --region=$LOCATION \
  --nat-custom-subnet-ip-ranges=mysubnet \
  --nat-external-ip-pool=myoriginip
```

Cloud Runリソースを作成.
`--vpc-connector`でCloud Runが接続するVPC Connectorを指定.
`--vpc-egress=all-traffic`でCloud RunからのすべてのネットワークへのアクセスをVPC Connector経由とすることを指定.

```sh
gcloud run deploy sample-go \
   --image=gcr.io/$GOOGLE_CLOUD_PROJECT/sample-go \
   --vpc-connector=myconnector \
   --vpc-egress=all-traffic
```

## Cloud SSQL with Cloud RUN[APPRUN]

gcloud services enable run.googleapis.com

gcloud config set compute/region us-central1

LOCATION="us-central1"

Cloud SQL instanceを作成

Cloud SQLの実体はプロジェクトのVPC上には存在しないことに注意.

- postgreSQLのユーザ`postgres`のパスワードを設定
- Public IP Adressでのアクセスを可とする
- 作成したインスタンスConnection Nameを確認する

> セキュリティを確保するには色々な方法が考えらるが、その一つが以下.
>
> Nat機能を設定、NatのIPアドレスのみをCloud SQLのDBにアクセス可能なIPアドレスとして指定.
> Natの設定としては、VPC内の特定のサブネットから外部へのアクセスを対象とする.
> VPC ConnectorをNat対象となるVPC上のサブネットに接続する形で作成.
> Cloud Runで設定したVPC Connectorを使用するように設定.

Cloud SQLにgcloudコマンド経由でアクセスし、必要となるテーブルなどを作成する.

```sh
gcloud sql connect poll-database --user=postgres

\connect postgres;

CREATE TABLE IF NOT EXISTS votes
( vote_id SERIAL NOT NULL, time_cast timestamp NOT NULL,
candidate VARCHAR(6) NOT NULL, PRIMARY KEY (vote_id) );

CREATE TABLE IF NOT EXISTS totals
( total_id SERIAL NOT NULL, candidate VARCHAR(6) NOT NULL,
num_votes INT DEFAULT 0, PRIMARY KEY (total_id) );

INSERT INTO totals (candidate, num_votes) VALUES ('TABS', 0);

INSERT INTO totals (candidate, num_votes) VALUES ('SPACES', 0);
```

Cloud Runリソースを、作成したCloud SQLにアクセスするための情報を環境変数として与える形で作成.
`--add-cloudsql-instances`を使用するとCloud SQL Auth Proxyを使用したアクセスとなり、自動的にCloud SQLインスタンスへのデータが暗号化された形になるっぽい.
[Cloud SQL Auth Proxy を使用して Cloud Run から Cloud SQL に接続する](https://blog.g-gen.co.jp/entry/from-cloudrun-to-cloud-sql-with-auth-proxy)
`--set-env-vars`でCloud Runが実行されるときの環境変数を注入.

```sh
gcloud beta run deploy poll-service \
   --image gcr.io/qwiklabs-resources/gsp737-tabspaces \
   --region $LOCATION \
   --allow-unauthenticated \
   --add-cloudsql-instances=$CLOUD_SQL_CONNECTION_NAME \
   --set-env-vars "DB_USER=postgres" \
   --set-env-vars "DB_PASS=secretpassword" \
   --set-env-vars "DB_NAME=postgres" \
   --set-env-vars "CLOUD_SQL_CONNECTION_NAME=$CLOUD_SQL_CONNECTION_NAME"
```

## Using Cloud PubSub with Cloud Run [APPRUN]

Cloud Run APIを有効化.

```sh
gcloud services enable run.googleapis.com
```

gcloudコマンドの設定、環境変数の設定.

```sh
gcloud config set compute/region us-central1
LOCATION="us-central1"
```

認証されていないユーザでも実行可能なCloud Runリソースを`store-service`という名前で作成.

```sh
gcloud run deploy store-service \
 --image gcr.io/qwiklabs-resources/gsp724-store-service \
 --region $LOCATION \
 --allow-unauthenticated
```

認証されていないユーザは実行できないCloud Runリソースを`order-service`という名前で作成.

```sh
gcloud run deploy order-service \
 --image gcr.io/qwiklabs-resources/gsp724-order-service \
 --region $LOCATION \
 --no-allow-unauthenticated
```

Pub/Subトピックを作成.

```sh
gcloud pubsub topics create ORDER_PLACED
```

`order-service`起動用のサービスアカウントを作成.

```sh
gcloud iam service-accounts create pubsub-cloud-run-invoker \
   --display-name "Order Initiator"

gcloud iam service-accounts list --filter="Order Initiator"
```

先ほど作ったサービスアカウントに、Cloud Runリソース`order-serivce`に対する起動権限を付与.

```sh
gcloud run services add-iam-policy-binding order-service \
  --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
  --role=roles/run.invoker
  --platform managed
```

Pub/Subサービスを実行しているサービスアカウント`service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com`に対して他のアカウントのトークンを作成するために必要となるロール(`roles/iam.serviceAccountTokenCreator`)をプロジェクトレベルで付与.

```sh
PROJECT_NUMBER=$(gcloud projects list \
  --filter="qwiklabs-gcp" \
  --format='value(PROJECT_NUMBER)')

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
   --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com \
   --role=roles/iam.serviceAccountTokenCreator
```

先ほど作ったTopicに投稿されたときに、`order-serivce`を起動するようにサブスクリプションを作成.
`--push-endpoint`で`order-service`のエンドポイントURLを指定.
`--push-auth-service-account`で起動時に使用するサービスアカウントを指定.

```sh
ORDER_SERVICE_URL=$(gcloud run services describe order-service \
   --region $LOCATION \
   --format="value(status.address.url)")

gcloud pubsub subscriptions create order-service-sub \
   --topic ORDER_PLACED \
   --push-endpoint=$ORDER_SERVICE_URL \
   --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

認証なしのCloud Runサービス(`store-service`)にアクセス.
`store-service`はその処理の中で上記で作成したTopicに対して投稿するようになっている.
その結果、`order-service`がPub/Sub経由で実行されることになる.

```sh
test.json

{
  "billing_address": {
    "name": "Kylie Scull",
    "address": "6471 Front Street",
    "city": "Mountain View",
    "state_province": "CA",
    "postal_code": "94043",
    "country": "US"
  },
  "shipping_address": {
    "name": "Kylie Scull",
    "address": "9902 Cambridge Grove",
    "city": "Martinville",
    "state_province": "BC",
    "postal_code": "V1A",
    "country": "Canada"
  },
  "items": [
    {
      "id": "RW134",
      "quantity": 1,
      "sub-total": 12.95
    },
    {
      "id": "IB541",
      "quantity": 2,
      "sub-total": 24.5
    }
 ]
}

STORE_SERVICE_URL=$(gcloud run services describe store-service \
   --region $LOCATION \
   --format="value(status.address.url)")

curl -X POST -H "Content-Type: application/json" -d @test.json $STORE_SERVICE_URL
```

## Cloud Build を使ってみる

Cloud BuildでDockerイメージを作成するために必要なファイルを作成.

- Dockerfile
- DockerfileでCOPYされるファイル

Cloud BuildでDockerイメージを作成してプロジェクト固有のArtifact Repositryに格納.

```sh
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/quickstart-image .
```

----

Cloud Buildのサンプルを取得

```sh
git clone https://github.com/GoogleCloudPlatform/training-data-analyst

ln -s ~/training-data-analyst/courses/ak8s/v1.1 ~/ak8s

cd ~/ak8s/Cloud_Build/a
```

Cloud Buildで実行する内容を記述したファイルを参照.

```sh
cat cloudbuild.yaml
```

```yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: [ 'build', '-t', 'gcr.io/$PROJECT_ID/quickstart-image', '.' ]
images:
- 'gcr.io/$PROJECT_ID/quickstart-image'
```

ビルドステップを記述したファイルを指定してCloud BuildでDockerイメージを作成.

```sh
gcloud builds submit --config cloudbuild.yaml .
```

----

```sh
cd ~/ak8s/Cloud_Build/b
```

Cloud Buildで実行する内容を記述したファイルを参照.

```sh
cat cloudbuild.yaml
```

```yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: [ 'build', '-t', 'gcr.io/$PROJECT_ID/quickstart-image', '.' ]
- name: 'gcr.io/$PROJECT_ID/quickstart-image'
  args: ['fail']
images:
- 'gcr.io/$PROJECT_ID/quickstart-image
```

ビルドステップを記述したファイルを指定してCloud BuildでDockerイメージを作成.

```sh
gcloud builds submit --config cloudbuild.yaml .
```
