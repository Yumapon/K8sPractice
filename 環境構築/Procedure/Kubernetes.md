# Kubernetes環境構築

## この手順でできること

* AWS上にkubeadmを使用せず、一からKubernetes Clusterを展開することができます。
* Kubernetesの各コンポーネントや、それぞれの通信設定等基礎の部分を把握することができます
* コントローラノードはシングル構成で、ワーカーノードを２台構成で作成します。[^1]

[^1]: コントローラノード３台構成かつワーカーノードも３台構成でLBもかました本番環境構成を作成したい場合は[Kubernetes The Hardway](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws)を参照してください

### K8sを構築するための開発用サーバをAWS上に用意する

* 作業ディレクトリ

    ``環境構築/CloudFormation/``

* VPC,Subnet,RouteTable,IGW等の作成

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8snetwork \
    --template-body file://./Network.yaml \
    --parameters \
    ParameterKey=VPCCidr,ParameterValue="10.0.0.0/16" \
    ParameterKey=SubnetCidr,ParameterValue="10.0.1.0/24"

    #CREATE_COMPLETEと表示されるまで待つ
    watch -c "aws cloudformation describe-stacks --stack-name k8snetwork | jq '.Stacks[].StackStatus'"
    ```

* ルートの定義

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8sroute \
    --template-body file://./Route.yaml

    #CREATE_COMPLETEと表示されるまで待つ
    watch -c "aws cloudformation describe-stacks --stack-name k8sroute | jq '.Stacks[].StackStatus'"
    ```

* SecurityGroupの作成

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8ssg \
    --template-body file://./SecurityGroup.yaml \
    --parameters \
    ParameterKey=GroupName,ParameterValue="devenv-sg"

    #CREATE_COMPLETEと表示されるまで待つ
    watch -c "aws cloudformation describe-stacks --stack-name k8ssg | jq '.Stacks[].StackStatus'"
    ```

* キーペアの作成

    以下コマンドで作成するキーペアは、Kubernetes用のサーバにログインする際にも使用します

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

    #CREATE_COMPLETEと表示されるまで待つ
    watch -c "aws cloudformation describe-stacks --stack-name k8sdevinstance | jq '.Stacks[].StackStatus'"

    #EC2は起動までに少し時間がかかるので、状態チェックしていく、まずはインスタンスIDを取得(Stackが作成されるまではエラーになります)
    #EC2の詳細確認(running, ok 等動いてそうならOK)
    aws ec2 describe-instance-status \
    --instance-ids $(aws cloudformation describe-stacks --stack-name k8sdevinstance | jq '.Stacks[].Outputs[].OutputValue' | awk -F'["]' '{print $2}') \
    | jq .

    #こっちでも詳細は確認可能
    aws ec2 describe-instances --instance-ids $(aws cloudformation describe-stacks --stack-name k8sdevinstance | jq '.Stacks[].Outputs[].OutputValue' | awk -F'["]' '{print $2}') | jq .

    #接続方法等は後ほど記載しておきます
    ```

* EC2停止用のLambdaを作成

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8sdevstoplambda \
    --template-body file://./Lambda.yaml \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters \
    ParameterKey=FunctionName,ParameterValue="EC2StopLambda"

    #CREATE_COMPLETEと表示されるまで待つ
    watch -c "aws cloudformation describe-stacks --stack-name k8sdevstoplambda | jq '.Stacks[].StackStatus'"
    ```

* 23時にEC2を停止させるEventBridgeを作成（上記で作成したLambdaを23時に呼び出します）

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8seb \
    --template-body file://./EventBridge.yaml \
    --parameters \
    ParameterKey=RuleName,ParameterValue="stopec2-2300pm" \
    ParameterKey="StopScheduled",ParameterValue="cron(0 14 * * ? *)"

    #CREATE_COMPLETEと表示されるまで待つ
    watch -c "aws cloudformation describe-stacks --stack-name k8seb | jq '.Stacks[].StackStatus'"
    ```

* 月から金の８時半にEC2を停止させるEventBridgeを作成（上記で作成したLambdaを月から金の８時半に呼び出します）

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8seb2 \
    --template-body file://./EventBridge.yaml \
    --parameters \
    ParameterKey=RuleName,ParameterValue="stopec2-0830am" \
    ParameterKey=StopScheduled,ParameterValue="cron(30 23 ? * SUN-THU *)"

    #CREATE_COMPLETEと表示されるまで待つ
    watch -c "aws cloudformation describe-stacks --stack-name k8seb | jq '.Stacks[].StackStatus'"
    ```

* 土日の１３時にEC2を停止させるEventBridgeを作成（上記で作成したLambdaを土日の１３時に呼び出します）

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8seb3 \
    --template-body file://./EventBridge.yaml \
    --parameters \
    ParameterKey=RuleName,ParameterValue="stopec2-1300pm" \
    ParameterKey=StopScheduled,ParameterValue="cron(00 4 ? * SAT-SUN *)"

    #CREATE_COMPLETEと表示されるまで待つ
    watch -c "aws cloudformation describe-stacks --stack-name k8seb | jq '.Stacks[].StackStatus'"
    ```

### 開発用サーバにK8s構築に必要なツールをインストールする

* 開発用サーバにログインする

    ```sh
    #SSHLogin
    ssh -i ~/.ssh/kubernetespoc.pem ec2-user@$(aws ec2 describe-instances --instance-ids $(aws cloudformation describe-stacks --stack-name prometheusserver | jq '.Stacks[].Outputs[].OutputValue' | awk -F'["]' '{print $2}') | jq '.Reservations[].Instances[].PublicIpAddress' | awk -F'["]' '{print $2}')
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

### K8sを構築（鋭意整理中）

### Clean up
