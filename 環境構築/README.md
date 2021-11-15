# Kubernetes環境構築

## gitpodで環境を構築する場合の初期設定

```sh
#下記コマンドでそれなりに権限のあるAWSユーザにログインできる設定をしてください。
#シークレットキーとアクセスキーを聞かれます
aws configure
```

## まずはAWS上にKubernetesを構築する開発環境を準備

* Network.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8snetwork \
    --template-body file://./Network.yaml \
    --parameters \
    ParameterKey=VPCCidr,ParameterValue="10.0.0.0/16" \
    ParameterKey=SubnetCidr,ParameterValue="10.0.1.0/24"

    #作成したStackの詳細確認
    aws cloudformation describe-stacks --stack-name k8snetwork

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8snetwork | jq '.Stacks[].StackStatus'

    #進捗をwatchとかで見とくのもあり。
    watch -c "aws cloudformation describe-stacks --stack-name k8snetwork | jq '.Stacks[].StackStatus'"
    ```

* Route.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8sroute \
    --template-body file://./Route.yaml

    #作成したStackの詳細確認
    aws cloudformation describe-stacks --stack-name k8sroute | jq .

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8sroute | jq '.Stacks[].StackStatus'

    #進捗をwatchとかで見とくのもあり。
    watch -c "aws cloudformation describe-stacks --stack-name k8sroute | jq '.Stacks[].StackStatus'"
    ```

* SecurityGroup.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8ssg \
    --template-body file://./SecurityGroup.yaml \
    --parameters \
    ParameterKey=GroupName,ParameterValue="devenv-sg"

    #作成したStackの詳細確認
    aws cloudformation describe-stacks --stack-name k8ssg | jq .

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8ssg | jq '.Stacks[].StackStatus'

    #進捗をwatchとかで見とくのもあり。
    watch -c "aws cloudformation describe-stacks --stack-name k8ssg | jq '.Stacks[].StackStatus'"
    ```

* Key

    ```sh
    #unix系(Windowsのキー格納場所は知らないので適宜変えてください。) 
    aws ec2 create-key-pair --key-name kubernetespoc --query 'KeyMaterial' --output text > ~/.ssh/kubernetespoc.pem
    ```

* Instance.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8sdevinstance \
    --template-body file://./Instance.yaml

    #作成したStackの詳細確認
    aws cloudformation describe-stacks --stack-name k8sdevinstance | jq .

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8sdevinstance | jq '.Stacks[].StackStatus'

    #進捗をwatchとかで見とくのもあり。
    watch -c "aws cloudformation describe-stacks --stack-name k8sdevinstance | jq '.Stacks[].StackStatus'"

    #EC2は起動までに時間がかかるので、状態チェックしていく、まずはインスタンスIDを取得(Stackが作成されるまではエラーになります)
    aws cloudformation describe-stacks --stack-name k8sdevinstance | jq '.Stacks[].Outputs[]'

    #EC2の詳細確認(running, ok 等動いてそうならOK)
    aws ec2 describe-instance-status \
    --instance-ids [上記で検索したインスタンスID] \
    | jq .

    #こっちでも詳細は確認可能
    aws ec2 describe-instances --instance-ids [上記で検索したインスタンスID] | jq .

    #接続方法等は後ほど記載しておきます
    ```

* Lambda.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8sdevstoplambda \
    --template-body file://./Lambda.yaml \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters \
    ParameterKey=FunctionName,ParameterValue="EC2StopLambda"

    #作成したStackの詳細確認
    aws cloudformation describe-stacks --stack-name k8sdevstoplambda | jq .

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8sdevstoplambda | jq '.Stacks[].StackStatus'

    #進捗をwatchとかで見とくのもあり。
    watch -c "aws cloudformation describe-stacks --stack-name k8sdevstoplambda | jq '.Stacks[].StackStatus'"
    ```

* EventBridge.yaml（２３時停止）

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8seb \
    --template-body file://./EventBridge.yaml \
    --parameters \
    ParameterKey=RuleName,ParameterValue="stopec2-2300pm" \
    ParameterKey="StopScheduled",ParameterValue="cron(0 14 * * ? *)"

    #作成したStackの詳細確認
    aws cloudformation describe-stacks --stack-name k8seb | jq .

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8seb | jq '.Stacks[].StackStatus'

    #進捗をwatchとかで見とくのもあり。
    watch -c "aws cloudformation describe-stacks --stack-name k8seb | jq '.Stacks[].StackStatus'"
    ```

