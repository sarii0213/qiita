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

CloudFormationを使って作成するAWSリソース①は、Route53, ACM, VPC, IAM, RDS, Secret Manager, ELB, CloudFront, S3, ECRです。これらを作成後、ECRにscaffoldアプリのDockerイメージをプッシュします。（[→次の手順へ](#scaffoldアプリのdockerイメージをecrへプッシュ)）

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
- トラフィックをパス毎に振り分け, health check, http→httpsのリダイレクト を設定

#### CloudFront
- キャッシュ配信とエッジロケーションでレスポンスが高速化
- 「静的ファイルはS3へ, それ以外はELBへ」とトラフィックの振り分けも行う

#### S3
- アプリの静的ファイル用 & ログ用のファイル置き場

#### ECR
- Docker Imageを保管する場所
- ローカルからECRにimageをプッシュして、それをECS Task Definitionにて参照する


### scaffoldアプリのDockerイメージをECRへプッシュ

ビルドからプッシュまでのコマンドは、AWSマネジメントコンソールのECRの画面 > View push commands でも確認できます。

1. Dockerfileからイメージをビルド
    ```sh
    docker build --platform linux/x86_64 .
    # デプロイ先の環境に対応させるためにplatformを指定
    # （自PCのCPUアーキテクチャがArm(macOS Apple Silicon), デプロイ先はx86_64）
    ```
    <br>

2. ECRへのプッシュができるよう認証 
    ```sh
    aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com
    # <account_id> = 自分のAWS account ID
    ```
    <br>

3. ビルドしたイメージにタグ付け
    ```sh
    docker tag <built_image_name>:latest <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com/<built_image_name>:latest
    # Dockerイメージはpushする時にプッシュ先をタグで判断する。ECRにプッシュするにはそのECRのURLを含む名前でタグ付けしておく必要がある。
    # <built_image_name> = ビルドしたイメージ名
    ```
    <br>
  
1. ECRにイメージをプッシュ
   ```sh
   docker push <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com/<built_image_name>:latest
   ```

### CFnでのリソース作成② (概略と注意点など)

ECRにscaffoldアプリのDockerイメージをプッシュできたら、再度CloudFormationにてリソースを作成します。
作るAWSリソースは、EC2(Security Group, Launch Template), Auto Scaling Group, ECS (Cluster, Task Definition, Service)です。

#### Security Group
- アクセス制御（ファイアウォール的役割）
#### Launch Template
- 起動するEC2インスタンスの設定
#### Auto Scaling Group
- スケーリングの設定（どのタイミングで何台起動するか）
#### ECS Cluster
- ECS Service, ECS Taskの外枠
#### ECS Task Definition
- ECS Serviceが起動するTaskの定義（各コンテナの設定）
#### ECS Service
- Taskを常時起動する仕組み。
- EC2起動タイプとFargateのうち、EC2起動タイプを選んだ理由：

