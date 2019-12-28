# pipeline-for-pre-testing

## 概要

## 構成

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

```sh
aws cloudformation create-stack \
    --stack-name pipeline-for-pre-testing \
    --capabilities CAPABILITY_NAMED_IAM \
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
