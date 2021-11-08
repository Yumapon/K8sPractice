# Kubernetes The Hard Way

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

## 開発環境にログインし、構築に必要なクライアントツールを準備する

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

* 作業フォルダ

環境構築/CloudFOrmation/Kubernetes/

* Network.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8snodenetwork \
    --template-body file://./Network.yaml

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8snodenetwork | jq '.Stacks[].StackStatus'
    ```

* Route.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8snoderoute \
    --template-body file://./Route.yaml

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8snoderoute | jq '.Stacks[].StackStatus'
    ```

* SecurityGroup.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8snodesg \
    --template-body file://./SecurityGroup.yaml

    #完了したかどうかだけの確認なら以下でOK("CREATE_COMPLETE"と表示される)
    aws cloudformation describe-stacks --stack-name k8ssg | jq '.Stacks[].StackStatus'

    ```

* LoadBalancer.yaml

    ```sh
    #作成
    aws cloudformation create-stack \
    --stack-name k8snodelb \
    --template-body file://./LoadBalancer.yaml

    #作成したStackの詳細確認
    aws cloudformation describe-stacks --stack-name k8snodelb | jq .
    ```

* ControllerNode.yaml

    ```sh
    #作成
    for i in 0 1 2; do
        aws cloudformation create-stack \
        --stack-name k8snodecontrollerinstance${i} \
        --template-body file://./ControllerNode.yaml \
        --parameters \
        ParameterKey=InstanceName,ParameterValue="controller-${i}" \
        ParameterKey=KeyName,ParameterValue="kubernetes" \
        ParameterKey=InstanceIP,ParameterValue="10.240.0.1${i}"
    done
    ```

* WorkerNode.yaml

    ```sh
    #作成
    for i in 0 1 2; do
        aws cloudformation create-stack \
        --stack-name k8snodeworkerinstance${i}  \
        --template-body file://./WorkerNode.yaml \
        --parameters \
        ParameterKey=InstanceName,ParameterValue="worker-${i} " \
        ParameterKey=KeyName,ParameterValue="kubernetes" \
        ParameterKey=InstanceIP,ParameterValue="10.240.0.2${i} "
    done
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
