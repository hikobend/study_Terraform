# 学習ステップ

1. 構成図を作成する

2. 構成図を描画

3. コード一文ごとの内容をメモする

4. resourceごとに表を作成する

5. 1〜4を繰り返す

## terraform初期設定
・Terraform用IAMユーザーを作成

・ターミナルにIAMユーザーを登録(aws configure)

・.gitignoreファイル作成(https://www.toptal.com/developers/gitignore/api/terraform)

・terraform initをして初期化

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

## tfstateを使ったコマンド

|  コード  |  内容  |
|  ----  |  ----  |
|  terraform state list ADDRESS <br> ADDRESS : 絞り込みたいリソース名 |  tfstateのリソースを一覧表示する  |
|  terraform state show ADDRESS <br> ADDRESS : 詳細情報を確認したいリソース名 |  指定されたリソース名の詳細情報を確認  |
|  terraform state mv SOURCE DESTINATION <br> SOURCE : 移動元リソース名 <br> DESTINATION : 移動先リソース名 |  指定されたリソース詳細を移動  |
|  terraform import ADDRESS ID <br> ADDRESS : 取り込み先のリソース名 <br> ID : 取り込みたい稼働中リソースID |  指定されたリソースを取り込む  |
|  terraform state rm ADDRESS ADDRESS : 管理対象外にしたいリソース名 |  指定されたリソースを管理対象外  |
|  terraform refresh |  現在のクラウド上の状態をステートファイルへ反映する  |

## バージョン設定コード

````terraform

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "us-east-1"
}
````

https://www.terraform.io/language/providers/requirements

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

# Configure the AWS Provider
provider "aws" {
  region = "ap-northeast-1"
}

````
## 必要な設定

・tfstateをS3に移動

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

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  backend  |  ●  |  block  |  bucket : バケット名<br>key : キー<br>region : リージョン<br>profile : プロフィール  |

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
    bucket  = "custoemr-db-bucket"
    key     = "customer-db-dev.tfstate"
    region  = "ap-northeast-1"
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "ap-northeast-1"
}
````


保存先を変更するため、
````terminal
terraform init
````

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

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)

通信をする上で必須

![vpc](https://user-images.githubusercontent.com/92671446/186351089-b6d1e105-ab0a-4ad9-9216-682eaffd1927.png)

### 公式ページのVPCサンプル

````terraform
resource "aws_vpc" "main" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "main"
  }
}
````

### 代表的なリファレンス

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  cidr_block  |  ●  |  string  |  IPv4CIDRブロック  |
|  enable_dns_hostnames  ||  bool  |  DNSホスト名  |
|  tags  ||  object  |  タグ  |

````terraform
resource "aws_vpc" "vpc" {
  cidr_block                       = "10.0.0.0/20"
  enable_dns_hostnames             = true

  tags = {
    Name = "customer-db-${var.env}-vpc"
    Env  = var.env
  }
}
````

[DNSホスト名を有効にした方が良い理由](https://qiita.com/fumiya-konno/items/f94ed3e3c114793c898a)

## サブネットの作成

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)

機能説明

![サブネット](https://user-images.githubusercontent.com/92671446/186549564-59337186-8197-45ab-b566-a1b2883e1a21.png)


### 公式ページのコードサンプル

````terraform
resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "Main"
  }
}
````

### 代表的なリファレンス

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  vpc_id  |  ●  |  string  |  設置するVPCを場所  |
|  availability_zone  |  ●  |  string  |  設置するAZを指定  |
|  cidr_block  |  ●  |  number(?)  |  サブネットの範囲  |
|  map_public_ip_on_launch  |  ●  |  bool  |  パブリックIPを設定するか  |

````terraform
resource "aws_subnet" "public_subnet_1a" {
  vpc_id                  = aws_vpc.vpc.id
  availability_zone       = "ap-northeast-1a"
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true

  tags = {
    Name = "customer-db-${var.env}-public-1a"
    Type = "public"
    Env  = var.env
  }
}
````


````terraform
resource "aws_subnet" "private_subnet_1a" {
  vpc_id                  = aws_vpc.vpc.id
  availability_zone       = "ap-northeast-1a"
  cidr_block              = "10.0.3.0/24"
  map_public_ip_on_launch = false

  tags = {
    Name = "customer-db-${var.env}-private-1a"
    Type = "private"
    Env  = var.env
  }
}
````

