---
title: 既存RailsアプリをECSに初めてデプロイ(CI/CD)したときの手順②【CloudFormationでリソース作成】
tags:
  - 'Rails'
  - 'ECS'
  - 'CI/CD'
  - 'GitHub Actions'
  - 'CloudFormation'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

CloudFormation(略してCFn)でデプロイ先のAWSリソースを作っていきます。

ここでの流れとしては、
[CFnでのリソース作成① (VPC, RDS, ELB, CloudFront, ECR etc.)](#cfnでのリソース作成-概略と注意点など)
↓
[scaffoldアプリのimageをECRへプッシュ](#scaffoldアプリのimageをecrへプッシュ)
↓
[CFnでのリソース作成② (EC2 Launch Template & Auto Scaling Group, ECS)](#cfnでのリソース作成-概略と注意点など-1)
  
という順に行なっていきます。
今回作成したテンプレートの全YAMLファイルはこちら↓にあります。

https://github.com/sarii0213/lean_up/tree/main/deploy

:::note
実際の作業としては、CFnでのリソース作成①②にて列挙している順番に、YAMLファイルひとつひとつを S3へアップロード → AWSマネジメントコンソールにてCFnの"create stack"ボタン押下（テンプレートにはS3 URLを指定）からAWSリソースを作っていきます。
:::

<details><summary>作成の手順も例示する？（自分の備忘録として）❗️</summary>

</details>


<details><summary>作成するawsリソースの構成図 (todo: 自作する ❗️)</summary>
todo: 自作する ❗️
</details>


### CFnでのリソース作成① (概略と注意点など)

#### Route53
- [Value Domain](https://www.value-domain.com/)にて取得済みのドメイン名をHosted Zoneとして登録。CAAレコードも作成。（SSL/TLS証明書の不正発行防止のため）
- Route53にて生成されるNSレコードをValue Domain側に入力（名前解決の際に取得したドメイン名の権威サーバがAWSの権威サーバのIPを返すようにするため）

#### ACM (Certificate Manager)
- SSL/TLS証明書を発行・管理するリソース（SSL証明書：HTTPS通信において必須）
- ACMは証明書発行時に、AmazonのCAを発行元として許可しているかCAAレコードを確認する

#### VPC
- ネットワークの外枠
- この中に、RDS, ELB, CloudFront Origin, ECS Cluster等を作っていく

#### IAM
- ECS task role, task execution roleを作成（＝task自身が持つ権限, task起動時に必要な権限）
- ECS Execの権限を作成。（ECS Exec：ECS Taskの外側からTask内での処理を命令できる機能） 
- （※ AWSを操作するためのIAM Userは事前に作成済み）

#### RDS
- ECS Task内にDB用のコンテナを立てることも可能だが、AWSに管理を丸投げできて運用負荷抑えられる ＆ EC2上で自前のDBを構築するよりRDS Auroraの方がパフォーマンスや信頼性の面で最適化されているためRDSを使用

#### Secrets Manager
- CloudFrontで使うカスタムヘッダの値＆DBの認証情報を保管
- `RAILS_MASTER_KEY`も保管 \:point_left: scaffoldアプリの`config/master.key`の値をコピペする
  （`RAILS_MASTER_KEY`：アプリ内のcredentialsを復号するための暗号鍵）

#### ELB
- トラフィックをパス毎に振り分け, health check, http → httpsのリダイレクト を設定

#### CloudFront
- キャッシュ配信とエッジロケーションでレスポンスが高速化
- 「静的ファイルはS3へ, それ以外はELBへ」とトラフィックの振り分けも行う

#### S3
- アプリの静的ファイル用とログ用のファイル置き場

#### ECR
- Docker Imageを保管する場所
- ローカルからECRにimageをプッシュして、それをECS Task Definitionにて参照する


### scaffoldアプリのimageをECRへプッシュ

scaffoldアプリのイメージをビルドし、ECRにプッシュします。
ビルドからプッシュまでのコマンドは、AWSマネジメントコンソールのECRの画面 > View push commands でも確認できます。

1. Dockerfileからimageをbuild
  ```sh
  docker build --platform linux/x86_64 .
  # デプロイ先の環境に対応させるためにplatformを指定
  # （自PCのCPUアーキテクチャがArm, デプロイ先はx86_64）
  ```
1. ecrへのpushができるよう認証 
  ```sh
  aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com
  # <account_id> = 自分のAWS account ID
  ```

1. buildしたimageにタグ付け
  ```sh
  docker tag <built_image_name>:latest <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com/<built_image_name>:latest
  # Docker Imageはpushする時にpush先をtagで判断する。ECRにpushするにはそのECRのURLを含む名前でタグ付けしておく必要がある。
  # <built_image_name> = ビルドしたImage名
  ```
  
1. ECRにimageをpush
   ```sh
   docker push <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com/<built_image_name>:latest
   ```

### CFnでのリソース作成② (概略と注意点など)
- launch template
- ec2 auto scaling
- ecs (cluster, task definition, service) 
