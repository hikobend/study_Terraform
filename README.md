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

|  コード  |  詳細  |
|  ----  |  ----  |
|required_version|terraformのバージョン|
|required_providers|awsの設定|
|source|hashicorpのものを使用|
|version|awsのバージョン|

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
変数の型を設定

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

