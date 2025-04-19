---
title: 既存RailsアプリをECSに初めてデプロイ(CI/CD)したときの手順
tags:
  - 'Rails'
  - 'ECS'
  - 'CI/CD'
  - 'GitHub Actions'
  - 'CloudFormation'
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

はじめてのAWS ECSでのデプロイ。
Herokuやfly.ioなどでの比較的手軽にできるデプロイとは違い、
１から作っていく感を楽しみつつ（もがきつつ）、その手順をまとめました。
初めてECSでデプロイをする際の参考になったらと思います…！
（※ ある程度機能を作り、Docker化済みのRailsアプリのデプロイです。）


手順としては、以下の通りです。
1. [scaffoldアプリを作る](#1-scaffoldアプリを作る)
2. [scaffoldアプリをDocker化](#2-scaffoldアプリをdocker化)
3. 必要なAWSリソースを作る (CloudFormation)
4. 手動デプロイしてみる
5. CI/CDの設定
6. 自前のRailsアプリをデプロイ (CI/CD) :tada:


Ruby version: 3.2.4, Rails version: 7.2.2.1 (scaffold) / 8.0.2 (自前)

## 1. scaffoldアプリを作る

RailsアプリをECSにデプロイするのが初めてだったので、最小限の構成のRailsアプリをまず準備します。


1. scaffoldアプリ用のディレクトリを作成し、そのディレクトリに入る
    ```sh
      mkdir ecs_deploy_sample && cd $_
    ```
    <br>
2. Gemfile作成（← bundlerはこれを元にgemのインストール, バージョン管理を行う）
    ```sh
      bundle init
    ```
    <br>
3. Gemfileにて、`gem ‘rails’`のコメントアウトを解除
   <br>
4. gemのインストール先をシステム全体ではなくプロジェクト内に設定（CI/CDで有効な設定）
    ```sh
      bundle config set path ‘vendor/bundle’
    ```
    <br>    
5. Gemfileをもとに必要なgemを一括インストール
    
    ```sh
      bundle install
      # `rbenv local 3.2.4`のように互換性のあるrubyバージョンに変更してからのインストールでないとエラーで失敗した
    ```
    <br>
6. Railsアプリ新規作成

    ```sh
      bundle exec rails new . --javascript importmap --database=postgresql
      # 自前のアプリがpostgresqlだったためpostgresqlを指定
    ```
    <br>
7. DB新規作成
    ```sh
    bundle exec rails db:create
    ```
    <br>
8. サーバ起動し、ブラウザで疎通確認
    ```sh
    bundle exec rails server
    # 無事にブラウザでRailsアプリの表示ができていることを確認
    ```
    <br>
    
1.  scaffoldを使って簡単なCRUD実装
    ```sh
    bundle exec rails generate scaffold Post name:string title:string content:text
    # ↓
    bundle exec rails db:migrate
    ```
  

## 2. scaffoldアプリをDocker化

scaffoldを利用して作成した、最小限の構成のRailsアプリをDocker化します。

編集・作成するファイル：
- Dockerfile（開発環境と本番環境の２つ）
- docker-entrypoint
- docker-compose.yml
- database.yml
- production.rb

上記のファイルはこちらのリポジトリ↓にあります。

https://github.com/sarii0213/lean_up/tree/main

### Dockerfile（開発用と本番用）
開発環境と本番環境でイメージの中身（環境変数や行う処理）が異なるので、ファイルを分けます。

<details><summary>Dockerfile.dev (開発環境用)</summary>

  ```:Dockerfile.dev
  ARG RUBY_VERSION=3.2.4
  FROM ruby:$RUBY_VERSION-slim

  ENV TZ=Asia/Tokyo \
      RAILS_ENV=development \
      BUNDLE_PATH="/usr/local/bundle" \
      BUNDLE_JOBS=4 \
      BUNDLE_RETRY=3 \
      BUNDLE_WITHOUT=""

  WORKDIR /rails

  RUN apt-get update -qq && \
      apt-get install -y \
        build-essential \
        libpq-dev \
        nodejs \
        vim \
        postgresql-client \
        libvips && \
      rm -rf /var/lib/apt/lists/* # remove apt cache

  COPY Gemfile Gemfile.lock ./
  RUN bundle install

  COPY . .

  ENTRYPOINT ["/rails/bin/docker-entrypoint"]

  EXPOSE 3000

  CMD ["./bin/rails", "server", "-b", "0.0.0.0", "-p", "3000"]

  ```
</details>

<details><summary>Dockerfile (本番環境用, 中身はデフォルトのもの＋環境変数を追加)</summary>

  ```:Dockerfile
  ARG RUBY_VERSION=3.2.4
  FROM docker.io/library/ruby:$RUBY_VERSION-slim AS base

  # Rails app lives here
  WORKDIR /rails

  # Install base packages
  RUN apt-get update -qq && \
      apt-get install --no-install-recommends -y curl libjemalloc2 libvips postgresql-client && \
      rm -rf /var/lib/apt/lists /var/cache/apt/archives

  # Set production environment
  ENV RAILS_ENV="production" \
      BUNDLE_DEPLOYMENT="1" \
      BUNDLE_PATH="/usr/local/bundle" \
      BUNDLE_WITHOUT="development:test"

  # Throw-away build stage to reduce size of final image
  FROM base AS build

  # Install packages needed to build gems
  RUN apt-get update -qq && \
      apt-get install --no-install-recommends -y build-essential git libpq-dev libyaml-dev pkg-config && \
      rm -rf /var/lib/apt/lists /var/cache/apt/archives

  # Install application gems
  COPY Gemfile Gemfile.lock ./
  RUN bundle install && \
      rm -rf ~/.bundle/ "${BUNDLE_PATH}"/ruby/*/cache "${BUNDLE_PATH}"/ruby/*/bundler/gems/*/.git && \
      bundle exec bootsnap precompile --gemfile

  # Copy application code
  COPY . .

  # Precompile bootsnap code for faster boot times
  RUN bundle exec bootsnap precompile app/ lib/

  # Precompiling assets for production without requiring secret RAILS_MASTER_KEY
  RUN SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile




  # Final stage for app image
  FROM base

  # Copy built artifacts: gems, application
  COPY --from=build "${BUNDLE_PATH}" "${BUNDLE_PATH}"
  COPY --from=build /rails /rails

  # Run and own only the runtime files as a non-root user for security
  RUN groupadd --system --gid 1000 rails && \
      useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash && \
      chown -R rails:rails db log storage tmp
  USER 1000:1000

  # Entrypoint prepares the database.
  ENTRYPOINT ["/rails/bin/docker-entrypoint"]

  # Start server via Thruster by default, this can be overwritten at runtime
  EXPOSE 3000
  CMD ["./bin/rails", "server"]
  ```
</details>


### docker-entrypoint
コンテナ起動時に実行されるファイル。
ファイル内で、環境毎に実行する処理を分けています。

<details><summary>bin/docker-entrypoint</summary>

  ```sh:docker-entrypoint
  #!/bin/bash -e

  # Enable jemalloc for reduced memory usage and latency.
  if [ -z "${LD_PRELOAD+x}" ]; then
      LD_PRELOAD=$(find /usr/lib -name libjemalloc.so.2 -print -quit)
      export LD_PRELOAD
  fi

  # Remove any pre-existing server.pid for Rails
  rm -f tmp/pids/server.pid

  # If running the rails server then create or migrate existing database
  if [ "${1}" == "./bin/rails" ] && [ "${2}" == "server" ]; then
    ./bin/rails db:prepare

    if [ "$RAILS_ENV" == "development" ]; then
      bundle exec rake ridgepole:apply
    fi
  fi

  exec "${@}"
  ```
</details>

### docker-compose.yml
開発環境で起動するコンテナの定義をまとめたもの。
本番環境では、docker-compose.ymlは使われず、AWSのTask Definitionを使ってコンテナの定義を行います。

<details><summary>docker-compose.yml</summary>

```yaml:docker-compose.yml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
      args:
        RUBY_VERSION: 3.2.4
    volumes:
      - .:/rails:cached
      - bundle:/usr/local/bundle
    ports:
      - '3000:3000'
    depends_on:
      - db
      - chrome
      - redis
      - sidekiq
    stdin_open: true
    tty: true
    restart: always
    environment:
      RAILS_ENV: development
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      SELENIUM_DRIVER_URL: http://chrome:4444/wd/hub
      REDIS_URL: redis://redis:6379

  db:
    image: postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - '3306:3306'
    volumes:
      - postgres_volume:/var/lib/postgresql/data
    restart: always

  chrome:
    image: selenium/standalone-chrome-debug:latest
    ports:
      - '4448:4444'

  sidekiq:
    build:
      context: .
      dockerfile: Dockerfile.dev
    environment:
      REDIS_URL: redis://redis:6379
    volumes:
      - .:/lean_up:cached
      - bundle:/usr/local/bundle
    depends_on:
      - db
      - redis
    command: bundle exec sidekiq

  redis:
    image: redis:latest
    ports:
      - 6379:6379
    volumes:
      - redis:/data

volumes:
  bundle:
  postgres_volume:
  redis:

```
</details>

### database.yml
開発環境と本番環境でのPostgresqlの接続設定をします。
本番環境については、ひとまず環境変数を取得するようにだけしておきます。
（後ほどAWS Secrets ManagerにてDB接続に必要な値を生成します。）

<details><summary>database.yml</summary>

```yaml:config/database.yml
# PostgreSQL. Versions 9.3 and up are supported.
#
# Install the pg driver:
#   gem install pg
# On macOS with Homebrew:
#   gem install pg -- --with-pg-config=/usr/local/bin/pg_config
# On Windows:
#   gem install pg
#       Choose the win32 build.
#       Install PostgreSQL and put its /bin directory on your path.
#
# Configure Using Gemfile
# gem "pg"
#
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  username: 'postgres'
  password: 'password'
  # For details on connection pooling, see Rails configuration guide
  # https://guides.rubyonrails.org/configuring.html#database-pooling
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>


development:
  <<: *default
  database: lean_up_development

  # The specified database role being used to connect to PostgreSQL.
  # To create additional roles in PostgreSQL see `$ createuser --help`.
  # When left blank, PostgreSQL will use the default role. This is
  # the same name as the operating system user running Rails.
  #username: lean_up

  # The password associated with the PostgreSQL role (username).
  #password:

  # Connect on a TCP socket. Omitted by default since the client uses a
  # domain socket that doesn't need configuration. Windows does not have
  # domain sockets, so uncomment these lines.
  #host: localhost

  # The TCP port the server listens on. Defaults to 5432.
  # If your server runs on a different port number, change accordingly.
  #port: 5432

  # Schema search path. The server defaults to $user,public
  #schema_search_path: myapp,sharedapp,public

  # Minimum log levels, in increasing order:
  #   debug5, debug4, debug3, debug2, debug1,
  #   log, notice, warning, error, fatal, and panic
  # Defaults to warning.
  #min_messages: notice

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test:
  <<: *default
  database: lean_up_test

# As with config/credentials.yml, you never want to store sensitive information,
# like your database password, in your source code. If your source code is
# ever seen by anyone, they now have access to your database.
#
# Instead, provide the password or a full connection URL as an environment
# variable when you boot the app. For example:
#
#   DATABASE_URL="postgres://myuser:mypass@localhost/somedatabase"
#
# If the connection URL is provided in the special DATABASE_URL environment
# variable, Rails will automatically merge its configuration values on top of
# the values provided in this file. Alternatively, you can specify a connection
# URL environment variable explicitly:
#
#   production:
#     url: <%= ENV["MY_APP_DATABASE_URL"] %>
#
# Read https://guides.rubyonrails.org/configuring.html#configuring-a-database
# for a full overview on how database connection configuration can be specified.
#
production:
  <<: *default
  host: <%= ENV.fetch("DB_HOST", 'host') %>
  database: <%= ENV.fetch("DB_NAME", 'database') %>
  username: <%= ENV.fetch("DB_USERNAME", 'username') %>
  password: <%= ENV.fetch("DB_PASSWORD", 'password') %>

```
</details>

### production.rb
本番環境で接続を許可するホストヘッダーとパス（ヘルスチェック用）の設定をします。
（ホストヘッダーの値は後ほどTask Definitionにて、用意済みのドメイン名を入れます。）
DNSリバインディング攻撃からの保護は開発環境ではデフォルトでON,本番環境ではデフォルトでOFFになっているらしい。
今回は本番環境でもONにしました。
参考：[DNSリバインディング(DNS Rebinding)対策総まとめ](https://blog.tokumaru.org/2022/05/dns-rebinding-protection.html)

<details><summary>production.rb</summary>

```rb:config/environments/production.rb
...
  config.hosts << ENV["RAILS_CONFIG_HOSTS"] if ENV["RAILS_CONFIG_HOSTS"]

  config.host_authorization = { exclude: ->(request) { request.path == "/up" } }
...
```
</details>


## 3. 必要なAWSリソースを作る

CloudFormation(略してCFn)でデプロイ先のAWSリソースを作っていきます。

流れとしては、作るもの①を作り終わったら、Imageのプッシュの作業をしたりなどして、作るもの②を作ります。
今回作成したCloudFormationの全YAMLファイルはこちら↓にあります。
（作るもの①②にて列挙している順番に、YAMLファイルひとつひとつを S3へアップロード->CloudFormationでcreate stackで作っていきます。）

https://github.com/sarii0213/lean_up/tree/main/deploy


### 構成図
todo: 自作する

### CFnで作るもの① (概略と注意点など)
- Route53 (Value Domainにて取得済みのドメイン名を登録。Route53にて生成されるNSレコードをValue Domain側に入力し、名前解決の際にドメイン名の権威サーバがAWSの権威サーバのIPを返すようにする)
- Certificate Manager (https通信を可能にするためのもの。SSL/TLS証明書を発行・管理する)
- VPC（ネットワークの外枠）
- IAM (ECS Execの権限を作成。ECS Execは、ECS Taskの外側からTask内での処理を命令できる機。 ※ AWSを操作するためのIAM Userは事前に作成済み)
- RDS（データベース。ECS Task内でもDB用のコンテナ作ることも可能だが、AWSに管理を丸投げできたりするのでRDSを使う。）
- Secrets Manager（CloudFrontで使うカスタムヘッダの値＆DBの認証情報を保管する。`RAILS_MASTER_KEY`も保存する。<- config/master.keyの値をコピペして保存。）
- ELB（トラフィックをパス毎に振り分けをする。health checkもしている。http->httpsのリダイレクトも。）
- CloudFront（キャッシュ配信とエッジロケーションでレスポンスが高速化。静的ファイルはS3へ、それ以外はELBへ、とトラフィックの振り分けも行う）
- S3（アプリの静的ファイル用とログ用のファイル置き場）
- ECR（Docker Imageを保管する場所。ローカルからECRにimageをプッシュして、それをECS Task Definitionにて参照する。）

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

### CFnで作るもの① (概略と注意点など)
- launch template
- ec2 auto scaling
- ecs (cluster, task definition, service) 
