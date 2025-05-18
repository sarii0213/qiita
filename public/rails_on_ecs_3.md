---
title: 既存RailsアプリをECSに初めてデプロイ(CI/CD)したときの手順③【CI/CD】
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
| 2   | [CFnでAWSリソース作成]()                     | 
| 3   | CI/CDの設定 👈 この記事                                 | 


この記事では、デプロイの自動化をしていきます。

まずは、scaffoldアプリのソースコードを変更して手動デプロイを行ってその手順を把握してから、デプロイの自動化をしていきます。

今回の流れは、こちら。
[1. ソースコード更新し手動デプロイ]()
↓
[2. CI/CD設定]()
↓
[3. 自前アプリのデプロイ]()
↓
[4. 自前アプリのCI/CD設定]()
↓
[5. 挙動確認]()　\:tada:

### 1. ソースコード更新し手動デプロイ

1. ソースコードを更新
    ```bash
    # scaffoldを使ってCRUDが実行できるオブジェクトを追加
    bundle exec rails generate scaffold User username:string email:string

    # DBにも反映
    bundle exec rails db:migrate
    ```
    <br>
2. イメージをビルドしECRにプッシュ
    ```bash
    # イメージをビルドし、プッシュ先のレジストリ名, リポジトリ名でタグ付け
    # account_id = AWSのアカウントID, repository_name = ECRのリポジトリ名
    docker build --platform linux/x86_64 --tag <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com/<repository_name>:latest .

    # ECRのログインパスワードを取得し、ECRレジストリにそのパスワードでログイン
    aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com

    # ビルドしたイメージをECRにプッシュ
    docker push <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com/<repository_name>:latest
    ```
    <br>
3. CLIでデプロイ
    ```bash
    aws ecs update-service --cluster <cluster_name> \
    --service <service_name> --force-new-deployment
    ```

ソースコード更新後のデプロイの流れを把握できたので、自動化の設定をしていきます。

### 2. CI/CD設定

1. GitHub Secretsに環境変数を追加
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`
  - `ECR_REPOSITORY`
  - `ECS_CLUSTER`
  - `ECS_SERVICE`
  <br>
2. deploy.ymlを作成
