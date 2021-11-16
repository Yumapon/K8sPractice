# Prometheus HandsOn

## この手順でできること

* AWSのEC2インスタンス上にPrometheus Server, Alert Manager etc..を構築できます
* Prometheusを使用した基本的なメトリクス収集を実践できます（OS metrics, Python metrics, Spring metrics etc..）
* Kubernetesのメトリクス監視（鋭意作成中）

## 前提条件

[Kubernetes.md](./Kubernetes.md)のハンズオンにてK8sを構築済みであることが前提条件です。  
※VPCなど、すでに作成されている前提で手順を記載しています。

## Prometheus Serverの構築

* 作業ディレクトリ

    ``Kubenetes/環境構築/CloudFormation/Kubernetesdev``

* PrometheusServer用のSecurity Groupを作成する

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name prometheussg \
    --template-body file://./10_PrometheusSG.yaml

    #CREATE_COMPLETEと表示されるまで待つ
    watch -c "aws cloudformation describe-stacks --stack-name prometheussg | jq '.Stacks[].StackStatus'"
    ```

* PrometheusServerを構築する

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name prometheusserver \
    --template-body file://./11_PrometheusServer.yaml \
    --parameters \
    ParameterKey=InstanceName,ParameterValue="prometheusserver" \
    ParameterKey=KeyName,ParameterValue="kubernetespoc" \
    ParameterKey=InstanceIP,ParameterValue="10.240.0.31"

    #CREATE_COMPLETEと表示されるまで待つ
    watch -c "aws cloudformation describe-stacks --stack-name prometheusserver | jq '.Stacks[].StackStatus'"

    #EC2は起動までに少し時間がかかるので、状態チェックしていく、まずはインスタンスIDを取得(Stackが作成されるまではエラーになります)
    #EC2の詳細確認(running, ok 等動いてそうならOK)
    aws ec2 describe-instance-status \
    --instance-ids $(aws cloudformation describe-stacks --stack-name prometheusserver | jq '.Stacks[].Outputs[].OutputValue' | awk -F'["]' '{print $2}') \
    | jq .

    #こっちでも詳細は確認可能
    aws ec2 describe-instances --instance-ids $(aws cloudformation describe-stacks --stack-name prometheusserver | jq '.Stacks[].Outputs[].OutputValue' | awk -F'["]' '{print $2}') | jq .
    ```

## PrometheusServerのセットアップ（鋭意整理中）

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
  #  slack_api_url: 'slackのwebhookurl'

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

## Pythonで作成したアプリケーションのメトリクスを取得してみる（鋭意整理中）

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

## Spring Boot Appicationからメトリクスを取得する（鋭意整理中）

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
