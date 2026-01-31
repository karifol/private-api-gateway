# Private API Gateway Cross-Account Experiment

Account1の EC2 から Account2 の Private API Gateway を PrivateLink 経由で呼び出す構成。

## 構成概要

```
Account1                                  Account2
┌───────────────────────────┐             ┌──────────────────────────┐
│ VPC (10.0.0.0/16)         │             │                          │
│  ┌─ Public Subnet ─────┐  │             │  Private API Gateway     │
│  │  EC2 ──► VPC Endpoint ──PrivateLink──►  GET /prod/             │
│  │         (execute-api) │  │             │      │                  │
│  └───────────────────────┘  │             │      ▼                  │
│                             │             │  Lambda (Python 3.12)   │
└───────────────────────────┘             └──────────────────────────┘
```

## 前提条件

- AWS CLI がインストール済み
- Account1 / Account2 それぞれの AWS プロファイルが設定済み
- Account1 に EC2 キーペアが作成済み

## デプロイ手順

以下、AWS CLI プロファイル名を `account1`, `account2` と仮定。リージョンは `ap-northeast-1` を例にする。

### Step 1: Account2 にスタックをデプロイ

初回は `Account1VpceId` をデフォルト (`*`) のままデプロイする。

```bash
aws cloudformation deploy \
  --template-file account2/template.yaml \
  --stack-name private-api \
  --capabilities CAPABILITY_IAM \
  --profile account2 \
  --region ap-northeast-1
```

API Gateway の ID を確認しておく。

```bash
aws cloudformation describe-stacks \
  --stack-name private-api \
  --query 'Stacks[0].Outputs' \
  --output table \
  --profile account2 \
  --region ap-northeast-1
```

### Step 2: Account1 にスタックをデプロイ

自分のグローバル IP を確認する。

```bash
curl -s https://checkip.amazonaws.com
```

スタックをデプロイする。

```bash
aws cloudformation deploy \
  --template-file account1/template.yaml \
  --stack-name private-api-client \
  --parameter-overrides \
    KeyPairName=<your-key-pair-name> \
    MyIp=<your-ip>/32 \
  --profile account1 \
  --region ap-northeast-1
```

### Step 3: VPC Endpoint ID を取得

```bash
VPCE_ID=$(aws cloudformation describe-stacks \
  --stack-name private-api-client \
  --query 'Stacks[0].Outputs[?OutputKey==`VpceId`].OutputValue' \
  --output text \
  --profile account1 \
  --region ap-northeast-1)

echo $VPCE_ID
```

### Step 4: Account2 のスタックを更新（VPCE ID を設定）

Resource Policy を Account1 の VPC Endpoint に限定する。

```bash
aws cloudformation deploy \
  --template-file account2/template.yaml \
  --stack-name private-api \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides Account1VpceId=$VPCE_ID \
  --profile account2 \
  --region ap-northeast-1
```

> **Note**: API Gateway の Resource Policy を変更した後、API の再デプロイが必要になる場合がある。
> その場合は以下を実行する。
>
> ```bash
> API_ID=$(aws cloudformation describe-stacks \
>   --stack-name private-api \
>   --query 'Stacks[0].Outputs[?OutputKey==`ApiId`].OutputValue' \
>   --output text \
>   --profile account2 \
>   --region ap-northeast-1)
>
> aws apigateway create-deployment \
>   --rest-api-id $API_ID \
>   --stage-name prod \
>   --profile account2 \
>   --region ap-northeast-1
> ```

### Step 5: EC2 にSSHして動作確認

EC2 の Public IP を確認する。

```bash
EC2_IP=$(aws cloudformation describe-stacks \
  --stack-name private-api-client \
  --query 'Stacks[0].Outputs[?OutputKey==`Ec2PublicIp`].OutputValue' \
  --output text \
  --profile account1 \
  --region ap-northeast-1)

ssh -i <your-key-pair>.pem ec2-user@$EC2_IP
```

EC2 上で API を呼び出す。

```bash
# API_ID は Step 1 で確認した値、またはAccount2のスタック出力から取得
curl https://<API_ID>.execute-api.ap-northeast-1.amazonaws.com/prod/
```

期待するレスポンス:

```json
{"message": "Hello from private API"}
```

## クリーンアップ

削除は **Account1 → Account2** の順で行う。
Account2 の API Gateway Resource Policy が Account1 の VPC Endpoint を参照しているため、
先に Account2 を消すと依存関係でエラーになる場合がある。

### 1. Account1 のスタックを削除

```bash
aws cloudformation delete-stack \
  --stack-name private-api-client \
  --profile account1 \
  --region ap-northeast-1
```

削除完了を待つ:

```bash
aws cloudformation wait stack-delete-complete \
  --stack-name private-api-client \
  --profile account1 \
  --region ap-northeast-1
```

### 2. Account2 のスタックを削除

```bash
aws cloudformation delete-stack \
  --stack-name private-api \
  --profile account2 \
  --region ap-northeast-1
```

削除完了を待つ:

```bash
aws cloudformation wait stack-delete-complete \
  --stack-name private-api \
  --profile account2 \
  --region ap-northeast-1
```

### 3. 削除確認

両アカウントでスタックが残っていないことを確認する。

```bash
aws cloudformation list-stacks \
  --stack-status-filter DELETE_COMPLETE \
  --query 'StackSummaries[?StackName==`private-api-client`].[StackName,StackStatus]' \
  --output table \
  --profile account1 \
  --region ap-northeast-1

aws cloudformation list-stacks \
  --stack-status-filter DELETE_COMPLETE \
  --query 'StackSummaries[?StackName==`private-api`].[StackName,StackStatus]' \
  --output table \
  --profile account2 \
  --region ap-northeast-1
```

> **Note**: 削除に失敗した場合は `DELETE_FAILED` ステータスになる。
> `describe-stack-events` でエラー原因を確認し、手動でリソースを削除してからリトライする。
>
> ```bash
> aws cloudformation describe-stack-events \
>   --stack-name <stack-name> \
>   --query 'StackEvents[?ResourceStatus==`DELETE_FAILED`].[LogicalResourceId,ResourceStatusReason]' \
>   --output table \
>   --profile <profile> \
>   --region ap-northeast-1
> ```

## トラブルシューティング

| 症状 | 対処 |
|------|------|
| curl がタイムアウトする | VPC Endpoint の SG で 443 が許可されているか確認。Private DNS が有効か確認。 |
| 403 Forbidden | API Gateway の Resource Policy の `aws:sourceVpce` が正しい VPCE ID か確認。Step 4 の再デプロイを実行。 |
| SSH できない | `MyIp` パラメータが自分の IP と一致しているか確認。 |
