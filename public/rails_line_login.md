---
title: Ruby on RailsでLINEログイン機能を実装する手順
tags:
  - 'Ruby on Rails'
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

Ruby on RailsにLINEログイン機能を実装する手順をまとめました。

LINEログインをRailsアプリに組み込む方法として、少し前の記事だと`omniauth-line`gemを使っているものが多いですが、GitHubを見る限りしばらくメンテナンスされていないgemなので、カスタムでstrategyを作る方法を取りました。

:::note info 
Ruby, gemのバージョン：
  - Ruby: 3.4.3
  - rails gem: 8.0.2
  - omniauth gem: 2.1.3
  - omniauth-oauth2 gem: 1.8.0
:::

[実装手順 ↓](#lineログイン機能実装の手順)の前に、Webアプリケーションから外部Webサービス（今回だとLINE）での認可・認証について、整理します。

## 認証・認可の基本情報
（OAuth, OIDC, access token, ID token, request phase, callback phase, OmniAuth, strategy, provider）
- **OAuth**：Webサービスにおいて、リソースへのアクセス許可を安全に委譲する認可の仕組み。
  - **OAuth 2.0**ではクライアントアプリ（今回だとRailsアプリ）が、認可サーバ（今回だとLINE）に対してアクセストークンを要求し、認可サーバはユーザーの許可を得てクライアントアプリにアクセストークンを発行する。
  - ちなみに... OAuth 1.0は認証フローが複雑＆対応アプリに制限あり＆すべてのAPIリクエストで署名必須だった。OAuth 2.0は、OAuth 1.0の問題点を解決し、より柔軟で使いやすい認証・認可フレームワークを提供している。
  - 参考：
    - [徹底解説：OAuth 1.0とOAuth 2.0の違い](https://apidog.com/jp/blog/oauth-1-2-difference/)
    - [一番分かりやすい OAuth の説明](https://qiita.com/TakahikoKawasaki/items/e37caf50776e00e733be)
  <br>
- **OIDC** (OpenID Connect): 異なるWebサービス間における認証の仕組み。（OAuth 2.0の拡張仕様）
  - クライアントアプリがOpenIDプロバイダー（今回だとLINE）にIDトークンを要求し、ユーザーの認証＆許可取得後に、OpenIDプロバイダーがクライアントアプリにIDトークンを発行する。
  - アクセストークン v.s. IDトークン
    - **アクセストークン**：OAuth 2.0で定義された、リソースへのアクセスを認可するためのトークン
      - LINEの場合：「このWebアプリに対して、このLINEユーザのIDとメールアドレスを渡せるよ。ユーザの許可もらったから」
    - **IDトークン**：OIDCで定義された、ユーザが認証されたことを証明するトークン
      - LINEの場合：「このユーザーはLINEログイン成功済み。このユーザーのIDとメールアドレス情報練りこんであるよ」
    - LINEの場合の参考：[LINEログイン v2.1 APIリファレンス](https://developers.line.biz/ja/reference/line-login/)
  <br>
- **LINEログインのフロー（OAuth 2.0 + OIDC）** [🔗 フロー図](https://developers.line.biz/ja/docs/line-login/integrate-line-login/#login-flow)
  1. ユーザーが「LINEログイン」ボタン押下
  2. WebアプリがLINE認可サーバにアクセスし、LINEログイン画面にリダイレクト
  3. ユーザーがログイン画面で同意（=認証＆認可）すると、LINE認可サーバはWebアプリに認可コードを発行
  4. WebアプリはLINE認可サーバに`アクセストークン`を発行要求（受け取った認可コードを添付）
  5. LINE認可サーバは認可コードを検証 → `アクセストークン`を発行（`IDトークン`, `scope`を添付。`scope`は2でWebアプリから認可サーバに送ったパラメータのひとつでアクセス権限を定義）
  6. WebアプリはLINE認可サーバにユーザー情報を要求（`IDトークン`を添付）
  7. LINE認可サーバは`IDトークン`を検証 → LINE ID, LINEユーザー名などをWebアプリに返す
  8. Webアプリにて、返されたユーザー情報を元にusers#create, sessions#createを行う
  （1~2: **request phase**, 3~8: **callback phase**） 
  <br>
- **OmniAuthにおけるstrategy, provider**
  - **OmniAuth**: "multi-provider Authentication"。いろんな認証フローを標準化したライブラリ。
  - **provider**: どの認証フロー(=**strategy**)を使うかの設定 ← by Rackミドルウェアにstrategyを登録
  - **strategy**: コントローラと認可サーバとの間の窓口係として、request / callback phaseにて二者の間に立って処理をする。
    - （request phase 例） 上記フローの1~2にて、LINEログインボタン押下後に`omniauth_callbacks#passthru`を通過後、line providerがリクエストのパラメータなどを取りまとめてLINE認可サーバへ投げる。
    - （callback phase 例） 上記フローの3にて、LINE認可サーバから発行された認可コードは、まずline providerが受け取り、アクセストークン＆IDトークンのやりとりを経て取得したLINEユーザ情報をセットし、`omniauth_callbacks#line`へ処理が引き継がれる
  - 参考：[OmniAuth GitHub](https://github.com/omniauth/omniauth)

## LINEログイン機能実装の手順
1. **LINEログインチャネルの作成** [🔗 公式doc](https://developers.line.biz/ja/docs/line-login/integrate-line-login/#create-a-channel)
   1. WebアプリとLINEプラットフォームを接続するチャネルをLINE Developersコンソールで作成
   2. メールアドレスの取得権限を申請（LINEログイン画面に表示される文言のスクショでOK）
   <br>
2. **gemのインストール**
   ```rb:Gemfile
   gem 'omniauth-oauth2' # devise ❓
   gem 'omniauth-rails_csrf_protection'
   gem 'dotenv-rails' # 環境変数の管理 (LINE_CHANNEL_ID, SECRET etc.)
   ```
3. **custom strategyを作成**（=provider）
4. deviseにproviderを登録（dot-env gem install & 環境変数の設定）
   1. 環境変数の設定：dev環境では.envに、prod環境ではAWS Secrets Managerに登録し、ECS ServiceのCFnテンプレートにてSecrets Managerの値を参照
5. usersテーブルの構造変更（uid, providerカラムの追加）
6. ルーティング設定（LINE認可サーバからのリダイレクト先の設定）
   1. routes.rb
   2. LINEログインチャネルにて、コールバックURL（=上記フローの3にて、ユーザの認証＆認可後に認可コードを受け取るWebアプリのURL）を設定
7. callbackコントローラの作成（LINE認可サーバから情報取得後の挙動）
8. user model method追加（LINE認証情報付加処理）



