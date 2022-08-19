# 学習ステップ

1. 構成図を作成する

2. コード一文ごとの内容をメモする

3. resourceごとに表を作成する

4. 1〜3を繰り返す

## terraform初期設定
・Terraform用IAMユーザーを作成

・ターミナルにIAMユーザーを登録(aws configure)

・.gitignoreファイル作成(https://www.toptal.com/developers/gitignore/api/terraform)

・terraform initをして初期化

## 必要な設定

・tfstateをS3に移動

## 公式ドキュメント

|  種類  |  説明  |  URL  |
|  ----  |  ----  |  ----  |
|  HCL2  |  HCL2の仕様に関するドキュメント  |  https://www.terraform.io/language  |
|  CLI  |  Terraformのコマンドに関わるドキュメント  |  https://www.terraform.io/cli  |
|  provider  |  providerに関するドキュメント  |  https://registry.terraform.io/providers/hashicorp/aws/latest/docs  |

## 基本コマンド

|コマンド|内容|
|----|----|
|terraform init|ワークスペースを初期化|
|terraform fmt|フォーマットを修正|
|terraform plan|変更内容を確認|
|terraform apply|変更点を反映(確認あり)|
|terraform apply-auto-approve|変更点を反映(確認なし)|

## バージョン設定コード

|  コード  |  詳細  |
|  ----  |  ----  |
|required_version|terraformのバージョン|
|required_providers|awsの設定|
|source|hashicorpのものを使用|
|version|awsのバージョン|

````terraform
terraform {
  required_version = "1.2.4"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}
````

<!-- 
## tfstateをS3に移動

### なぜ実装する必要があるのか。

各々で実行するとtfstateの整合性が取れなくなる。管理対象システムとは異なるアカウントに共有S3バケットを作る。

### ステップ

1. すでに共有バケットが存在するなら10に移動。
2. マネージメントコンソールでS3に移動
3. バケットを作成。名前とリージョンを追加する。それ以外の設定は基本デフォルト。
4. ブロックパブリックアクセスを全てのブロックを解除する。
5. バケットポリシーを編集。
6. バケットポリシー欄からポリシージェネレータを作成する。
7. ユーザーのARN、許可するアクション、バケットのARNを作成する。バケットARNを貼る時/*もつける。
8. バケットポリシーを作成後、貼る。不要な部分は削除する。
9. ブロックパブリックアドレスを元に戻す。
10. tfstateをバケットに入れるようにterraformを修正する。


````terraform
terraform {
  required_version = "1.2.4"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
  
  # 追加
  backend "s3" {  
  bucket = "custoemr-db-bucket.tfstate"
  key = "customer-db-dev.tfstate"
  region = "ap-northeast-1"
  profile = "yamanaka@smasta"
  }
}
````


保存先を変更するため、
````terminal
terraform init
````

-->


## 変数の型を設定
````terraform
variable "enviroment" {
  type = string
}
````

変数の値を設定

terraform.tfvars

````terraform
enviroment = "dev"
````

## VPCの作成

### 作成理由

通信をする上で必須

VPC

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  cidr_block  |  ●  |  string  |  IPv4CIDRブロック  |
|  assign_generated_ipv6_cidr_block  ||  string  |  IPv6CIdRブロック  |
|  instance_tenancy  ||  enum  |  テナンシー(default, dedicated)  |
|  enable_dns_support  ||  bool  |  DNS解決  |
|  enable_dns_hostnames  ||  bool  |  DNSホスト名  |
|  tags  ||  object  |  タグ  |

````terraform
resource "aws_vpc" "vpc" {
  cidr_block                       = "10.0.0.0/20"
  instance_tenancy                 = "default"
  enable_dns_support               = true
  enable_dns_hostnames             = true
  assign_generated_ipv6_cidr_block = false

  tags = {
    Name = "customer-db-${var.env}"
  }
}
````

## サブネットの作成

### 作成理由

インターネットに公開するかしないかを決めるため

サブネット

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  vpc_id  |  ●  |  string  |  VPC ID  |
|  availability_zone  ||  string  |  AZ  |
|  cidr_block  |  ●  |  string  |  CIDRブロック  |
|  map_public_ip_on_launch  ||  bool  |  自動割り当てIP設定  |
|  tags  ||  object  |  タグ  |

パブリック
````terraform
resource "aws_subnet" "public-1a" {
  vpc_id                  = aws_vpc.vpc.id
  availability_zone       = "ap-northeast-1a"
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true #注意

  tags = {
    Name = "customer-db-${var.env}-public-1a"
    Type = "public"
    Env  = var.env
  }
}
````

プライベート
````terraform
resource "aws_subnet" "private-1a" {
  vpc_id                  = aws_vpc.vpc.id
  availability_zone       = "ap-northeast-1a"
  cidr_block              = "10.0.2.0/24"
  map_public_ip_on_launch = false #注意

  tags = {
    Name = "customer-db-${var.env}-private-1a"
    Type = "private"
    Env  = var.env
  }
}
````

## ルートテーブル、ルートテーブルアソシエーション  

### 作成理由

ネットワークの経路を設定するコンポーネント。サブネット内の通信がどの宛先のネットワークに対して、どのコンポーネント(IGWやEC2)に転送されてほしいか設定。VPCとサブネットを紐づける。

ルートテーブル

枠を作成する

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  vpc_id  |  ●  |  string  |  VPC ID  |
|  tags  ||  object  |  タグ  |

````terraform
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "customer-db-${var.env}-public-rt"
    Type = "public"
    Env  = var.env
  }
}
````

ルートテーブルアソシエーション

枠に入れるサブネットを選択

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  route_table_id  |  ●  |  string  |  ルートテーブル ID  |
|  subnet_id  |  ●  |  string  |  サブネット ID  |
|  tags  ||  object  |  タグ  |

````terraform
resource "aws_route_table_association" "public_rt_1a" {
  route_table_id = aws_route_table.public_rt.id
  subnet_id      = aws_subnet.public_subnet_1a.id
}
````

## インターネットゲートウェイ(以下、IGWと省略)

### 作成理由

VPCに接続し、ルートテーブルと関連づけて、インターネットに出られるようにする。

IGW

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  vpc_id  |  ●  |  string  |  VPC ID  |
|  tags  ||  object  |  タグ  |

````terraform
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "customer-db-${var.env}-igw"
    Env  = var.env
  }
}
````

IGWアソシエーション

IGWとルートテーブルを接続

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  route_table_id  |  ●  |  string  |  ルートテーブル ID  |
|  destination_cidr_block  |  ●  |  string  |  送信先  |
|  gateway_id  |  ●  |  string  |  IGW ID  |

````terraform
resource "aws_route" "public_rt_igw_r" {
  route_table_id         = aws_route_table.public_rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}
````

## セキュリティーグループ

### 作成理由

インバウンド・アウトバウンドを定義

セキュリティグループ(箱)

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  name  ||  string  |  タグ  |
|  description  ||  string  |  説明  |
|  vpc_id  ||  string  |  VPC ID  |
|  tags  ||  object  |  タグ  |

````terraform
resource "aws_security_group" "web_sg" {
  name = "${var.env}-web-sg"
  description = "web front security group"
  vpc_id = aws_vpc.vpc.id

    tags = {
    Name = "customer-db-${var.env}-web-sg"
    Env  = var.env
  }
}
````


セキュリティグループルール(インバウンド、アウトバウンド)

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  security_group_id  |●|  string  |  セキュリティグループID  |
|  type  |●|  enum  |  ingress,egraee  |
|  protocol  |●|  enum  |  tcp,udp,icmp  |
|  from_port  |●|  number  |  開始ポート  |
|  to_port  |●|  number  |  終了ポート  |
|  cidr_blocks  ||  string  |  CIDRブロックを指定  |
|  source_security_group_id  ||  string  |  セキュリティグループIDを指定 |

````terraform
resource "aws_security_group_rule" "web_in_https" {
  security_group_id = aws_security_group.web_sg.id
  type = "ingress"
  protocol = "tcp"
  from_port = 443
  to_port = 443
  cidr_blocks = [ "0.0.0.0/0" ] #どちらか一方を使用する
  source_security_group_id = aws_security_group.app_sg.id #どちらか一方使用する
}
````

※source_security_group_id

ingressのときとegressで挙動が違う

|    |  ingress  |  egress  |
|  ----  |  ----  |  ----  |
|  source_security_group_id  |  送信元を指定  |  送信先を指定  |
