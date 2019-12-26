# pipeline-for-pre-testing

## 概要

## 構成

## デプロイ

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

Settings > Webhooks > Add webhook

- Payload URL: (e.g.) https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/v1
- Content type: application/json
- Secret: *empty*
- SSL verification: Enable SSL verification
- Which events would you like to trigger this webhook?: Let me select individual events.
  - Pull requests
- Active: *checked*
