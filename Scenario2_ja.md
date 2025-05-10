
# AWS Lambda × S3：クロスアカウント画像処理システムの安全設計

<p align="center">
  <img src="https://img.shields.io/badge/AWS-Lambda-green?logo=amazonaws&style=for-the-badge" />
  <img src="https://img.shields.io/badge/S3-Cross--Account-blue?logo=amazonaws&style=for-the-badge" />
</p>

> **目的**  
> “Account A の Lambda 関数が Account B の S3 バケットへサムネイルを生成・保存する”  
> **クロスアカウント構成**における安全設計のベストプラクティスを解説します。

---

## 📘 Scenario（シナリオ）

```
┌───────────┐            ┌──────────────┐
│ Account A │            │  Account B   │
│ (Producer)│            │ (Storage)    │
│           │ put object │              │
│Lambda Fn  ├──────────► │ S3 Bucket    │
│           │ get object │  (Images)    │
└───────────┘            └──────────────┘
```

Lambda関数が別アカウントのS3に安全にアクセスするには、IAMの「AssumeRole」や「バケットポリシー」の設計が鍵となります。

---

## 🎯 Test Points（考察ポイント）

- クロスアカウントでのアクセス許可の構成
- IAM ロールと信頼ポリシーの使い分け
- sts:AssumeRole による一時的なクレデンシャルの利用
- バケットポリシーにおける Principal 指定
- セキュリティベストプラクティスと監査手法

---

## ✅ Summary（まとめ）

| やること | 理由 |
|----------|------|
| Account B に IAM ロール (ThumbXRole) を作成 | 権限を明示的に定義 |
| 信頼ポリシーで Account A の Lambda 実行ロールを許可 | Cross-account のトラストを確立 |
| 権限ポリシーで S3 Get/Put を付与 | 最小権限 |
| Account A 側で sts:AssumeRole を許可 | 実行ロール側に Assume 権限付与 |
| Lambda から一時クレデンシャルでアクセス | 安全なアクセス |

---

## 🛠️ 実装例

### Account B: IAM ロール（ThumbXRole）

```json
// 信頼ポリシー
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::111111111111:role/AccountA-LambdaRole" },
    "Action": "sts:AssumeRole"
  }]
}
```

```json
// 権限ポリシー
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": [
    "arn:aws:s3:::account-b-thumb-bucket",
    "arn:aws:s3:::account-b-thumb-bucket/*"
  ]
}
```

---

### Account A: Lambda 実行ロールに追加

```json
{
  "Effect": "Allow",
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::222222222222:role/ThumbXRole"
}
```

---

### Lambda コードで AssumeRole 実行

```python
import boto3
import os

role_arn = os.environ["TARGET_ROLE_ARN"]
sts = boto3.client("sts")
creds = sts.assume_role(
    RoleArn=role_arn,
    RoleSessionName="LambdaCrossAccountSession"
)['Credentials']

s3 = boto3.client(
    's3',
    aws_access_key_id=creds['AccessKeyId'],
    aws_secret_access_key=creds['SecretAccessKey'],
    aws_session_token=creds['SessionToken']
)

# s3.get_object などで操作実行
```

---

## 🔐 セキュリティチェック

- IAM ポリシーは最小権限設計
- CloudTrail 有効化でアクセス追跡
- バケットポリシーで意図しないアクセスを拒否
- Access Analyzer で外部アクセス検査

---

これでクロスアカウントでも安全に Lambda → S3 の処理パイプラインを構築できます 💡