* EventBridge.yaml（月から金の８時半停止）

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8seb2 \
    --template-body file://./EventBridge.yaml \
    --parameters \
    ParameterKey=RuleName,ParameterValue="stopec2-0830am" \
    ParameterKey=StopScheduled,ParameterValue="cron(30 23 ? * SUN-THU *)"

    #作成したStackの詳細確認
    aws cloudformation describe-stacks --stack-name k8seb | jq .

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8seb | jq '.Stacks[].StackStatus'

    #進捗をwatchとかで見とくのもあり。
    watch -c "aws cloudformation describe-stacks --stack-name k8seb | jq '.Stacks[].StackStatus'"
    ```

* EventBridge.yaml（土日の１３時停止）

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8seb3 \
    --template-body file://./EventBridge.yaml \
    --parameters \
    ParameterKey=RuleName,ParameterValue="stopec2-1300pm" \
    ParameterKey=StopScheduled,ParameterValue="cron(00 4 ? * SAT-SUN *)"

    #作成したStackの詳細確認
    aws cloudformation describe-stacks --stack-name k8seb | jq .

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8seb | jq '.Stacks[].StackStatus'

    #進捗をwatchとかで見とくのもあり。
    watch -c "aws cloudformation describe-stacks --stack-name k8seb | jq '.Stacks[].StackStatus'"
    ```

* 削除手順

    ```sh
    aws cloudformation delete-stack --stack-name k8seb
    aws cloudformation delete-stack --stack-name k8seb2
    aws cloudformation delete-stack --stack-name k8seb3

    #上の３つが消えてから実行
    aws cloudformation delete-stack --stack-name k8sdevstoplambda

    #lamdaの削除が完了してから実行
    aws cloudformation delete-stack --stack-name k8sdevinstance

    #EC2削除が完了してから実行
    aws cloudformation delete-stack --stack-name k8ssg

    aws cloudformation delete-stack --stack-name k8sroute

    aws cloudformation delete-stack --stack-name k8snetwork
    ```

## 開発環境にログインし、K8s構築に必要なクライアントツールを準備する

* まずはログイン

    ```sh
    #EC2のインスタンスIDを取得
    aws cloudformation describe-stacks --stack-name k8sdevinstance | jq '.Stacks[].Outputs[].OutputValue'
    #インスタンスIDからIPを取得
    aws ec2 describe-instances --instance-ids i-036504421a955f4da | jq '.Reservations[].Instances[].PublicIpAddress'

    ssh ec2-user@<上記で取得したした開発用EC2のIP>
    ```

* CFSSLとCFSSLjsonをインストール(SSL証明書発行用)

    ```sh
    #インストール
    wget https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl
    wget https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson

    chmod +x cfssl cfssljson

    sudo mv cfssl cfssljson /usr/local/bin/

    #検証
    cfssl version

    cfssljson --version
    ```

* kubectlをインストール

    ```sh
    #インストール
    wget https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl

    chmod +x kubectl

    sudo mv kubectl /usr/local/bin/

    #検証
    kubectl version --client
    ```

## Kubernetes用のサーバをデプロイする

* 作業ディレクトリ

    環境構築/CloudFormation/Kubernetes/

* 01_Network.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8snodenetwork \
    --template-body file://./01_Network.yaml

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8snodenetwork | jq '.Stacks[].StackStatus'
    ```

* 02_Route.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8snoderoute \
    --template-body file://./02_Route.yaml

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8snoderoute | jq '.Stacks[].StackStatus'
    ```

* 03_SecurityGroup.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8snodesg \
    --template-body file://./03_SecurityGroup.yaml

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8ssg | jq '.Stacks[].StackStatus'

    ```

* LoadBalancer.yaml（ControllerNodeを冗長構成にしないので、当手順は実施しない）

    ```sh
    #作成
    #aws cloudformation create-stack \
    #--stack-name k8snodelb \
    #--template-body file://./LoadBalancer.yaml

    #作成したStackの詳細確認
    #aws cloudformation describe-stacks --stack-name k8snodelb | jq .
    ```

* 04_ControllerNode.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8snodecontrollerinstance0 \
    --template-body file://./04_ControllerNode.yaml \
    --parameters \
    ParameterKey=InstanceName,ParameterValue="controller-0" \
    ParameterKey=KeyName,ParameterValue="kubernetes" \
    ParameterKey=InstanceIP,ParameterValue="10.240.0.10"
    ```

