---
title: 既存RailsアプリをECSに初めてデプロイ(CI/CD)したときの手順①【scaffoldアプリ作成 & Docker化】
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

はじめてのAWS ECS（EC2起動タイプ）でのデプロイ。
Herokuやfly.ioなどでの比較的手軽にできるデプロイとは違い、
１から作っていく感を楽しみつつ（もがきつつ）、その手順をまとめました。

初めてECSでデプロイをする際の参考になったらと思います…！
（※ ある程度機能を作り、Docker化済みのRailsアプリのデプロイです。）


手順としては、以下の通りです。

|     |                                        | 
| --- | -------------------------------------------- | 
| 1   | scaffoldアプリ作成＆Docker化 👈 この記事 | 
| 2   | [CFnでAWSリソース作成]()                     | 
| 3   | [CI/CDの設定]()                                  | 


この記事では、scaffoldアプリの作成＆Docker化について書いていきます。

流れは、
[1. scaffoldアプリを作る](#1-scaffoldアプリを作る)
↓
[2. scaffoldアプリをDocker化](#2-scaffoldアプリをdocker化)
↓
[3. 挙動確認](#3-挙動確認)　\:tada:

という順に行なっていきます。


## 1. scaffoldアプリを作る

RailsアプリをECSにデプロイするのが初めてだったので、テスト用に最小限の構成のRailsアプリをまず準備します。

Ruby version: 3.2.4, Rails version: 7.2.2.1 (自前) / 8.0.2 (scaffold)
（Railsのバージョンも自前アプリとscaffoldで合わせた方がより安心かと思います...）


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
      # 互換性エラーで失敗したので、`rbenv local 3.2.4`のように互換性のあるrubyバージョンに変更してからインストールし直し成功
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
    # -> 無事にブラウザでRailsアプリの表示ができていることを確認
    ```
    <br>
    
9.  scaffoldを使って簡単なCRUD実装
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
開発環境と本番環境の間で、環境変数や行う処理が異なるのでファイルを分けます。

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
:::note 
  DBテーブル構造の更新（`ridgepole:apply`）が行われるのは、開発環境ではdockerビルド時、本番環境ではデプロイ時、と分けています。本番環境ではAWS側の都合等でECS Taskが落ちてデプロイ時以外でビルドし直されることがあるので、予期せぬタイミングでDB構造の変更が起きないように分けています。
:::


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
本番環境については、環境変数を取得するように書いておきます。
（後ほどAWS Secrets ManagerにてDB接続に必要な値を生成します。）

<details><summary>database.yml</summary>

```yaml:config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  username: 'postgres'
  password: 'password'
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>


development:
  <<: *default
  database: lean_up_development

test:
  <<: *default
  database: lean_up_test

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
（ホストヘッダーの環境変数の値は、すでに購入してあるドメイン名を、後ほどCloudFormationのTask Definitionにて設定します。）
DNSリバインディング攻撃からの保護は開発環境ではデフォルトでON,本番環境ではデフォルトでOFFになっているらしい。（todo: ちゃんと理解して書く ❗）
今回は本番環境でもONにしました。
参考：[DNSリバインディング(DNS Rebinding)対策総まとめ](https://blog.tokumaru.org/2022/05/dns-rebinding-protection.html)

<details><summary>production.rb</summary>

```rb:config/environments/production.rb
  config.hosts << ENV["RAILS_CONFIG_HOSTS"] if ENV["RAILS_CONFIG_HOSTS"]
  config.host_authorization = { exclude: ->(request) { request.path == "/up" } }
```
</details>

## 3. 挙動確認

scaffoldアプリのDocker化が正しく行われていることを最後に確認します。
`docker-compose up`でコンテナを起動し、ブラウザで正常に表示されることを確認できたら、
[CloudFormationdでのAWSリソース作成]()に入ります。
