# メモ

* CFSSLとCFSSLjsonをインストール(SSL証明書発行用)
  
  Kubernetes The Hard Wayではある程度準備してくれているが、一旦ちゃんと公式手順に沿ってインストールしてみる。  
  [公式サイト](https://github.com/cloudflare/cfssl)

    ```sh
    #CFSSLの動作前提であるGOをインストール
    #公式サイト: https://golang.org/doc/install
    sudo su

    #CFSSL自体はGo1.14以上が前提だが、AWS　Linuxのyum拡張repoではgo1.11までのみの提供であるため、手動でインストールする
    wget https://golang.org/dl/go1.17.2.linux-amd64.tar.gz

    rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.2.linux-amd64.tar.gz

    export PATH=$PATH:/usr/local/go/bin

    go version

    #次はCFSSLをGitから取得する必要があるため、Gitもインストールする
    #これはyumにあったので、これを使用
    yum info git

    yum install git -y

    git --version

    #ここでやっとインストール(goプロジェクト配下に展開する)
    #まずは下記コマンドでGo Packageの場所を確認(インストール時に指定したので、GOROOTが/usr/local になっているはず。)
    go env

    #次に、CFSSLを展開する場所を指定。(デフォルトで/root/go となっているので、そのままいく)
    #export GOPATH=$HOME/go

    #確認
    go env | grep GOPATH

    #gitリポジトリを$GOPATH/src/github.com/cloudflare/cfsslにクローンする
    git clone https://github.com/cloudflare/cfssl.git $GOPATH/src/github.com/cloudflare/cfssl

    cd $GOPATH/src/github.com/cloudflare/cfssl

    #c言語をコンパイルするために必要？(ないとエラー出る)
    yum install gcc -y

    #コンパイル
    make

    #cfsslコンパイル確認
    yum install tree -y
    tree bin
    #こんな感じならOK
    #bin
    #├── cfssl
    #├── cfssl-bundle
    #├── cfssl-certinfo
    #├── cfssl-newkey
    #├── cfssl-scan
    #├── cfssljson
    #├── mkbundle
    #├── multirootca
    #└── rice

    #やっとここでインストールする
    go get github.com/cloudflare/cfssl/cmd/cfssl
    go get github.com/cloudflare/cfssl/cmd/cfssljson

    #PATHの作成
    export PATH=$GOPATH/bin:$GOROOT/bin:$PATH

    #確認
    cfssl version
    #これでいいの？（要確認）
    #Version: dev
    #Runtime: go1.17.2
    ```

* kubectlのインストール

    [公式サイト](https://kubernetes.io/ja/docs/tasks/tools/install-kubectl/)

    ```sh
    curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"

    chmod +x ./kubectl

    sudo mv ./kubectl /usr/local/bin/kubectl
    ```