* 05_WorkerNode.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8snodeworkerinstance0  \
    --template-body file://./05_WorkerNode.yaml \
    --parameters \
    ParameterKey=InstanceName,ParameterValue="worker-0 " \
    ParameterKey=KeyName,ParameterValue="kubernetes" \
    ParameterKey=InstanceIP,ParameterValue="10.240.0.20 "
    ```

## Ubuntuにログインする際の注意点

[参考](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/managing-users.html)

```sh
#ec2-userではログインできないので注意(用意されていない)
ssh -i ~/.ssh/<指定したPemキー> ubuntu@<EC2のIPアドレス>
```

## CAのプロビジョニングとTLS証明書の生成

* CA用の構成ファイル(ca.csr)やCA自身の証明書、秘密鍵を生成する

```sh
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
# 以下ファイルが作成される
# ca-key.pem
# ca.csr SSLサーバ証明書への署名を申請する内容
# ca.pem 
```

* 次はadmin用のクライアント証明書と秘密鍵を生成

```sh
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
#以下ファイルが作成される
#admin-csr.json
#admin-key.pem
#admin.csr
#admin.pem
```

VPC_ID = $(aws ec2 describe-vpcs --filters Name=tag:Name,Values=kubernetes-the-hard-way --output text --query 'Vpcs[0].VpcId')
ROUTE_TABLE_ID = $(aws ec2 create -route-table --vpc-id ${VPC_ID} --output text --query 'RouteTable.RouteTableId')

sg-0bce7f29e9b74b8f5    
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port ${NODE_PORT} \
  --cidr 0.0.0.0/0

## PrometheusServerの構築

* 作業ディレクトリ

  Kubenetes/環境構築/CloudFormation/Kubernetesdev

* 前提条件

  手順：【Kubernetes用のサーバをデプロイする】の以下項目がすでに作成済みであること  
  1. Network.yaml  
  2. Route.yaml  
  ※K8s インスタンスを構築したサブネット上にPrometheusも構築するため。

* PrometheusSG.yaml

  ```sh
  #作成
  aws cloudformation create-stack \
  --stack-name prometheussg \
  --template-body file://./10_PrometheusSG.yaml

  #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
  aws cloudformation describe-stacks --stack-name prometheussg | jq '.Stacks[].StackStatus'
  ```

* PrometheusServer.yaml

  ```sh
  #作成
  aws cloudformation create-stack \
  --stack-name prometheusserver \
  --template-body file://./11_PrometheusServer.yaml \
  --parameters \
  ParameterKey=InstanceName,ParameterValue="prometheusserver" \
  ParameterKey=KeyName,ParameterValue="kubernetespoc" \
  ParameterKey=InstanceIP,ParameterValue="10.240.0.31"
  ```

## PrometheusServerのセットアップ

* Prometheusの起動

  ```sh
  #SSHLogin
  ssh -i ~/.ssh/kubernetespoc.pem ec2-user@$(aws ec2 describe-instances --instance-ids $(aws cloudformation describe-stacks --stack-name prometheusserver | jq '.Stacks[].Outputs[].OutputValue' | awk -F'["]' '{print $2}') | jq '.Reservations[].Instances[].PublicIpAddress' | awk -F'["]' '{print $2}')

  #Prometheusのダウンロード(URLは適宜最新版に変えてください。：https://prometheus.io/download/)
  wget "https://github.com/prometheus/prometheus/releases/download/v2.31.1/prometheus-2.31.1.linux-amd64.tar.gz"

  #展開
  tar -xzf prometheus-2.31.1.linux-amd64.tar.gz

  #設定の修正
  vim ./prometheus-2.31.1.linux-amd64/prometheus.yml
  #修正内容は下記の通り(#は消してください)
  #global:
  #  scrape_interval: 10s
  #scrape_configs:
  #  - job_name: "prometheus"
  #    static_configs:
  #      - targets: ["localhost:9090"]

  #実行
  cd ./prometheus-2.31.1.linux-amd64
  ./prometheus
  ```

* Node exporterを使用してOSのメトリクスを取得する

  ```sh
  #SSHLoginできていなければ実行
  ssh -i ~/.ssh/kubernetespoc.pem ec2-user@$(aws ec2 describe-instances --instance-ids $(aws cloudformation describe-stacks --stack-name prometheusserver | jq '.Stacks[].Outputs[].OutputValue' | awk -F'["]' '{print $2}') | jq '.Reservations[].Instances[].PublicIpAddress' | awk -F'["]' '{print $2}')

  #Node exporterのダウンロード(URLは適宜最新版に変えてください。：https://prometheus.io/download/)
  wget "https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz"

  #展開
  tar -xzf node_exporter-1.2.2.linux-amd64.tar.gz

  #実行
  cd node_exporter-1.2.2.linux-amd64
  ./node_exporter

  #Prometheusのダッシュボードで色々見てみると、ターゲットが増えていたり取得しているデータが増えている
  ```

* 設定を変更してPrometheusからアラートを発行させる

  ```sh
  #Alert Ruleを設定
  vim prometheus.yml
  #修正内容は下記の通り(#は消してください)
  #global:
  #  scrape_interval: 10s
  #  evaluation_interval: 10s
  #alerting:
  #  alertmanagers:
  #    - static_configs:
  #        - targets:
  #          - localhost:9093
  #rule_files:
  #    - "rules.yml"
  #scrape_configs:
  #  - job_name: "prometheus"
  #    static_configs:
  #      - targets: ["localhost:9090"]
  #  - job_name: "node"
  #    static_configs:
  #      - targets: ["localhost:9100"]

  vim rules.yml
  #修正内容は下記の通り(#は消してください)
  #groups:
  #- name: example
  #  rules:
  #  - alert: InstanceDown
  #    expr: up == 0
  #    for: 1m
  #  labels:
  #          severity: __severity__
  #        annotations:
  #          summary: testalert
  #          description: testalert

  #設定を読み込ませるために再起動(すでに起動している場合は一度落としてから下記コマンドで起動してください)
  ./prometheus

  ```

* Alert Managerを使用してPrometheusが吐くアラートを処理する(今回はSlackに送信)

  ```sh
  #SSHLoginできていなければ実行
  ssh -i ~/.ssh/kubernetespoc.pem ec2-user@$(aws ec2 describe-instances --instance-ids $(aws cloudformation describe-stacks --stack-name prometheusserver | jq '.Stacks[].Outputs[].OutputValue' | awk -F'["]' '{print $2}') | jq '.Reservations[].Instances[].PublicIpAddress' | awk -F'["]' '{print $2}')

  #ダウンロード
  wget "https://github.com/prometheus/alertmanager/releases/download/v0.23.0/alertmanager-0.23.0.linux-amd64.tar.gz"

  #展開
  tar -xzf alertmanager-0.23.0.linux-amd64.tar.gz

  #AlertManagerの設定ファイルを編集
  cd alertmanager-0.23.0.linux-amd64
  vim alertmanager.yml

  #修正内容は下記の通り(#は消してください)
  #global:
  #  slack_api_url: 'https://hooks.slack.com/services/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'

  #route:
  #  receiver: 'slack-notifications' # (1)

  #receivers:
  #- name: 'slack-notifications' # (2)
  #  slack_configs:
  #  - channel: 'prometheus-alert'
  #    title: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}" # (3)
  #    text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"

  #alertmanagerを実行
  ./alertmanager
  ```

## Pythonで作成したアプリケーションのメトリクスを取得してみる

* Smapleアプリケーション

  ```python3
  import random
  import time
  import http.server
  from prometheus_client import start_http_server
  from prometheus_client import Counter
  from prometheus_client import Gauge
  from prometheus_client import Summary

  #Hello Worldが返された回数を追跡
  REQUESTS = Counter('hello_worlds_total', 'Hello Worlds requested.')
  EXCEPTIONS = Counter('hello_world_exceptions_total', 'Exceptions serving Hello World.')
  INPROGRESS = Gauge('hello_worlds_inprogress', 'Number of Hello Worlds in progress')
  LAST = Gauge('hello_world_last_time_seconds', 'The last time a Hello World was served.')
  LATENCY = Summary('hello_world_latency_seconds', 'Time for a request Hello World.')

  class MyHandler(http.server.BaseHTTPRequestHandler):
    @INPROGRESS.track_inprogress()
    @LATENCY.time()
    def do_GET(self):
      REQUESTS.inc()#Counterを１増加させる
      with EXCEPTIONS.count_exceptions():
        if random.random() < 0.2:
          raise Exception
      self.send_response(200)
      self.end_headers()
      self.wfile.write(b"Hello World")
      LAST.set_to_current_time()

  if __name__ == "__main__":
    start_http_server(8000) #port8000にPrometheusにメトリクスを配信するHTTPサーバを起動する
    server = http.server.HTTPServer(('localhost', 8001), MyHandler)
    server.serve_forever()
  ```

* pythonを実行

  ```sh
  #Python確認
  python3 --version

  #ライブラリをインストール
  pip3 install prometheus_client

  #SampleProgramを配置(内容は上記のSampleアプリケーション)
  vim sample.py
  ```

* PromQLで色々確認

  ```PromQL
  #Pythonが上がっているかを確認
  up

  #pythonの情報を確認

  #pythonが叩かれた回数をチェック(EC2上でcurlを打鍵し、pytonを何度か呼び出してみてからチェックしてみてください。)
  rate(hello_worlds_total[1m])

  #pythonの呼び出し
  #注意：PythonコードでHTTPサーバを立てているわけではないので、別端末からhttpリクエストを送信しても応答はない。
  curl "http://localhost:8001"
  ```

## Spring Boot Appicationからメトリクスを取得する

  今回、以下のソースコードを使用してメトリクスを取得してみる  
  [Yumapon/SpringTaskApp](https://github.com/Yumapon/MetricsTest.git)  

* Applicationの用意

  ```sh
  #git install
  sudo yum isntall -y

  #git clone
  git clone https://github.com/Yumapon/MetricsTest.git

  #実行前にJava11をインストールしといてください

  #ビルド
  cd MetricsTest/
  ./mvnw clean package -f ./pom.xml

  #実行
  java $JAVA_OPTIONS -jar target/demo-0.0.1-SNAPSHOT.jar

  #確認
  curl "http://$(aws ec2 describe-instances --instance-ids $(aws cloudformation describe-stacks --stack-name prometheusserver | jq '.Stacks[].Outputs[].OutputValue' | awk -F'["]' '{print $2}') | jq '.Reservations[].Instances[].PublicIpAddress' | awk -F'["]' '{print $2}'):8080/test"

  #prometheusのメトリクスが取得できているか確認
  curl "http://$(aws ec2 describe-instances --instance-ids $(aws cloudformation describe-stacks --stack-name prometheusserver | jq '.Stacks[].Outputs[].OutputValue' | awk -F'["]' '{print $2}') | jq '.Reservations[].Instances[].PublicIpAddress' | awk -F'["]' '{print $2}'):8080/actuator/prometheus"
  ```

* prometheus側のメトリクス取得設定

  ```sh
  #SSH Login
  ssh -i ~/.ssh/kubernetespoc.pem ec2-user@$(aws ec2 describe-instances --instance-ids $(aws cloudformation describe-stacks --stack-name prometheusserver | jq '.Stacks[].Outputs[].OutputValue' | awk -F'["]' '{print $2}') | jq '.Reservations[].Instances[].PublicIpAddress' | awk -F'["]' '{print $2}')

  #設定ファイルの編集
  cd prometheus-2.31.1.linux-amd64/
  vim promethes.yml
  #修正内容は下記の通り(#は消してください)
  # my global config
  #global:
  #  scrape_interval: 10s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  #  evaluation_interval: 10s # Evaluate rules every 15 seconds. The default is every 1 minute.
  ## Alertmanager configuration
  #alerting:
  #  alertmanagers:
  #    - static_configs:
  #        - targets:
  #          - localhost:9093

  ## Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
  #rule_files:
  #    - "rules.yml"
  #scrape_configs:
  #  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  #  - job_name: "prometheus"
  #    static_configs:
  #      - targets: ["localhost:9090"]
  #  - job_name: "node"
  #    static_configs:
  #      - targets: ["localhost:9100"]
  #  - job_name: "python"
  #    static_configs:
  #      - targets: ["localhost:8000"]
  #  - job_name: "springboot"
  #    metrics_path: "/actuator/prometheus"
  #    static_configs:
  #      - targets: ["localhost:8080"]

  #設定変更後、再起動してください
  
  ```

## メモ
