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

aws_route_table

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

aws_route_table_association

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

aws_internet_gateway

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

aws_route

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

aws_security_group

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

aws_security_group_rule

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  security_group_id  |●|  string  |  セキュリティグループID  |
|  type  |●|  enum  |  ingress,egraee  |
|  protocol  |●|  enum  |  tcp,udp,icmp  |
|  from_port  |●|  number  |  開始ポート  |
|  to_port  |●|  number  |  終了ポート  |
|  cidr_blocks  |どちらか一方|  string  |  CIDRブロックを指定  |
|  source_security_group_id  |どちらか一方|  string  |  セキュリティグループIDを指定 |

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

## RDS

### 作成理由

リレーショナルデーターベースの作成

パラメーターグループ

データベースに割り当てるメモリなどのリソースの量を指定

aws_db_parameter_group

|  コード  |  必須  |  型  |  内容  |
|  ----  |  ----  |  ----  |  ----  |
|  name  ||  string  |  パラメータグループ名  |
|  family  ||  string  |  パラメータグループのバージョン  |
|  parameter  ||  block  |  具体的なパラメータ<br>name : パラメータ名 <br> value : パラメータ値  |
|  tags  ||  object  |  タグ  |

````terraform
resource "aws_db_parameter_group" "mysql_parametergroup" {
  name   = "${var.env}-mysql-parametergroup"
  family = "mysql8.0"

  parameter {
    name  = "character_set_database" #データベースの文字コード設定を追加
    value = "utf8mb4"
  }

  parameter {
    name  = "character_set_server" #データベースの文字コード設定を追加
    value = "utf8mb4"
  }
}
````

オプショングループ

aws_db_option_group

|  コード  |  必須  |  型  |  内容  |
|  ----  |  :--:  |  ----  |  ----  |
|  name  ||  string  |  オプショングループ名  |
|  engine_name  |●|  string  |  関連づけるエンジン名(mysqlなど)  |
|  major_engine_version  ||  string  |  関連づけるエンジンバージョン(5.7,8.0など)  |
|  option  ||  block  |  具体的なオプション設定<br>option_name(必須) : オプション名 <br> option_settings : オプション設定  |
|  tags  ||  object  |  タグ  |

````terraform
resource "aws_db_option_group" "name" {
  name                 = "${var.env}-mysql-optiongroup"
  engine_name          = "mysql"
  major_engine_version = "8.0"
}
````

サブネットグループ

プライベートサブネットを束ねるサブネットグループを作成

|  コード  |  必須  |  型  |  内容  |
|  ----  |  :--:  |  ----  |  ----  |
|  name  ||  string  |  サブネットグループ名  |
|  subnet_ids  |●|  string  |  サブネットID  |
|  tags  ||  object  |  タグ  |

ランダム文字列

パスワードを作成

random_string

|  コード  |  必須  |  型  |  内容  |
|  ----  |  :--:  |  ----  |  ----  |
|  length  |  ●  |  number  |  ランダム文字列の長さ  |
|  upper  ||  bool  |  大文字英字を使うかどうか  |
|  lower  ||  bool  |  小文字文字列を使うかどうか  |
|  number  ||  bool  |  数字を使うかどうか  |
|  special  ||  bool  |  特殊文字を使うかどうか  |
|  override_special  ||  string  |  利用したい特殊文字  |

RDSインスタタンス

aws_db_instance(基本設定)

|  コード  |  必須  |  型  |  内容  |
|  ----  |  :--:  |  ----  |  ----  |
|  engine  |  ●  |  string  |  データベースエンジン  |
|  engine_version  ||  string  |  データベースエンジンのバージョン  |
|  identifier  |  ●  |  string  |  RDSインスタンスリソース名  |
|  instance_class  |  ●  |  string  |  インスタンスタイプ  |
|  username  |  ●  |  string  |  マスターDBのユーザー名  |
|  password  |  ●  |  string  |  マスターDBユーザーのパスワード  |
|  tags  ||  object  |  タグ  |

aws_db_instance(ストレージ)

|  コード  |  必須  |  型  |  内容  |
|  ----  |  :--:  |  ----  |  ----  |
|  allocated_storage  |    |  string  |  割り当てるストレージ(ギガバイト)  |
|  max_allocated_storage  |    |  string  |  オートスケールさせる最大サイズ  |
|  storage_type  |    |  enum  |  "standerd"(磁気),"gp2"(SSD), "io1"(IOPS SSD)  |
|  storage_encrypted  |    |  string  |  DBを暗号化するKMS鍵IDまたはfalse  |

aws_db_instance(ネットワーク)

