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
[1. ソースコード更新し手動デプロイ](#1-ソースコード更新し手動デプロイ)
↓
[2. CI/CD設定](#2-cicd設定)
↓
[3. 自前アプリの手動デプロイ](#3-自前アプリの手動デプロイ)
↓
[4. 自前アプリのCI/CD設定](#4-自前アプリのcicd設定)
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
  - `ECS_TASK_DEFINITION`
  - `RDS_CLUSTER_IDENTIFIER`
  <br>
2. deploy.ymlを作成

  手動で行ったデプロイの内容をdeploy.ymlに落とし込みます。
  内容は、mainブランチにプッシュ感知 → Dockerイメージをビルド → ECRにプッシュ → ECS Serviceのデプロイ、という感じです。

  <details><summary>deploy.yml</summary>

  ```yaml
  name: Deploy to ECS

  on:
    push:
      branches:
        - main

  jobs:
    deploy:
      name: Build and Deploy to ECS
      runs-on: ubuntu-latest

      steps:
        - name: Checkout source code
          uses: actions/checkout@v4

        - name: Configure AWS credentials
          uses: aws-actions/configure-aws-credentials@v2
          with:
            aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
            aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            aws-region: ap-northeast-1

        - name: Log in to ECR
          id: login-ecr
          uses: aws-actions/amazon-ecr-login@v1

        - name: Build, tag, and push image to ECR
          env:
            ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
            ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}
            IMAGE_TAG: ${{ github.sha }}
          run: |
            docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
                        -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
                        --platform linux/x86_64 .
            docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
            docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

        - name: Update ECS service
          env:
            CLUSTER: ${{ secrets.ECS_CLUSTER }}
            SERVICE: ${{ secrets.ECS_SERVICE }}
          run: |
              aws ecs update-service \
                --cluster $CLUSTER \
                --service $SERVICE \
                --force-new-deployment \

  ```
  </details>

### 3. 自前アプリの手動デプロイ

- Docker関連ファイルを編集
  - Dockerfile, Dockerfile.dev, docker-compose.yml, docker-entrypoint, production.rb, database.yml, ridgepole.rakeをサンプルのscaffoldアプリの通りに修正

- AWS Secrets Managerの`RAILS_MASTER_KEY`を自前アプリのものに修正

- 手動デプロイ
  ```bash
  # イメージビルド & タグ付け
  dc buildx build --platform linux/x86_64 --tag <account_id>.dkr.ecr.ap-northeast-1.amazonaws.com/leanup-prod-web:latest .

  # ECRの認証
  aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 627286173480.dkr.ecr.ap-northeast-1.amazonaws.com

  # ECRにイメージをプッシュ
  docker push 627286173480.dkr.ecr.ap-northeast-1.amazonaws.com/leanup-prod-web:latest

  # ECS Serviceでデプロイ
  aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --force-new-deployment

  # DBからscaffoldアプリのデータを削除
  aws ecs run-task \
  --cluster leanup-prod-viacdn-ecs-cluster \
  --task-definition leanup-prod-viacdn-ecs-task \
  --launch-type EC2 \
  --count 1 \
  --overrides '{
    "containerOverrides":
    [{"name": "web",
    "command": ["bin/rails", "db:purge"],
    "environment": [
      { "name": "DISABLE_DATABASE_ENVIRONMENT_CHECK", "value": "1" },
      { "name": "RAILS_ENV", "value": "production" }
    ]
    }]
  }'

  # 自前アプリのテーブル構造をDBに反映
  aws ecs run-task \
  --cluster leanup-prod-viacdn-ecs-cluster \
  --task-definition leanup-prod-viacdn-ecs-task \
  --launch-type EC2 \
  --count 1 \
  --overrides '{
    "containerOverrides": [{
      "name": "web", 
      "command": ["bundle", "exec", "rake", "ridgepole:apply"],
      "environment": [
        { "name": "RAILS_ENV", "value": "production" }
      ]
    }]
  }'
  ```

### 4. 自前アプリのCI/CD設定
- deploy.ymlをサンプルのscaffoldアプリの通りに修正
- S3に静的ファイルを保存できるようにする
  1. `aws-sdk` gemを追加
  2. AWS, ストレージの設定
     ```sh
     # config/initializers/aws.rb
     # ECS環境での実行時のみECS Task Roleから認証情報を生成 
     if ENV['AWS_CONTAINER_CREDENTIALS_RELATIVE_URI'].present?
      Aws.config.update({ credentials: Aws::ECSCredentials.new })
     end

     # config/storage.yml
     amazon:
      service: S3
      region: ap-northeast-1
      bucket: <S3_bucket_name>

     # config/environments/production.rb
     config.active_storage.service= :amazon
     ```
  3. デプロイして、本番環境で画像がS3に保存できるようになっているか、画像URLを確認




