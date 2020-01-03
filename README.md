# pipeline-for-pre-testing

## 概要

GitHubのプルリクエストが発生した際に、プルリクエストしたブランチとマージ先のブランチをマージ＋テストし、結果をプルリクエストに反映させるためのサンプル

## 構成

- GitHubのブランチのプルリクエストが発生した際、WebhookでAPI GatewayのエンドポイントをHook
- API Gatewayは、hookされた情報をS3に配置するLambdaを呼び出す（**WakeUpFunction**）
- S3にファイルを配置したことをトリガーにパイプラインを実行（**Source**）
- S3のhookされた情報からブランチを取得しマージ。マージが成功した場合、マージしたソースコードをS3に配置（**PreMerge**）
- S3に配置したソースコードに対してテストを実行（**Testing**）
- パイプラインの結果をCloudWatchEventsでキャッチし、実行した結果をGitHubAPIを使ってプルリクエストに結果を返すLambdaを呼び出す（**NotifyFunction**）

## デプロイ

**GitHubからソースコードを取得するためのアクセストークンを生成**

```sh
curl -u "your github username" \
     -d '{"scopes":["repo"],"note":"pipeline-for-ecs-deploy-v1"}' \
     https://api.github.com/authorizations
  # {
  #   ...
  #   "token": "774d8f6c********************************",
  #   ...
  # }
```

**生成したトークンをSSMパラメータストアに登録**

```sh
GITHUB_OAUTH_TOKEN=774d8f6c********************************
aws ssm put-parameter \
    --name pipeline-for-pre-testing-github-oauth-token \
    --value ${GITHUB_OAUTH_TOKEN} \
    --type String
  # {
  #     "Tier": "Standard",
  #     "Version": 1
  # }
```

**テンプレートから環境を構築**

- *TESTING_PROJECT_NAME*
  - テストを行うCodeBuildのプロジェクト名
- *TESTING_ROLE_NAME*
  - テストを行うCodeBuildのプロジェクトのサービスロール名
  - pipeline-for-pre-testingのS3バケットを参照するPolicyを、当該サービスロールに追加

*e.g.*

```sh
TESTING_PROJECT_NAME=<CodeBuild Project Name for Testing>
TESTING_ROLE_ARN=$(aws codebuild batch-get-projects \
    --names ${TESTING_PROJECT_NAME} \
    --query 'projects[].serviceRole' \
    --output text)
TESTING_ROLE_NAME=${TESTING_ROLE_ARN##*/}
```

```sh
aws cloudformation create-stack \
    --stack-name pipeline-for-pre-testing \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters ParameterKey=TestingProjectName,ParameterValue=${TESTING_PROJECT_NAME} \
                 ParameterKey=TestingServiceRoleName,ParameterValue=${TESTING_ROLE_NAME} \
    --template-body file://template.yaml
```

**WebHookで呼び出すURLを確認**

```sh
aws cloudformation describe-stacks \
    --stack-name pipeline-for-pre-testing \
    --query 'Stacks[].Outputs[?OutputKey==`InvokeURL`].OutputValue' \
    --output text
  # (e.g.) https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/v1
```

**対象のGitHubリポジトリのWebhookを登録**

Settings > Webhooks > Add webhook

- Payload URL: (e.g.) https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/v1
- Content type: application/json
- Secret: *empty*
- SSL verification: Enable SSL verification
- Which events would you like to trigger this webhook?: Let me select individual events.
  - Pull requests
- Active: *checked*

## アンデプロイ

**CloudFormationのステップ間でデータを受け渡すためのバケットを空にする**

```sh
S3BUCKET=$(aws cloudformation describe-stacks \
    --stack-name pipeline-for-pre-testing \
    --query 'Stacks[].Outputs[?OutputKey==`S3Bucket`].OutputValue' \
    --output text)
aws s3 rm s3://${S3BUCKET} --recursive
```

この構成では、S3バケットのオブジェクトを削除し、CloudFormationスタックを削除してもエラーが出ます。
これはS3バケットのバージョニングを有効にしているためです。パイプラインのSourceをS3バケットとしているため、S3バケットのバージョニングの有効は必須となっています。
これらは、aws-cliで削除することが出来ないため、S3バケットをコンソール画面から開き、バージョニングされているオブジェクトを削除してあげる必要があります。

**環境を削除**

```sh
aws cloudformation delete-stack \
    --stack-name pipeline-for-pre-testing
```
