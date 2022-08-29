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

|  コード  |  詳細  |
|  ----  |  ----  |
|  backend  |  bucket : バケット名<br>key : キー<br>region : リージョン<br>profile : プロフィール  |

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

|  コード  |  詳細  |
|  ----  |  ----  |
|  cidr_block  |  IPv4CIDRブロック  |
|  enable_dns_hostnames  |  DNSホスト名  |
|  tags  |  タグ  |

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

|  コード  |  詳細  |
|  ----  |  ----  |
|  vpc_id  |  設置するVPCを場所  |
|  availability_zone  |  設置するAZを指定  |
|  cidr_block  |  サブネットの範囲  |
|  map_public_ip_on_launch  |  パブリックIPを設定するか  |

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

|  コード  |  詳細  |
|  ----  |  ----  |
|  vpc_id  |  設置するVPCを指定  |

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

|  コード  |  詳細  |
|  ----  |  ----  |
|  vpc_id  |  設置するVPCを指定  |

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

![アーキテクチャ](https://user-images.githubusercontent.com/92671446/186575981-e7c1c6d8-1e74-471a-9622-d881a2c78734.png)

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

|  コード  |  詳細  |
|  ----  |  ----  |
|  vpc_id  |  設置するVPC  |

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

|  コード  |  詳細  |
|  ----  |  ----  |
|  route_table_id  |  ルーティングするテーブルのID  |
|  destination_cidr_block  |  宛先CIDRブロック  |
|  gateway_id  |  VPCインターネットゲートウェイか仮想プライベートゲートウェイ  |

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

|  コード  |  詳細  |
|  ----  |  ----  |
|  〇〇  |  〇〇  |

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

|  コード  |  詳細  |
|  ----  | ----  |
|  security_group_id  |  セキュリティグループID  |
|  type  |  ingress,egraee  |
|  protocol  |  tcp,udp,icmp  |
|  from_port  |  開始ポート  |
|  to_port  |  終了ポート  |
|  cidr_blocks  |  CIDRブロックを指定  |
|  source_security_group_id  |  セキュリティグループIDを指定 |

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

・サブネットグループ

## パラメータグループの作成

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_parameter_group)

機能説明

データベースサービスのコンフィグ（パラメーター）を管理する設定です。ジョブ最大数や、I/O操作で読み込むブロックの最大数等の、チューニング関連

![アーキテクチャ](https://user-images.githubusercontent.com/92671446/186810476-c7ee1899-58a4-402e-bdc0-e751867a7167.png)

### 公式ページのコードサンプル

````terraform
resource "aws_db_parameter_group" "default" {
  name   = "rds-pg"
  family = "mysql5.6"

  parameter {
    name  = "character_set_server"
    value = "utf8"
  }

  parameter {
    name  = "character_set_client"
    value = "utf8"
  }
}
````

### 代表的なリファレンス

|  コード  |  詳細  |
|  ----  |  ----  |
|  name  |  パラメータグループの名前  |
|  family  |  DB パラメータ グループのファミリ  |

````terraform
resource "aws_db_parameter_group" "mysql_parametergroup" {
  name   = "${var.env}-mysql-parametergroup"
  family = "mysql8.0"

  parameter {
    name  = "character_set_database"
    value = "utf8mb4"
  }

  parameter {
    name  = "character_set_server"
    value = "utf8mb4"
  }
}
````

### 実装時の注意点など

character_set_databaseとcharacter_set_serverは必須

## オプショングループの作成

### 必要な設定

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_option_group)

機能説明

データベースサーバで使用できる機能を設定

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

|  コード  |  詳細  |
|  ----  |  ----  |
|  name  |  RDSの名前  |
|  engine_name  |  オプション グループを関連付けるエンジンの名前を指定  |
|  major_engine_version  |  このオプション グループを関連付けるエンジンのメジャー バージョンを指定  |

````terraform
resource "aws_db_option_group" "mysql_optiongroup" {
  name                 = "${var.env}-mysql-optiongroup"
  engine_name          = "mysql"
  major_engine_version = "8.0"
}
````

### 実装時の注意点など

## サブネットグループの作成

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_subnet_group)

機能説明

### 公式ページのコードサンプル

