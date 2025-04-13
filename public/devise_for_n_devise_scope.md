---
title: 'devise_for, devise_scopeの使い方'
tags:
  - Rails
  - devise
private: false
updated_at: '2025-04-13T19:04:13+09:00'
id: a7fa14c8bbe66bf846f8
organization_url_name: null
slide: false
ignorePublish: false
---

Railsアプリにdeviseを導入すると、routes.rbにこのように書いたりする。

```ruby:routes.rb
  devise_for :users, controllers: {
    sessions: 'users/sessions',
    registrations: 'users/registrations',
  }

  devise_scope :user do
    get 'signup', to: 'users/registrations#new', as: :signup
    get 'login', to: 'users/sessions#new', as: :login
    delete 'logout', to: 'users/sessions#destroy', as: :logout
  end
```

`devise_for`と`devise_scope`とは？その使い分けは？ を整理した。


## `devise_for`, `devise_scope` = devise独自のルート定義メソッド

`devise_for`, `devise_scope` = ActionDispatch::Routing::Mapperクラスのメソッド。

Routing::Mapperクラスには、`Rails.application.routes.draw do ... end`ブロック内で使える、ルート定義を行うメソッドたちが含まれる。

`get`, `delete`, `resources`などのRails標準搭載のRouting::Mapperクラスのメソッドに追加して、devise独自のメソッドとして`devise_for`, `devise_scope` などが定義されている。

## `devise_for` と `devise_scope`の使い分け

:::note info
結論
`devise_for`で標準ルーティングを一括生成して、さらに必要なカスタムルーティングを`devise_scope`で生成する。
:::

### devise_for 詳細

`devise_for`:  deviseを使うモデル (`User`など) で定義したmodule (e.g. `registerable`, `validatable`) をもとに、deviseの標準的なルーティングを一括生成。モデル, パス, コントロール etc.の指定ができる。
(`devise_for`の内部では`devise_scope`を呼んでいる)

```rb
devise_for :resources

# example
devise_for :users do
	class_name: 'Account', # モデルの指定 User -> Account
	# --- or --- 
	path: 'accounts', # パスの指定 /users -> /accounts
	# --- or --- 
	controllers: { sessions: 'users/sessions' } # コントローラの指定 devise/sessions -> users/sessions
end
```

参考：[devise rubydoc > devise_for](https://www.rubydoc.info/github/plataformatec/devise/ActionDispatch%2FRouting%2FMapper:devise_for)


### devise_scope 詳細
`devise_scope`: 新たにdevise関連のルーティングを追加する時に使う。パスとcontroller#actionを対応付ける。
(`scope` = deviseを使っているモデルに対応した識別子)

```rb
devise_scope :scope do 
	get '/some/route', to: 'some_devise_controller#action', as: :some_path_name
end

# `as`がエイリアスメソッドなので下記のようにも書ける
as :user do
	get 'login', to: 'devise/sessions#new', as: :login
end
```

参考：[devise rubydoc > devise_for](https://www.rubydoc.info/github/plataformatec/devise/ActionDispatch%2FRouting%2FMapper:devise_scope)

