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

ソースコードを変更してデプロイをするのを手動で行いその手順を把握してから、デプロイの自動化をしていきます。