````terraform
resource "aws_db_subnet_group" "default" {
  name       = "main"
  subnet_ids = [aws_subnet.frontend.id, aws_subnet.backend.id]

  tags = {
    Name = "My DB subnet group"
  }
}
````

### 代表的なリファレンス

|  コード  |  詳細  |
|  ----  |  ----  |
|  name  |   サブネットグループの名前  |
|  subnet_ids  |  RDSを格納するsubnetの場所  |

````terraform
resource "aws_db_subnet_group" "mysql_subnetgroup" {
  name = "${var.env}-mysql-subnetgroup"
  subnet_ids = [
    aws_subnet.private_subnet_1a.id,
    aws_subnet.private_subnet_1c.id
  ]

  tags = {
    Name = "customer-db-${var.env}-mysql-subnetgroup"
    Env  = var.env
  }
}
````

### 実装時の注意点など

## RDSの作成

[公式](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance)

機能説明

### 公式ページのコードサンプル

````terraform
resource "aws_db_instance" "default" {
  allocated_storage    = 10
  engine               = "mysql"
  engine_version       = "5.7"
  instance_class       = "db.t3.micro"
  name                 = "mydb"
  username             = "foo"
  password             = "foobarbaz"
  parameter_group_name = "default.mysql5.7"
  skip_final_snapshot  = true
}
````

### 代表的なリファレンス

|  コード  |  詳細  |
|  ----  |  ----  |
|  engine  |  指定するデータベースエンジン  |
|  engine_version   |  使用するエンジンのバージョン  |
|  identifier  |  RDSインスタンスの名前  |
|  instance_class  |  RDSのインスタンスタイプ  |
|  username  |  データベースのマスターユーザー  |
|  password  |  パスワード  |
|  allocated_storage  |  割り当てたストレージ  |
|  max_allocated_storage  |  RDSが自動的にスケーリングできる上限  |
|  storage_type  |  standard gp2 io1のいずれか  |
|  storage_encrypted  |  DBインスタンスが暗号化されているか  |
|  multi_az  ||  RDSインスタンスがマルチAZかどうか指定  |
|  availability_zone  |  RDSインスタンスのAZ  |
|  db_subnet_group_name  |  DBサブネット名  |
|  vpc_security_group_ids  |  関連付けるDBセキュリティグループのリスト  |
|  publicly_accessible  |  インスタンスが公開されているか制御  |
|  port  |  DBが接続を受け入れるポート  |
|  parameter_group_name  |  関連づけるDBパラメータグループ  |
|  option_group_name  |  関連づけたオプショングループ  |
|  backup_window  |  自動バックアップが有効になっている場合に作成される毎日の時間範囲  |
|  backup_retention_period  |  バックアップを保持する日数  |
|  maintenance_window  |  メンテナンスを実行するウィンドウ  |
|  deletion_protection  |  DBインスタンスで削除保護を有効にする必要がある場合に設定|
|  skip_final_snapshot  |  DBインスタンスを削除する前に、最終的なDBスナップショットを作成するか  |
|  apply_immediately  |  データベースの変更をすぐに適応するか指定  |

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

  deletion_protection = true
  skip_final_snapshot = false
  apply_immediately   = true

  tags = {
    Name = "customer-db-${var.env}-mysql"
    Env  = var.env
  }
}
````

### 実装時の注意点など

## EC2の作成

### 必要な設定
・AMI

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance)

機能説明

サーバー

![アーキテクチャ](https://user-images.githubusercontent.com/92671446/186848388-28221914-392e-4514-a30e-33b5f1c1bf52.png)


### 公式ページのコードサンプル

````terraform
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name = "HelloWorld"
  }
}
````
## AMIのイメージ取得

[公式ページ](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance)

機能説明

サーバーを立てるために使用

### 呼び出すためのプログラム

````
aws ec2 describe-images --image-ids ami-xxxxxx
````

### 公式ページのコードサンプル

````terraform
data "aws_ami" "example" {
  executable_users = ["self"]
  most_recent      = true
  name_regex       = "^myami-\\d{3}"
  owners           = ["self"]

  filter {
    name   = "name"
    values = ["myami-*"]
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
