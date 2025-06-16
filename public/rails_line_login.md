---
title: Ruby on RailsでLINEログイン機能を実装する手順
tags:
  - 'Ruby on Rails'
  - 'devise'
  - 'LINE'
  - 'OAuth'
  - 'OIDC'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

Ruby on RailsにLINEログイン機能を実装する手順をまとめました。

少し前の記事だと`omniauth-line`gemを使っているものが多いですが、GitHubを見る限りしばらくメンテナンスされていないgemなので、カスタムでstrategyを作る方法を取りました。

Webアプリケーションから外部Webサービス（今回だとLINE）での認証を行う時に何が起きているのか、まず整理します。

## 基本情報の整理
OIDC, OAuth2, strategy, provider, request phase, callback phase とは

## LINEログイン機能実装の手順
1. LINE Developersの登録
2. 必要なgemのインストール
3. custom strategyを作成（=provider）
4. deviseにproviderを登録（dot-env gem install & 環境変数の設定）
5. usersテーブルの構造変更（uid, providerカラムの追加）
6. ルーティング設定（LINE認可サーバからのリダイレクト先の設定 ← LINEでも登録？）
7. callbackコントローラの作成（LINE認可サーバから情報取得後の挙動）
8. user model method追加（LINE認証情報付加処理）