|  コード  |  必須  |  型  |  内容  |
|  ----  |  :--:  |  :--:  |  ----  |
|  multi_az  |    |  bool  |  マルチAZ配置するかどうか  |
|  availability_zone  ||  string  |  シングルインスタンス時に配置するAZ  |
|  db_subnet_group_name  |    |  string  |  サブネットグループ名  |
|  vpc_security_group_ids  |    |  string  |  セキュリティグループID  |
|  publicky_accessible  |    |  bool  |  パブリックアクセス許可するか  |
|  port  |    |  number  |  ポート番号  |

aws_db_instance(DB設定)

|  コード  |  必須  |  型  |  内容  |
|  ----  |  :--:  |  :--:  |  ----  |
|  name  |    |  string  |  データベース名  |
|  parameter_group_name  ||  string  |  パラメータグループ名  |
|  option_group_name  |    |  string  |  オプショングループ名  |

aws_db_instance(バックアップ)

|  コード  |  必須  |  型  |  内容  |
|  ----  |  :--:  |  :--:  |  ----  |
|  backup_window  |    |  string  |  バックアップを行う時間帯  |
|  backup_retention_period  ||  bool  |  バックアップを残す数  |
|  maintenance_window  |    |  string  |  メンテナンスを行う時間帯  |
|  auto_minor_version_upgrade  |    |  bool  |  自動でマイナーバージョンアップグレードするか  |


aws_db_instance(削除防止)

|  コード  |  必須  |  型  |  内容  |
|  ----  |  :--:  |  :--:  |  ----  |
|  deletion_protection  |    |  bool  |  削除防止するか  |
|  skip_final_snapshot  ||  bool  |  削除時のスナップショットをスキップするか  |
|  apply_immediately  |    |  bool  |  即時反映するか  |

````terraform
resource "aws_db_instance" "mysql" {
  engine         = "mysql"
  engine_version = "8.0.28"
  identifier     = "${var.env}-mysql"
  instance_class = "db.t2.micro"
  username       = "admin"
  password       = random_string.db_password.result

  allocated_storage     = 20
  max_allocated_storage = 50
  storage_type          = "gp2"
  storage_encrypted     = false

  multi_az               = false
  availability_zone      = "ap-northeast-1a"
  db_subnet_group_name   = aws_db_subnet_group.mysql_subnetgroup.name
  vpc_security_group_ids = [aws_security_group.db_sg.id]
  publicly_accessible    = false
  port                   = 3306

  parameter_group_name = aws_db_parameter_group.mysql_parametergroup.name
  option_group_name    = aws_db_option_group.mysql_optiongroup.name

  backup_window              = "04:00-05:00"
  backup_retention_period    = 7
  maintenance_window         = "Mon:05:00-Mon:08:00"
  auto_minor_version_upgrade = false

  deletion_protection = true #
  skip_final_snapshot = false #
  apply_immediately   = true

  tags = {
    Name = "customer-db-${var.env}-mysql"
    Env  = var.env
  }
}
````

RDSを削除するときは、
````terraform
deletion_protection = false #
skip_final_snapshot = true #
````
に設定を変更してから、RDSを削除する

## EC2の作成

### 作成に必要なもの
 
・AMI

・キーペア

・サブネット

・セキュリティグループ

#### AMIの作成

AWS CLIでAMIを検索

````
aws ec2 describe-images[ --image-ids <AMI_ID> ] [ --owners <OWNER> ] [ --filters "Name= <KEY1>, Values= <VALUE1>,<VALUE2>,..." ]
````

|  引数  |  内容  |
|  ----  |  ----  |
|  --images-ids  |  AMI IDを指定  |
|  --owner  |  AMIの所有者を指定。<br>AWSアカウントIDまたはself,amazon,aws-marketplaceを指定。  |
|  --filters  |  AMI検索のフィルター条件を指定。  |

filtersオプションで指定できる検索条件

|  キー  |  説明  |
|  ----  |  ----  |
|  name  |  イメージ名 |
|  description  |  イメージ説明  |
|  root-device-type  |  接続するブロックストレージの種類。基本的に"ebs"  |
|  virtualization-type  |  仮装方式。基本は"hvm"  |

aws_ami

|  コード  |  必須  |  型  |  内容  |
|  ----  |  :--:  |  :--:  |  ----  |
|  owners  |  ●  |  enum  |  所有者。self(自分自身)  |
|  most_recent  ||  bool  |  最新のものを選択するか  |
|  executable_users  |    |  string  |  実行ユーザー。"self"またはアカウントID  |
|  filter  |    |  block  |  検索フィルター<br>name:検索条件<br>values:検索する値|
