# Private API Gateway Cross-Account Experiment

Account1 の EC2 から Account2 の Private API Gateway を PrivateLink 経由で呼び出す構成。

Private API Gateway はパブリックインターネットに公開されないため、ローカル PC から直接呼び出すことはできない。
VPC 内の VPC Endpoint 経由でのみアクセス可能。

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

ローカル PC ──✕──► Private API Gateway  (アクセス不可)
```

## 前提条件

- AWS CLI がインストール済み
- Account1 / Account2 それぞれの AWS プロファイルが設定済み
- Account1 に EC2 キーペアが作成済み（なければ `aws ec2 create-key-pair` で作成）
- **両アカウントで同じリージョンを使用すること**

## デプロイ手順

以下、AWS CLI プロファイル名を `account1`, `account2` と仮定。リージョンは `ap-northeast-1` を例にする。

> **重要**: Account1 と Account2 は必ず同じリージョンにデプロイすること。
> VPC Endpoint は同一リージョンの execute-api サービスにしか接続できないため、
> リージョンが異なると DNS 解決に失敗する。

### Step 1: Account1 にスタックをデプロイ（VPC + EC2 + VPC Endpoint）

キーペアがなければ作成する。

```bash
aws ec2 create-key-pair \
  --key-name private-api-test \
  --query 'KeyMaterial' \
  --output text \
  --profile account1 \
  --region ap-northeast-1 > private-api-test.pem

chmod 400 private-api-test.pem
```

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
    KeyPairName=private-api-test \
    MyIp=<your-ip>/32 \
  --profile account1 \
  --region ap-northeast-1
```

### Step 2: VPC Endpoint ID を取得

```bash
VPCE_ID=$(aws cloudformation describe-stacks \
  --stack-name private-api-client \
  --query 'Stacks[0].Outputs[?OutputKey==`VpceId`].OutputValue' \
  --output text \
  --profile account1 \
  --region ap-northeast-1)

echo $VPCE_ID
```

### Step 3: Account2 にスタックをデプロイ（API Gateway + Lambda）

Account1 の VPC Endpoint ID を指定してデプロイする。

```bash
aws cloudformation deploy \
  --template-file account2/template.yaml \
  --stack-name private-api \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides Account1VpceId=$VPCE_ID \
  --profile account2 \
  --region ap-northeast-1
```

API Gateway の ID と Invoke URL を確認する。

```bash
aws cloudformation describe-stacks \
  --stack-name private-api \
  --query 'Stacks[0].Outputs' \
  --output table \
  --profile account2 \
  --region ap-northeast-1
```

### Step 4: EC2 にSSHして動作確認

EC2 の Public IP を確認する。

```bash
EC2_IP=$(aws cloudformation describe-stacks \
  --stack-name private-api-client \
  --query 'Stacks[0].Outputs[?OutputKey==`Ec2PublicIp`].OutputValue' \
  --output text \
  --profile account1 \
  --region ap-northeast-1)

ssh -i private-api-test.pem ec2-user@$EC2_IP
```

EC2 上で API を呼び出す。

```bash
# API_ID は Step 3 で確認した値
curl https://<API_ID>.execute-api.ap-northeast-1.amazonaws.com/prod/
```

期待するレスポンス:

```json
{"message": "Hello from private API"}
```

ローカル PC から同じ URL を curl するとアクセスできないことも確認できる。

```bash
# ローカル PC から実行 → タイムアウトまたは接続エラーになる
curl https://<API_ID>.execute-api.ap-northeast-1.amazonaws.com/prod/
```

## 注意事項

- **クロスアカウントでは `VpcEndpointIds` は使用不可**: API Gateway の `EndpointConfiguration.VpcEndpointIds` には同一アカウント内の VPC Endpoint しか指定できない。クロスアカウントの場合は Resource Policy の `aws:sourceVpce` 条件のみでアクセス制御する。
- **リージョンを揃える**: Account1 の VPC Endpoint と Account2 の API Gateway は同じリージョンに作成する必要がある。

## クリーンアップ

削除は **Account2 → Account1** の順で行う。
Account2 の API Gateway が Account1 の VPC Endpoint ID を Resource Policy で参照しているため、
先に Account1 を消すと Account2 側の更新・削除時に不整合が起きる可能性がある。

### 1. Account2 のスタックを削除

```bash
aws cloudformation delete-stack \
  --stack-name private-api \
  --profile account2 \
  --region ap-northeast-1

aws cloudformation wait stack-delete-complete \
  --stack-name private-api \
  --profile account2 \
  --region ap-northeast-1
```

### 2. Account1 のスタックを削除

```bash
aws cloudformation delete-stack \
  --stack-name private-api-client \
  --profile account1 \
  --region ap-northeast-1

aws cloudformation wait stack-delete-complete \
  --stack-name private-api-client \
  --profile account1 \
  --region ap-northeast-1
```

### 3. キーペアの削除（必要に応じて）

```bash
aws ec2 delete-key-pair \
  --key-name private-api-test \
  --profile account1 \
  --region ap-northeast-1

rm -f private-api-test.pem
```

### 4. 削除確認

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
| DNS 解決できない (`Could not resolve host`) | Account1 と Account2 のリージョンが一致しているか確認。VPC Endpoint の Private DNS が有効か確認。 |
| curl がタイムアウトする | VPC Endpoint の SG で 443 が許可されているか確認。 |
| 403 Forbidden | API Gateway の Resource Policy の `aws:sourceVpce` が正しい VPCE ID か確認。 |
| SSH できない | `MyIp` パラメータが自分の現在の IP と一致しているか確認。 |
| ローカルからアクセスできない | 正常な動作。Private API Gateway は VPC Endpoint 経由でのみアクセス可能。 |
