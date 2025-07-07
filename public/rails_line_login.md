---
title: RailsでLINE ログイン/配信 機能を実装 (devise + custom OmniAuth strategy)
tags:
  - 'Rails'
  - 'devise'
  - 'OAuth2'
  - 'OIDC'
  - 'LINE'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

Ruby on RailsにLINE ログイン/配信 機能を実装する手順をまとめました。

LINEログインをRailsアプリに組み込む方法として、少し前の記事だと`omniauth-line`gemを使っているものが多いですが、GitHubを見る限りしばらくメンテナンスされていないgemなので、カスタムでOmniAuth strategyを作る方法を取りました。

:::note info 
Ruby, gemのバージョン：
  - Ruby: 3.4.3
  - rails gem: 8.0.2
  - devise gem: 4.9.4
  - omniauth gem: 2.1.3 (deviseでrequire)
  - omniauth-oauth2 gem: 1.8.0
:::

実装手順（[LINEログイン機能 ↓](#lineログイン機能実装の手順), [LINE配信機能 ↓](#line配信機能実装の手順)）の説明をする前に、Webアプリケーションから外部Webサービス（今回だとLINE）での認可・認証について、整理します。

## 認証・認可の基本情報
（OAuth, OIDC, access token, ID token, request phase, callback phase, OmniAuth, strategy, provider）
### OAuth
- Webサービスにおいて、リソースへのアクセス許可を安全に委譲する認可の仕組み。
- **OAuth 2.0**ではクライアントアプリ（今回だとRailsアプリ）が、認可サーバ（今回だとLINE）に対してaccess tokenを要求し、認可サーバはユーザーの許可を得てクライアントアプリにaccess tokenを発行する。
- ちなみに... OAuth 1.0は認証フローが複雑＆対応アプリに制限あり＆すべてのAPIリクエストで署名必須だった。OAuth 2.0は、OAuth 1.0の問題点を解決し、より柔軟で使いやすい認証・認可フレームワークを提供している。
- 参考：
  - [徹底解説：OAuth 1.0とOAuth 2.0の違い](https://apidog.com/jp/blog/oauth-1-2-difference/)
  - [一番分かりやすい OAuth の説明](https://qiita.com/TakahikoKawasaki/items/e37caf50776e00e733be)

### OIDC (OpenID Connect)
- 異なるWebサービス間における認証の仕組み。OAuth 2.0の拡張仕様。
- クライアントアプリがOpenIDプロバイダー（今回だとLINE）にID tokenを要求し、ユーザーの認証＆許可取得後に、OpenIDプロバイダーがクライアントアプリにID tokenを発行する。
- access token v.s. ID token
  - **access token**：OAuth 2.0で定義された、リソースへのアクセスを認可するためのトークン
    - LINEの場合：「ユーザーの許可もらってるから、このWebアプリに対してこのLINEユーザのIDとメールアドレスを渡せるよ」
  - **ID token**：OIDCで定義された、ユーザが認証されたことを証明するトークン
    - LINEの場合：「このユーザーはLINEログイン成功済み。このユーザーのIDとメールアドレス情報をtokenに練りこんであるよ」
  - LINEの場合の参考：[LINEログイン v2.1 APIリファレンス](https://developers.line.biz/ja/reference/line-login/)

### LINEログインのフロー（OAuth 2.0 + OIDC） [🔗 フロー図](https://developers.line.biz/ja/docs/line-login/integrate-line-login/#login-flow)
1. ユーザーが「LINEログイン」ボタン押下<br>（= GET `users/auth/line` → `<OmniAuth strategy>#request_phase`）
2. WebアプリがLINE認可サーバにアクセスし、LINEログイン画面にリダイレクト
3. ユーザーがログイン画面で同意（=認証＆認可）すると、LINE認可サーバはWebアプリに認可コードを発行<br>(= redirect to `users/auth/line/callback`)
4. WebアプリはLINE認可サーバに`access token`を発行要求（受け取った認可コードを添付）
5. LINE認可サーバは認可コードを検証 → `access token`を発行（`ID token`, `scope`を添付。`scope`は2でWebアプリから認可サーバに送ったパラメータのひとつでアクセス権限を定義）
6. WebアプリはLINE認可サーバにユーザー情報を要求（`ID token`を添付）
7. LINE認可サーバは`ID token`を検証 → LINE ID, LINEユーザー名などをWebアプリに返す(=`scope`で定義した情報)
8. Webアプリにて、返されたユーザー情報を元にユーザー新規作成やログイン処理を行う
（1~2: **request phase**, 3~7: **callback phase**） 

### OmniAuthにおけるstrategy, provider
- **OmniAuth**: "multi-provider Authentication"。多様な認証フローを標準化したライブラリ。
- **provider**: どの認証フロー(= **strategy**)を使うかの設定 ← by Rackミドルウェアにstrategy登録
- **strategy**: Railsアプリ本体(routes, controllers, views, models etc.)と認可サーバとの間の窓口係として、request / callback phaseにて二者の間に立って処理をする。
   - （request phase 例） 上記フローの1~2にて、LINEログインボタン押下時に、line strategyがリクエストのパラメータなどを取りまとめてLINE認可サーバへ投げる。
   - （callback phase 例） 上記フローの3にて、LINE認可サーバから発行された認可コードは、まずline strategyが受け取り、access token ＆ ID token のやりとりを経て取得したLINEユーザ情報をセットし、controller（`omniauth_callbacks#line`）へ処理が引き継がれる
- 参考：[OmniAuth GitHub](https://github.com/omniauth/omniauth)

<br>
<br>
Railsアプリと他Webサービスの間での認証・認可に使われる仕組みや必要な機構を把握できたところで、LINEログイン機能の実装をしていきます！

## LINEログイン機能実装の手順
1. **LINEログインチャネルの作成** [🔗 公式doc](https://developers.line.biz/ja/docs/line-login/integrate-line-login/#create-a-channel)
   1. チャネル（＝WebアプリとLINEプラットフォームを接続する通信路）をLINE Developersコンソールにて作成
   2. チャネルにて、メールアドレスの取得権限を申請<br>（LINEのメールアドレス情報を使いたい場合のみ）
   <br>
2. **gemのインストール**
   ```rb:Gemfile
   # devise gemはインストール済み
   gem 'omniauth-oauth2'
   gem 'omniauth-rails_csrf_protection'
   gem 'dotenv-rails' # 環境変数の管理 (LINE_CHANNEL_ID, SECRET etc.)
   ```
   <details><summary>:bulb: OmniAuth関連gemの役割</summary>

   - `omniauth`: request phase, callback phaseなど認証・認可の骨組み。`devise`が`omniauth`をロードしているので今回インストール不要
   - `oauth2`: OAuth 2.0の基本処理（リクエスト生成・アクセストークン取得・認可フロー etc.）の実装をサポート。`omniauth-oauth2`で読み込まれている。
   - `omniauth-oauth2`: OmniAuthのproviderとして使えるOAuth 2.0 Strategyのベースを提供
   - `omniauth-rails_csrf_protection`: OmniAuthのセキュリティ強化
   </details>
   <br>
3. **line strategyを作成**
   <details><summary>line strategyのコード</summary>

   ```rb:lib/strategies/line.rb
   require 'omniauth-oauth2'

    module Strategies
      class Line < OmniAuth::Strategies::OAuth2

        # request phase -----------------------------------------------------
        
        # IDトークン, プロフィール情報, メールアドレスの取得権限を含める
        option :scope, 'openid profile email'

        # optionを渡す先
        option :client_options, {
          site: 'https://api.line.me',
          authorize_url: 'https://access.line.me/oauth2/v2.1/authorize',
          token_url: 'https://api.line.me/oauth2/v2.1/token'
        }

        # callback phase ---------------------------------------------------

        # 取得したデータ(LINEユーザーID)からuid(=unique to the provider)をセット
        uid { raw_info['sub'] }

        # 取得したデータからinfo(= a hash of information about the user)をセット
        info do
          {
            name: raw_info['name'],
            email: raw_info['email']
          }
        end

        def raw_info
          @raw_info ||= verify_id_token
        end

        private

        # ID Tokenに必須のnonceをパラメータに追加
        def authorize_params
          super.tap do |params|
            params[:nonce] = SecureRandom.uuid
            session['omniauth.nonce'] = params[:nonce]
          end
        end

        # omniauthのcallback_urlはquery stringがついてしまい、LINE側に登録したcallback URLとの不一致エラーになるそうなのでoverride
        # 参考： https://zenn.dev/hid3/articles/40ab3d1060f013#%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF%E3%83%95%E3%82%A7%E3%83%BC%E3%82%BA
        # callback_url: https://github.com/omniauth/omniauth/blob/0bcfd5b25bf946422cd4d9c40c4f514121ac04d6/lib/omniauth/strategy.rb#L498
        def callback_url
          full_host + callback_path
        end

        # ID token 検証 & ユーザ情報取得のAPIリクエスト
        def verify_id_token
          @id_token_payload ||= begin
            client.request(:post, 'https://api.line.me/oauth2/v2.1/verify', 
              {
                body: {
                  id_token: access_token['id_token'],
                  client_id: options.client_id,
                  nonce: session.delete('omniauth.nonce')
                }
              }
            ).parsed
          rescue => e
            Rails.error.report(e, context: { 
              action: '[LINE login] ID token verification & get user info',
              client_id: options.client_id,
              has_id_token: access_token['id_token'].present?
            })
            raise
          end
        
          @id_token_payload
        end
      end
    end
   ```
   </details>
   <br>
4. **line strategyをdeviseのRackミドルウェアとして組み込む**
   1. 環境変数の設定
      dev環境では.envに`LINE_CHANNEL_ID`, `LINE_CHANNEL_SECRET`を追加
   2. DBにOmniAuthで必要なカラムを追加
      usersテーブルに`provider`カラム(string), `uid`カラム(string)を追加
   3. deviseのinitializer & user model にline strategyを登録
      ```rb
      # devise.rb (OmniAuthミドルウェアとしてline strategyを登録)
      require 'strategies/line'
      ...
      config.omniauth :line, ENV['LINE_CHANNEL_ID'], ENV['LINE_CHANNEL_SECRET']


      # user.rb (DeviseにLINEログインを組み込む宣言)
      devise :omniauthable, omniauth_providers: [:line]
      # -> user_line_omniauth_authorized_path, user_line_omniauth_callback_pathが自動生成

      validates :uid, uniqueness: { scope: :provider}, if: -> { provider.present? }
      ```
    <br>
5. **ルーティング ＆ callbacks controller の作成**
    ```rb:routes.rb
        devise_for :users, controllers: {
          # /users/auth/line/callback -> users/omniauth_callbacks#line
          omniauth_callbacks: 'users/omniauth_callbacks'
        } 
    ```

   <details><summary>callbacks controllerのコード（LINEサーバからユーザ情報取得後の挙動）</summary>

    ```rb:controllers/users/omniauth_callbacks_controller.rb
    module Users
      class OmniauthCallbacksController < Devise::OmniauthCallbacksController
        skip_before_action :verify_authenticity_token, only: :line

        def line
          @user = User.from_omniauth(request.env['omniauth.auth'], current_user)

          notify_line_already_linked and return if current_user && @user.nil?

          if @user.persisted?
            complete_line_login
          else
            fail_line_login
          end
        end

        private

        def notify_line_already_linked
          redirect_to user_setting_path
          set_flash_message(:alert, :failure, kind: 'LINE', reason: '他アカウントでLINE連携済みです')
        end

        def complete_line_login
          sign_in_and_redirect @user, event: :authentication
          set_flash_message(:notice, :success, kind: 'LINE')
        end

        def fail_line_login
          session['devise.line_data'] = request.env['omniauth.auth'].except(:extra)
          redirect_to new_user_registration_url
          set_flash_message(:alert, :failure, kind: 'LINE', reason: 'LINE連携に失敗しました')
        end
      end
    end
    ```
   </details>
   
    <details><summary>User.from_omniauthのコード（LINEユーザ情報からLINE連携/ログイン/ユーザー作成）</summary>

    ```rb:models/user.rb
    def self.from_omniauth(auth, current_user = nil)
      return link_line_account(auth, current_user) if current_user&.line_connected? == false

      sign_in_or_create_user_from_line(auth)
    end

    def self.link_line_account(auth, current_user)
      success = current_user.update(
        provider: auth.provider,
        uid: auth.uid,
        email: auth.info.email,
        line_notify: true
      )

      success ? current_user : nil
    end

    def line_connected?
      uid.present? && provider.present?
    end

    def self.sign_in_or_create_user_from_line(auth)
      # LINE連携済みのuserのusername, passwordは更新されない
      find_or_create_by(
        provider: auth.provider,
        uid: auth.uid,
        email: auth.info.email
      ) do |user|
        user.username = auth.info.name
        user.password = Devise.friendly_token[0, 20]
        user.line_notify = true
      end
    end
    ```
    </details>
   
   :::note warn
   LINEコンソール > LINEログインチャネルにて、**callback URL**の設定も必要
   (callback URL = [上記のLINEログインのフロー](#lineログインのフローoauth-20--oidc--フロー図)の3にて、ユーザの認証＆認可後に認可コードを受け取るWebアプリのURL `<domain name>/users/auth/line/callback`)
   :bulb: dev環境では[ngrok](https://ngrok.com/docs/getting-started/)を使って開発中アプリを公開しているので、ngrokから発行されたドメイン名を含んだcallback URLを登録する。
   :::
   <br>

6. **LINEログイン機能のfeature specを作成**
  
    <details><summary>LINEログイン feature specのコード</summary>

    ```rb:spec/feature/line_login_spec.rb
    require 'rails_helper'

    RSpec.describe 'LINEログイン機能', type: :feature do
      let(:line_uid) { '1234567890' }
      let(:line_email) { 'line_user@example.com' }
      let(:line_name) { 'line_user' }

      before do
        # /auth/line -> /auth/line/callback への即時リダイレクト設定
        OmniAuth.config.test_mode = true
        # /auth/line/callback へのリダイレクト時に渡されるデータ
        OmniAuth.config.mock_auth[:line] = 
          OmniAuth::AuthHash.new({
                                  provider: 'line',
                                  uid: line_uid,
                                  info: {
                                    name: line_name,
                                    email: line_email
                                  },
                                  credentials: {
                                    token: '1234qwerty'
                                  }
                                })

        Rails.application.env_config['devise.mapping'] = Devise.mappings[:user]
        Rails.application.env_config['omniauth.auth'] = OmniAuth.config.mock_auth[:line]
      end

      after do
        OmniAuth.config.mock_auth[:line] = nil
      end

      context '既存ユーザーがLINE未連携でログイン中の場合' do
        let!(:user) { create(:user, provider: nil, uid: nil) }

        before do
          login_as user
          visit user_setting_path
          click_button 'LINEと連携する'
        end

        it 'LINE連携時に、LINEに登録されたemailに更新され、LINE配信も許可に設定される' do
          user.reload
          expect(user.provider).to eq('line')
          expect(user.uid).to eq(line_uid)
          expect(user.email).to eq(line_email)
          expect(user.line_notify).to be(true)
        end
      end

      context '未サインアップでLINEログインにてアカウント作成する場合' do
        before do
          visit signup_path
          click_button 'LINEでログイン'
        end

        let(:created_user) { User.last }

        it 'LINE情報でアカウントが作成され、LINE配信が許可される' do
          expect(created_user.uid).to eq(line_uid)
          expect(created_user.provider).to eq('line')
          expect(created_user.email).to eq(line_email)
          expect(created_user.username).to eq(line_name)
          expect(created_user.line_notify).to be(true)
        end
      end

      context 'LINE連携済みのユーザーがLINEログインする場合' do
        let!(:user) do
          create(:user, provider: 'line', uid: line_uid, email: line_email, username: 'test_user', password: 'password')
        end

        before do
          visit login_path
          click_button 'LINEでログイン'
        end

        it 'LINEログイン時にusername, passwordはLINEのユーザ情報で上書きされない' do
          user.reload
          expect(user.username).to eq('test_user')
          expect(user.valid_password?('password')).to be(true)
        end
      end

      context 'すでに他ユーザーでLINE連携済みのLINEアカウントに対してLINE連携を試みた場合' do
        let(:user) { create(:user, provider: nil, uid: nil) }

        before do
          create(:user, provider: 'line', uid: line_uid, email: line_email)

          login_as user
          visit user_setting_path
          click_button 'LINEと連携する'
        end

        it 'LINE連携に失敗する' do
          expect(current_path).to eq(user_setting_path)
          expect(page).to have_content('他アカウントでLINE連携済みです')
        end
      end
    end

    ```
    </details>

    <br>
    ⇨ LINEログイン機能の実装完了 :tada:

## LINE配信機能実装の手順
1. **MessagingAPIチャネルを作成**
  LINEコンソールにてLINEログイン用チャネルと同じプロバイダ内に、MessaginAPI用のチャネルを作成
    <br>
1. **gemのインストール**
    ```rb:Gemfile
    gem 'line-bot-api'
    ```
    <br>
1. **LINE配信のジョブを作成**
   1. 環境変数を設定
      MessaginAPI用チャネルのアクセストークン`LINE_BOT_CHANNEL_ACCESS_TOKEN`とWebアプリのホスト名である`APP_HOST`を.envに保存
      ※ Webアプリのホスト名は、画像を配信する場合のURL生成に必要なため追加（`Rails.application.routes.default_url_options[:host]`）<br>
   2. usersテーブルに`line_notify`カラムを追加（LINE配信許可の設定値）
   （加えて、ユーザー設定画面にLINE配信許可の設定欄を追加し、コントローラでもparamsに`line_notify`追加）
   3. LINE配信のジョブを作成
      <details><summary>push_line_jobのコード</summary>

      ```rb:app/jobs/push_line_job.rb
      require 'line/bot'

      class PushLineJob < ApplicationJob
        queue_as :default

        def perform(*_args)
          users = User.where.not(uid: nil).where(line_notify: true).includes(:objectives)
          users.each do |user|
            # LINE配信許可がONのユーザーに対して、登録されたコンテンツの中からランダムに選択してメッセージ配信 
            objective = user.objectives.sample
            next if objective.blank?

            message = build_message(objective)
            request = Line::Bot::V2::MessagingApi::PushMessageRequest.new(to: user.uid, messages: [message])
            begin
              client.push_message(push_message_request: request)
            rescue StandardError => e
              # エラー通知処理
            end
          end
        end

        private

        def build_message(objective)
          # メッセージオブジェクトを生成
          # 画像メッセージ -> Line::Bot::V2::MessagingApi::ImageMessage.new(...)
          # テキストメッセージ -> Line::Bot::V2::MessagingApi::TextMessage.new(...)
        end

        def client
          Line::Bot::V2::MessagingApi::ApiClient.new(
            channel_access_token: ENV.fetch('LINE_BOT_CHANNEL_ACCESS_TOKEN', nil)
          )
        end
      end
      ```       
      </details>
    <br>
2. **SolidQueueを導入**
     1. SolidQueue関連テーブルを作成
         `bin/rails solid_queue:install`で生成されるqueue_schema.rbを元にDB更新
     2. SolidQueueの設定
        - `config/environments/*.rb`にて`config.active_job.queue_adapter = :solid_queue`
        - アプリのデータとSolidQueueのデータを同一のDBに相乗りさせたいので、`config/environments/*.rb`の`config.solid_queue.connects_to`削除
     3. SolidQueueの起動を設定
           <details><summary>開発環境ではpumaで起動する設定</summary>

           ```rb:config/puma.rb
           plugin :solid_queue if ENV["SOLID_QUEUE_IN_PUMA"] || Rails.env.development?
           ```

          </details>

         - 本番環境(AWS)ではコンテナ化して常時起動
           - Task Definitionにて、Railsアプリのコンテナと同一のTaskにSolid Queue用のコンテナを追加
            <br>
     4. LINE配信ジョブの定期実行を設定
        <details><summary>定期実行設定 YAMLのコード</summary>

        ```yaml:config/recurring.yml
        # 毎日19:45にLINE配信ジョブ実行（開発環境）
        development:
          push_line_job:
            class: PushLineJob
            args: []
            schedule: 45 19 * * * Asia/Tokyo

        # 毎日08:00にLINE配信ジョブ実行（本番環境）
        production:
          push_line_job:
            class: PushLineJob
            args: []
            schedule: 0 8 * * * Asia/Tokyo
        ```

        </details>
      
      <br>
3. **LINE配信機能のfeature specを作成**

    <details><summary>LINE配信 feature specのコード</summary>

      ```rb:spec/feature/push_line_job_spec.rb
      require 'rails_helper'

      RSpec.describe PushLineJob, type: :job do
        let(:user_without_line) { create(:user, provider: nil, uid: nil, line_notify: false) }
        let(:user_with_line_notify_on) { create(:user, provider: 'line', uid: '1234567890', line_notify: true) }
        let(:user_with_line_notify_off) { create(:user, provider: 'line', uid: '1234567891', line_notify: false) }

        let(:mock_client) { instance_double(Line::Bot::V2::MessagingApi::ApiClient) }

        before do
          allow(Line::Bot::V2::MessagingApi::ApiClient).to receive(:new).and_return(mock_client)
          allow(mock_client).to receive(:push_message).and_return(true)
        end

        context 'LINE未連携のユーザの場合' do
          let!(:user) { user_without_line }

          before do
            create(:objective, :image, user:)
            create(:objective, :verbal, user:)
          end

          it 'ビジョンボードの内容は配信されない' do
            described_class.perform_now
            expect(mock_client).not_to have_received(:push_message)
          end
        end

        context 'LINE連携済みだが通知許可がOFFの場合' do
          let!(:user) { user_with_line_notify_off }

          before do
            create(:objective, :image, user:)
            create(:objective, :verbal, user:)
          end

          it 'ビジョンボードの内容は配信されない' do
            described_class.perform_now
            expect(mock_client).not_to have_received(:push_message)
          end
        end

        context 'LINE連携済みで通知許可がONの場合' do
          let!(:user) { user_with_line_notify_on }

          before do
            create(:objective, :image, user:)
            create(:objective, :verbal, user:)
          end

          it 'ビジョンボードの内容が1件だけ配信される' do
            described_class.perform_now
            expect(mock_client).to have_received(:push_message).with(
              push_message_request: have_attributes(
                to: user.uid,
                messages: satisfy do |messages|
                  messages.all? do |m|
                    m.is_a?(Line::Bot::V2::MessagingApi::TextMessage) || m.is_a?(Line::Bot::V2::MessagingApi::ImageMessage)
                  end
                end
              )
            )
          end
        end
      end
      ```
    </details>
    <br>
    ⇨ LINE配信機能の実装完了 :tada:

<br>
