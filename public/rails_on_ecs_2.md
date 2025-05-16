---
title: 既存RailsアプリをECSに初めてデプロイ(CI/CD)したときの手順②【CFnでAWSリソース作成】
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

RailsアプリをはじめてECS（EC2起動タイプ）にデプロイした時の記録です。
全体の手順としては、以下の通りです。

|     |                                        | 
| --- | -------------------------------------------- | 
| 1   | [scaffoldアプリ作成＆Docker化]() | 
| 2   | CFnでAWSリソース作成 👈 この記事                    | 
| 3   | [CI/CDの設定]()                                  | 


この記事では、CloudFormation(略してCFn)でデプロイ先のAWSリソースを作っていきます。


ここでの流れとしては、
[1. CFnでのリソース作成① (VPC, RDS, ELB, CloudFront, ECR etc.)](#1-cfnでのリソース作成-概略と注意点など)
↓
[2. scaffoldアプリのimageをECRへプッシュ](#2-scaffoldアプリのimageをecrへプッシュ)
↓
[3. CFnでのリソース作成② (EC2 Launch Template & Auto Scaling Group, ECS)](#3-cfnでのリソース作成-概略と注意点など-1)
↓
[4. RDSとの接続設定](#4-rdsのdbユーザを作成)
  
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


### 1. CFnでのリソース作成① (概略と注意点など)

CloudFormationを使って作成するAWSリソース①は、Route53, ACM, VPC, IAM, RDS, Secret Manager, ELB, CloudFront, S3, ECRです。（[上記のリポジトリ](https://github.com/sarii0213/lean_up/tree/main/deploy)内のviacdnディレクトリ以外のYAMLファイル）
Secret Managerのリソース作成後には、[こちら](#secrets-manager)にも書いてある通り、scaffoldアプリの`config/master.key`の値を`RAILS_MASTER_KEY`として保管します。

これらを作成後、ECRにscaffoldアプリのDockerイメージをプッシュします。（[→次の手順](#scaffoldアプリのdockerイメージをecrへプッシュ)）

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


### 2. scaffoldアプリのDockerイメージをECRへプッシュ

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

### 3. CFnでのリソース作成② (概略と注意点など)

ECRにscaffoldアプリのDockerイメージをプッシュできたら、再度CloudFormationにてリソースを作成します。
作るAWSリソースは、EC2(Security Group, Launch Template, Auto Scaling), ECS (Cluster, Task Definition, Service)です。（リポジトリのviacdnディレクトリ内のYAMLファイル）

すべてのAWSリソースが作成できたら、DBとの接続ができるよう設定します。（[→次の手順](#rdsのdbユーザを作成)）

:::note
ECS Serviceのデプロイに謎に失敗してロールバックしてしまう場合の対処法
ECS Serviceのリソースを作成直後に、update serviceでdesired task: 0に設定するとデプロイは成功するのでコンテナに入ってデバッグしたり、デプロイ失敗の原因究明ができます。
デバッグ後にdesired task: 1にしてデプロイ成功するか様子を見て、試行錯誤していけます。
:::

#### Security Group
- ELB - ECS Task - RDS 間のトラフィック許可設定（ファイアウォール的役割）

#### Launch Template
- 起動するEC2インスタンスのデフォルトの設定
- EC2インスタンスがECSインスタンスとして機能できるようにするための設定：
  - User dataにて、ECSコンテナエージェントの設定値を入れる

#### Auto Scaling Group
- スケーリングの設定（インスタンスの起動数をどのタイミングで何台に増減させるか）

#### ECS Cluster
- ECS Service, ECS Taskの外枠

#### ECS Task Definition
- ECS Serviceが起動するTaskの定義（Task：コンテナの集まり）
- リバースプロキシサーバ（nginx）コンテナを設置しなかった理由：
  - CloudFront, ALBがnginxの役割を十分になってくれているから
    - CloudFront: 静的ファイル配信（静的コンテンツのキャッシュ, edge locationでの高速配信）, SSL終端, 強制HTTPS化
    - ALB: ルーティング, SSL終端, 強制HTTPS化
    - nginxの機能：リバースプロキシ（クライアントからのreqをbackend app serverに中継, ロードバランシング）←alb, SSL終端←cf&alb, 静的ファイル配信←cf, キャッシュ機能←cf, リクエスト制御・セキュリティ←cf, ヘルスチェック←alb, ルーティング←alb

#### ECS Service
- Taskを常時起動する仕組み
- EC2起動タイプとFargateのうち、EC2起動タイプを選んだ理由：
  - EC2起動タイプのデメリットがEC2の用意や管理の手間があること。インフラ, AWSの勉強のためにその手間を負いたかったから。
  - 上記の理由がなければ、FargateはEC2起動タイプと同様にECS Execでコンテナ内でのデバッグができるし、手間的にも料金的（短期間・小規模のため）にもFargateに軍配が上がりそう。


###  4. RDSのDBユーザを作成

RDSとの接続のために、DBユーザの作成をしていきます。
Task DefinitionでPostgresのTaskを定義して、AWS CLIもしくはAWSマネジメントコンソールでTaskを実行します。
実行時に、DBユーザ作成コマンドを実行するようオーバーライドします。

#### PostgresのTask定義
下記のテンプレートをS3にアップロードし、CFnでスタック作成します。

<details><summary>psql-task.yml</summary>

```yaml:psql-task.yml
AWSTemplateFormatVersion: "2010-09-09"
Description: Create task to execute psql command

Parameters:
  SystemName:
    Description: System Name
    Type: String
    Default: leanup
  Environment:
    Description: Environment
    Type: String
    Default: prod
    AllowedValues:
      - prod
      - stg
      - dev
  ResourceName:
    Description: Resource Name
    Type: String
    Default: viacdn

Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: Environment Configuration
        Parameters:
          - SystemName
          - Environment
          - ResourceName

Resources:
  PsqlExecTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: psql-exec-task
      RequiresCompatibilities:
        - EC2
      NetworkMode: bridge
      ExecutionRoleArn: !ImportValue iam-role-AmazonECSTaskExecutionRoleArn
      Cpu: '256'
      Memory: '512'
      ContainerDefinitions:
        - Name: psql
          Image: postgres:15
          EntryPoint: ["sh", "-c"]
          Command: ["sleep 3600"] # 後で run-task 時に上書きする
          Essential: true
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group:
                Fn::ImportValue: !Sub ${SystemName}-${Environment}-${ResourceName}-ecs-service-LogsLogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: psql

```
</details>

#### Taskの実行
マネジメントコンソールにて、DBユーザ作成コマンドで上書きしてTaskの実行をします。
（DB作成は、デプロイ時にdocker-entrypointに書かれた`db:prepare`にて行われます。）
下記の`<db_user>`, `<db_password>`は、Secret Managerに保管してある非ルートユーザの値を入れます。

<details><summary>マネジメントコンソール画面（ECS Cluster選択 > Tasks > Run new task > Container overrides > psql）にて、DBユーザ作成コマンドとルートユーザのDBパスワードを入力し、Task実行</summary>

![psql_task_exec.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2669070/7585156d-ea7b-4ab1-981c-499f6236fcc9.png)
</details>


  ```bash
  # DBユーザ作成コマンド（作成できたかの確認コマンドも含む）
  psql -h <db_instance_endpoint> -U root -d postgres -c "CREATE USER <db_user> WITH ENCRYPTED PASSWORD '<db_password>'; ALTER USER <db_user> CREATEDB; SELECT * FROM pg_catalog.pg_user;"
  ```
ECS Cluster画面 > Tasks > Run new task > Environment variables overrides にて、Keyに`PGPASSWORD`, ValueにSecret ManagerのRDSのルートユーザのDBパスワードの値をコピペします。


### 挙動確認
自身のドメイン名でアクセスしてブラウザで正常に表示できるか確認します。
アプリの表示を確認できたら、サンプルアプリの初回デプロイ達成です 🎉

次に、[CI/CDの設定]()をしていきます！