### 実装時の注意点など

パブリックIPを許可・拒否することで、Publicサブネットかprivateサブネットを設定する。

## ルートテーブル作成

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table)

機能説明

![アーキテクチャ](https://user-images.githubusercontent.com/92671446/186557734-90ed2e7d-b653-44f4-9183-5ede7c3caf7f.png)

### 公式ページのコードサンプル

````terraform
resource "aws_route_table" "example" {
  vpc_id = aws_vpc.example.id

  route {
    cidr_block = "10.0.1.0/24"
    gateway_id = aws_internet_gateway.example.id
  }

  route {
    ipv6_cidr_block        = "::/0"
    egress_only_gateway_id = aws_egress_only_internet_gateway.example.id
  }

  tags = {
    Name = "example"
  }
}
````

### 代表的なリファレンス

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  vpc_id  |  ●  |  string  |  設置するVPCを指定  |

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

### 実装時の注意点など

アソシエーションも作成する。

## ルートテーブルアソシエーション作成

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table_association)

### 公式ページのコードサンプル

````terraform
resource "aws_main_route_table_association" "a" {
  vpc_id         = aws_vpc.foo.id
  route_table_id = aws_route_table.bar.id
}
````

### 代表的なリファレンス

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  vpc_id  |  ●  |  string  |  設置するVPCを指定  |

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

## IGWの作成

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway
)

機能説明

![名称未設定](https://user-images.githubusercontent.com/92671446/186575981-e7c1c6d8-1e74-471a-9622-d881a2c78734.png)

### 公式ページのコードサンプル

````terraform
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main"
  }
}
````

### 代表的なリファレンス

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  vpc_id  |  ●  |  string  |  設置するVPC  |

````terraform
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "customer-db-${var.env}-igw"
    Env  = var.env
  }
}
````

### 実装時の注意点など

## IGWアソシエーションの作成

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route)

機能説明

IGWとルートテーブルを繋げる

### 公式ページのコードサンプル

````terraform
resource "aws_route" "r" {
  route_table_id            = "rtb-4fbb3ac4"
  destination_cidr_block    = "10.0.1.0/22"
  vpc_peering_connection_id = "pcx-45ff3dc1"
  depends_on                = [aws_route_table.testing]
}
````

### 代表的なリファレンス

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  route_table_id  |  ●  |  string  |  ルーティングするテーブルのID  |
|  destination_cidr_block  |  ●  |  string  |  宛先CIDRブロック  |
|  gateway_id  |    |  string  |  VPCインターネットゲートウェイか仮想プライベートゲートウェイ  |

````terraform
resource "aws_route" "public_rt_igw_r" {
  route_table_id         = aws_route_table.public_rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}
````

## セキュリティグループの作成

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group)

機能説明

![アーキテクチャ](https://user-images.githubusercontent.com/92671446/186593375-e1a03e8a-d057-4fe4-8b44-be03552d5701.png)

### 公式ページのコードサンプル

````terraform
resource "aws_security_group" "allow_tls" {
  name        = "allow_tls"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    description      = "TLS from VPC"
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = [aws_vpc.main.cidr_block]
    ipv6_cidr_blocks = [aws_vpc.main.ipv6_cidr_block]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "allow_tls"
  }
}
````

### 代表的なリファレンス

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  〇〇  |  ●  |  string  |  〇〇  |

````terraform
resource "aws_security_group" "web_sg" {
  name        = "${var.env}-web-sg"
  description = "web front security group"
  vpc_id      = aws_vpc.vpc.id

  tags = {
    Name = "customer-db-${var.env}-web-sg"
    Env  = var.env
  }
}
````

### 実装時の注意点など



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

・source_security_group_id

ingressのときとegressで挙動が違う

|    |  ingress  |  egress  |
|  ----  |  ----  |  ----  |
|  source_security_group_id  |  送信元を指定  |  送信先を指定  |

## RDSの作成

### 必要な設定
・パラメータグループ

・オプショングループ

## パラメータグループの作成

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_parameter_group)

機能説明

データベースサービスのコンフィグ（パラメーター）を管理する設定です。ジョブ最大数や、I/O操作で読み込むブロックの最大数等の、チューニング関連

