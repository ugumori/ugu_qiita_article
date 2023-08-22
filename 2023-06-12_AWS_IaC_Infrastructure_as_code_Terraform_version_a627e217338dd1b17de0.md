<!--
title:   terraformのバージョン Bump upを2年半ぶりに行った話
tags:    AWS,IaC,Infrastructure_as_code,Terraform,version
id:      a627e217338dd1b17de0
private: false
-->
我々のチームではAWSインフラをterraformを使って管理しています。

terraformを導入して2年半にもなるんですが、バージョン差分による混乱を避けるためチームでバージョンを統一して使っているので、簡単にBump upできず長いこと古いバージョンを使い続けてました。
しかし2年半にもなると、日進月歩なAWSのサービスに対し利用できないresourceが出てきたり、deprecatedになってるresourceを使い続けたりということが増えてきたので、意を決してBump upしました。
今回その内容を共有したいと思います。


## Bump up バージョン
- terraform
  - 0.14.2 -> 1.4.6
- terraform-provider-aws
  - 3.33.0 -> 4.64.0

メジャーバージョンが `0` -> `1` なあたり、隔世の感ある。

## 手順
terraformのclient、およびterraform-provider-awsのrequiredバージョンを更新。

```diff
 terraform {
-  required_version = "0.14.2"
+  required_version = "1.4.6"
   required_providers {
     aws = {
        source  = "hashicorp/aws"
-       version = "~> 3.33.0"
+       version = "~> 4.64.0"
     }
   }
 }
```

planを取ってバージョンアップによる差分を確認。量が膨大なのでファイルに出力する。
`zsh
terraform plan -out=plan_result.bin
terraform show -no-color plan_result.bin > plan_result.txt
`
1行目でplan結果をバイナリ出力し、
2行目でテキスト出力する。
こうやってバイナリを作っておくと、Plain TextではなくJSONを出したいときなどに改めて`terraform plan`する必要がなくなる（後述)。
resourceが膨大で何度もplanしたくないときに便利。


## `plan`差分確認
apply対象の値と異なりがないか確認する。

- AWSコンソールで現状の動作値と比較
- コンソール上で見れないものもあるので、AWS CLIも併用
- `sensitive` になっていて、何を`apply`しようとしてるかもterraform cli上で確認できない場合はJSON Dumpして確認する。やり方はとしては作成済みの`plan_result.bin`に対して、JSON出力を実行する。
    `zsh
    terraform show -json plan_result.bin  > plan_result.json
    `
  このJSONでは`sensitive`属性の値も確認できる。例えばこれはSecret Manager。
    `zsh
    jq  '.resource_changes[] | select(.address | contains("aws_secretsmanager_secret"))' plan_result.json
    {
        "address": "aws_secretsmanager_secret.my_secret",
        "mode": "managed",
        ...
    }
    `

- AWSではなくTerraform機能による属性の追加もある
    - 例えば `aws_security_group` の `revoke_rules_on_delete`。これはセキュリティグループのresourceを消す際、本属性を`true`にしておくと、アタッチされているリソースを自動的にデタッチしてくれるもの。こう言ったものを、必要か判断の上で設定する。今回のBump upでは全てデフォルトのままとした。


## 差分要因
具体的な数は省くが、膨大な差分が出た。
とは言っても差分の要因は同じものが多い。

差分の主な要因は以下。

1. 管理されていなかった属性が、新たに管理されるようになったことによるもの
各resourceや属性仕様が変わり、明示的に変更したいのでなければ指定する必要がなかった属性が(terraform provider的に)管理対象になった
1. 既存属性に新たにsensitive属性が付いたことによるもの
    - `password`や`key`といった機密性が高い属性はsensitive属性が付けられているものが多いが、既存属性に新たにsensitive属性がつけられた。
    - [terraform-provider-aws](https://github.com/hashicorp/terraform-provider-aws/blob/release/)のChangeLogを見るとある程度新たにsensitiveマークが付いた属性を確認できる（しかし、数が多すぎるのか全部は記載されていない模様）。


## apply
問題なければ`apply`。必要な動作確認をして完了。
`
terraform apply
`

terraform環境が複数あるなら、それぞれ差分確認、applyを行う。
アプリごとに環境が分かれてたり、さらに開発環境、ステージング環境、プロダクション環境とあるとかなり大変。

## まとめ
`plan`内容を1つ1つ確認しましたが、結果としてバージョンBump upによる`resource`や属性の変更は発生しなかったので、大層なコード修正も必要なかったです。メジャーバージョンの更新を含むので結構な覚悟をしていましたが、大きな問題はなし。terraform自身が後方互換をよく意識している証拠だと思います。
（それよりも、未コミットなコードがあって`plan`が取れない等、人為的なミスをいちいち担当者に確認するのが大変だった...）

これでまた2年しのげる...のかというとそれも微妙で、terraformは開発が活発で毎週バージョンが上がるので、やっぱりバージョン揃えて管理していくのはかなり面倒。
Terraform CloudやAtrantisの導入を考えたほうがいいかなぁ。

- [Terraform Cloud](https://app.terraform.io/)
- [Atrantis](https://www.runatlantis.io/)