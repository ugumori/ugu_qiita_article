<!--
title:   Lambda + FastAPI + Dockerで最小限&サーバーレスなAPIをデプロイする
tags:    APIGateway,Docker,FastAPI,Mangum,lambda
id:      ffcff085daa4536444dc
private: false
-->
## AWSで サーバーレス API のデプロイってちょっと大変

AWSでサーバーレスなBackend APIを作りたい時、まずAPI Gatewayを選択肢に挙げるBackendエンジニャー😺な方は多いかと思います。

ご存知の通り、API Gatewayは単体ではコードロジックを実行できません。
なので、API Gateway -> LambdaのようにProxyさせるやり方は、ど定番で誰もがやったことあるパターンです。

とはいっても、API Gatewayは独自な仕様が多く、AWSコンソールからいじるのも色々とキツく、かといってterraformで書くのも冗長な感じで、なかなか使いこなせないと思っているAWSユーザも多いかと思います。
（少なくとも私はそうです）

API Gatewayに頼らずもっと簡単にサーバーレスしたい。
そこで、**Mangum** とLambdaのFunctionURLを使い、Lambda単体でBackend APIをデプロイしてみましょう。
デプロイし、ブラウザでAPIを叩けたらゴールです。

## Mangum
https://mangum.io/

>Mangum は、AWS Lambda でASGIアプリケーションを実行し、関数 URL、API Gateway、ALB、Lambda@Edge イベントを処理するためのアダプタ


ということです（原文訳)。
今回は関数URLからのイベントを処理するということですね。

## 構成
- python3
- Lambda + FastAPI + Docker

## コード
`py`ファイルはこれだけです。

```python:main.py
from fastapi import FastAPI
from starlette.middleware.cors import CORSMiddleware
from mangum import Mangum

_app = FastAPI()
@_app.get("/health", include_in_schema=True)
async def health():
    print("Called")
    return "ok"

app = CORSMiddleware(
    app=_app,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# これだけ
handler = Mangum(app, lifespan="off")
```
Mangum独自の実装は、FastAPIインスタンスをWrapする1行だけ。
超簡単ですね。
※ガバガバなCORS入れてますが、適宜ちゃんと設定しましょう。

```requirements.txt
fastapi
mangum
starlette
```

DockerファイルにはエントリポイントとしてMangumインスタンスの`main.handler`を指定します。

```Dockerfile
FROM public.ecr.aws/lambda/python:3.10

RUN yum -y install git

RUN mkdir -p -m 0600 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts
COPY requirements.txt .
RUN --mount=type=ssh pip3 install -r requirements.txt --target "${LAMBDA_TASK_ROOT}"

# Copy function code
COPY . ${LAMBDA_TASK_ROOT}

# Set the CMD to your handler (could also be done as a parameter override outside of the Dockerfile)
CMD [ "main.handler" ] #　これ
```

## LambdaとFunction URLの作成
terraformはこんな感じです（本筋から外れるresourceは省略。GitHubを参照）。
`aws_lambda_function_url`でFunction URLを生成します。
`aws_lambda_permission`でリソースベースのポリシーを付与するのがミソです。


```terraform
resource "aws_lambda_function" "api" {
  function_name = "lambda_minimum_serverless_api"
  role          = aws_iam_role.api.arn
  package_type  = "Image"
  memory_size   = "512"
  timeout       = "60"
  image_uri     = "${local.aws_account_id}.dkr.ecr.${local.region}.amazonaws.com/${aws_ecr_repository.api.name}:latest"
}

resource "aws_lambda_function_url" "api_url" {
  function_name      = aws_lambda_function.api.function_name
  authorization_type = "NONE"
  cors {
    allow_credentials = false
    allow_headers = [
      "*",
    ]
    allow_methods = [
      "*",
    ]
    allow_origins = [
      "*",
    ]
    expose_headers = [
      "*",
    ]
    max_age = 0
  }
}

resource "aws_lambda_permission" "api_url" {
  statement_id           = "FunctionURLAllowPublicAccess"
  action                 = "lambda:InvokeFunctionUrl"
  function_name          = aws_lambda_function.api.arn
  function_url_auth_type = "NONE"
  principal              = "*"
}

output "api_url" {
  value = aws_lambda_function_url.api_url.function_url
}

```

## デプロイ

Dockerイメージを生成し、Lambdaへデプロイしましょう。
AWS ECRにイメージをpushし、そこからデプロイします。
ECRのリポジトリ名は`apps/sample/api`とします。

### ECRへのpush
イメージをビルドしましょう。
`shell
$ tar -cJh . | docker build -t apps/sample/api:latest -
`
ECR用のtagをつけます。
YOUR_ACCOUNT_IDはご自分のAWSアカウントIDを指定してください。
`shell
$ docker tag apps/sample/api ${YOUR_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/apps/sample/api
`
イメージができたので、ECRにpushします。
`shell
$ aws ecr get-login-password --region ap-northeast-1 | docker login  --username AWS --password-stdin ${YOUR_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com
$ docker push ${YOUR_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/apps/sample/api
Using default tag: latest
The push refers to repository [*************.dkr.ecr.ap-northeast-1.amazonaws.com/apps/sample/api]
3081d709ee8a: Pushed
...
latest: digest: sha256:d12aa06e6966509f877d5da4b61b0d87b68a6fab21409b7492f81714a7fa3cbe size: ****
`
sha256はLambdaをデプロイする時に使いますので控えておきます。

### Lambdaデプロイ
AWS CliからLambdaをupdateします。
`--image-uri`の指定で、上記のsha256を指定してください。
`shell
$ aws lambda update-function-code --region ap-northeast-1 --function-name lambda_minimum_serverless_api --image-uri ${YOUR_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/apps/sample/api@sha256:d12aa06e6966509f877d5da4b61b0d87b68a6fab21409b7492f81714a7fa3cbe
`

デプロイ完了です！

## 動作確認
ブラウザからdocsにアクセスしてみましょう！
`https://********************************.lambda-url.ap-northeast-1.on.aws/docs`

![capcap.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/137301/3355eaa3-5034-5dc6-3c7b-cdd8fbeeb4f7.png)

URLを見て頂くと分かる通り、LambdaのFunction URLにアクセスしています。
Lambda単独で、SwaggerUIからFastAPIで定義した`/health`が打てています！
Yeah！！！

## まとめ
Mangumを使ってLambda + FastAPI + Dockerで最小限&サーバーレスなAPIをデプロイし、ブラウザからAPIを打てることを確認しました。
API GatewayはCognitoとラクに連携できたり、APIのライフサイクル管理ができたりと多機能ですが、
「シンプルに何かのAPIを試したい」、「PoCやMVP用にAPIを作りたい」のような時は、今回のようなLambda単独の最小限な構成でAPIを作れると、とても便利ですね。


## GitHub
この記事のソースコードは全てこちらに置いてます。
terraformのコードもあるのでご参考に。

https://github.com/ugumori/lambda_minimum_serverless_api