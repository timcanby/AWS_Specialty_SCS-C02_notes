<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [AWS Lambda × S3 ：クロスアカウントサーバーレス画像処理システムの安全設計](#aws-lambda-%C3%97-s3-%E3%82%AF%E3%83%AD%E3%82%B9%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E3%83%AC%E3%82%B9%E7%94%BB%E5%83%8F%E5%87%A6%E7%90%86%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0%E3%81%AE%E5%AE%89%E5%85%A8%E8%A8%AD%E8%A8%88)
  - [📘 Scenario（シナリオ）](#-scenario%E3%82%B7%E3%83%8A%E3%83%AA%E3%82%AA)
  - [Account A の Lambda 関数が Account B の S3 バケットへサムネイルを生成・保存する
このような場合、Lambda が S3 にアクセスするための適切で安全な認可設定が重要なポイントとなります。
クロスアカウント構成における安全設計のベストプラクティスを解説します。](#account-a-%E3%81%AE-lambda-%E9%96%A2%E6%95%B0%E3%81%8C-account-b-%E3%81%AE-s3-%E3%83%90%E3%82%B1%E3%83%83%E3%83%88%E3%81%B8%E3%82%B5%E3%83%A0%E3%83%8D%E3%82%A4%E3%83%AB%E3%82%92%E7%94%9F%E6%88%90%E3%83%BB%E4%BF%9D%E5%AD%98%E3%81%99%E3%82%8B%0A%E3%81%93%E3%81%AE%E3%82%88%E3%81%86%E3%81%AA%E5%A0%B4%E5%90%88lambda-%E3%81%8C-s3-%E3%81%AB%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%81%99%E3%82%8B%E3%81%9F%E3%82%81%E3%81%AE%E9%81%A9%E5%88%87%E3%81%A7%E5%AE%89%E5%85%A8%E3%81%AA%E8%AA%8D%E5%8F%AF%E8%A8%AD%E5%AE%9A%E3%81%8C%E9%87%8D%E8%A6%81%E3%81%AA%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88%E3%81%A8%E3%81%AA%E3%82%8A%E3%81%BE%E3%81%99%0A%E3%82%AF%E3%83%AD%E3%82%B9%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E6%A7%8B%E6%88%90%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8B%E5%AE%89%E5%85%A8%E8%A8%AD%E8%A8%88%E3%81%AE%E3%83%99%E3%82%B9%E3%83%88%E3%83%97%E3%83%A9%E3%82%AF%E3%83%86%E3%82%A3%E3%82%B9%E3%82%92%E8%A7%A3%E8%AA%AC%E3%81%97%E3%81%BE%E3%81%99)
  - [🎯 Test Points（考察ポイント）](#-test-points%E8%80%83%E5%AF%9F%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88)
  - [✅ Summary（まとめ）](#-summary%E3%81%BE%E3%81%A8%E3%82%81)
  - [🛠️ 実装例](#-%E5%AE%9F%E8%A3%85%E4%BE%8B)
    - [Account B: IAM ロール（ThumbXRole）](#account-b-iam-%E3%83%AD%E3%83%BC%E3%83%ABthumbxrole)
    - [Account A: Lambda 実行ロールに追加](#account-a-lambda-%E5%AE%9F%E8%A1%8C%E3%83%AD%E3%83%BC%E3%83%AB%E3%81%AB%E8%BF%BD%E5%8A%A0)
    - [Lambda コードで AssumeRole 実行](#lambda-%E3%82%B3%E3%83%BC%E3%83%89%E3%81%A7-assumerole-%E5%AE%9F%E8%A1%8C)
  - [🔐 セキュリティチェック](#-%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3%E3%83%81%E3%82%A7%E3%83%83%E3%82%AF)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# AWS Lambda × S3 ：クロスアカウントサーバーレス画像処理システムの安全設計
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

- Lambda関数が別アカウントのS3に安全にアクセスするには、IAMの「AssumeRole」や「バケットポリシー」の設計が鍵となります。
Account A の Lambda 関数が Account B の S3 バケットへサムネイルを生成・保存する
このような場合、Lambda が S3 にアクセスするための適切で安全な認可設定が重要なポイントとなります。
クロスアカウント構成における安全設計のベストプラクティスを解説します。
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