![アーキテクチャ](https://user-images.githubusercontent.com/92671446/186810476-c7ee1899-58a4-402e-bdc0-e751867a7167.png)

### 公式ページのコードサンプル

````terraform
resource "aws_db_option_group" "example" {
  name                     = "option-group-test-terraform"
  option_group_description = "Terraform Option Group"
  engine_name              = "sqlserver-ee"
  major_engine_version     = "11.00"

  option {
    option_name = "Timezone"

    option_settings {
      name  = "TIME_ZONE"
      value = "UTC"
    }
  }

  option {
    option_name = "SQLSERVER_BACKUP_RESTORE"

    option_settings {
      name  = "IAM_ROLE_ARN"
      value = aws_iam_role.example.arn
    }
  }

  option {
    option_name = "TDE"
  }
}
````

### 代表的なリファレンス

|  コード  |  必須  |  型  |  詳細  |
|  ----  |  :--: |  :--:  |  ----  |
|  〇〇  |  ●  |  string  |  〇〇  |

````terraform
実際に作成したコード
````

### 実装時の注意点など

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
|  name  |    |  string  |  オプショングループ名  |
|  engine_name  |  ●  |  string  |  関連づけるエンジン名(mysqlなど)  |
|  major_engine_version  |    |  string  |  関連づけるエンジンバージョン(5.7,8.0など)  |
|  tags  |    |  object  |  タグ  |

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

無料利用枠の対象のAMIを所得する

````
aws ec2 describe-images --image-ids ami-0ecb2a61303230c9d
````

dataブロックを使用してAMIを所得する。

````terraform
data "aws_ami" "app" {
  most_recent = true
  owners      = ["self", "amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-kernel-5.10-hvm-2.0.*-x86_64-gp2"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
````

#### キーペアの生成

1. ssh-keygenでキーペア作成
2. キーペアリソースをコード記述
3. AWSへキーペアを登録

認証用の鍵を生成・管理・変換する

````
ssh-keygen [-t <TYPE>][-b <BITS>][-f <KEYFILE>]
````

|  引数  |  内容  |
|  ----  |  ----  |
|  -t <TYPE>  |  鍵の種類。"rsa", "dsa"など  |
|  -b <BITS>  |  鍵のビット数  |
|  -f <KEYFILE>  |  出力先鍵ファイル  |

aws_key_pair

|  コード  |  必須  |  型  |  説明  |
|  ----  |  :--:  |  :--:  |  ----  |
|  key_name  ||  string  |  キーペア名  |
|  public_key  |  ●  |  string  |  公開鍵  |
|  tags  |    |  object  |  タグ  |

#### EC2
1. 基本設定
2. ネットワーク
3. その他

aws_instance(基本設定)

|  コード  |  必須  |  型  |  説明  |
|  ----  |  :--:  |  :--:  |  ----  |
|  ami  |  ●  |  string  |  AMI ID  |
|  instance_type  |  ●  |  enum  |  インスタンスタイプ "t2micro"など  |
|  tags  |    |  object  |  タグ  |

aws_instance(ネットワーク)

|  コード  |  必須  |  型  |  説明  |
|  ----  |  :--:  |  :--:  |  ----  |
|  subnet_id  |    |  string  |  サブネット ID  |
|  associate_public_ip_address  |    |  bool  |  自動割り当てパブリックIP  |
|  vpc_security_group_ids  |    |  string  |  セキュリティグループID  |

aws_instance(その他)

|  コード  |  必須  |  型  |  説明  |
|  ----  |  :--:  |  :--:  |  ----  |
|  iam_instance_profile  |    |  string  |  IAMロール  |
|  key_name  |    |  string  |  キーペア名  |
|  user_data  |    |  string  |  ユーザーデータ  |

## ALBの追加

````terraform
resource "aws_lb" "test" {
  name               = "test-lb-tf"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.lb_sg.id]
  subnets            = [for subnet in aws_subnet.public : subnet.id]

  enable_deletion_protection = true

  access_logs {
    bucket  = aws_s3_bucket.lb_logs.bucket
    prefix  = "test-lb"
    enabled = true
  }

  tags = {
    Environment = "production"
  }
}
````
