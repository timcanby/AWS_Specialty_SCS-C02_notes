<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [AWS Lambda × S3 ：サーバーレス画像処理システムの安全設計](#aws-lambda-%C3%97-s3-%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E3%83%AC%E3%82%B9%E7%94%BB%E5%83%8F%E5%87%A6%E7%90%86%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0%E3%81%AE%E5%AE%89%E5%85%A8%E8%A8%AD%E8%A8%88)
  - [📘 Scenario（シナリオ）](#-scenario%E3%82%B7%E3%83%8A%E3%83%AA%E3%82%AA)
  - [🎯 Test Points（考察ポイント）](#-test-points%E8%80%83%E5%AF%9F%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88)
  - [🧠 Related Knowledge（関連知識）](#-related-knowledge%E9%96%A2%E9%80%A3%E7%9F%A5%E8%AD%98)
    - [IAM ユーザー vs. IAM ロール](#iam-%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC-vs-iam-%E3%83%AD%E3%83%BC%E3%83%AB)
    - [IAM ポリシー](#iam-%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC)
    - [S3 バケットポリシー](#s3-%E3%83%90%E3%82%B1%E3%83%83%E3%83%88%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC)
    - [AWS Secrets Manager（参考）](#aws-secrets-manager%E5%8F%82%E8%80%83)
    - [EC2 Key Pair](#ec2-key-pair)
    - [Security Group](#security-group)
  - [✅ Summary（まとめ）](#-summary%E3%81%BE%E3%81%A8%E3%82%81)
    - [推奨されるアプローチ](#%E6%8E%A8%E5%A5%A8%E3%81%95%E3%82%8C%E3%82%8B%E3%82%A2%E3%83%97%E3%83%AD%E3%83%BC%E3%83%81)
    - [避けるべきアプローチ](#%E9%81%BF%E3%81%91%E3%82%8B%E3%81%B9%E3%81%8D%E3%82%A2%E3%83%97%E3%83%AD%E3%83%BC%E3%83%81)
  - [IAMロールにおける「信頼ポリシー」と「権限ポリシー」の違い](#iam%E3%83%AD%E3%83%BC%E3%83%AB%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8B%E4%BF%A1%E9%A0%BC%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E3%81%A8%E6%A8%A9%E9%99%90%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E3%81%AE%E9%81%95%E3%81%84)
    - [💡 イメージで理解する](#-%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%81%A7%E7%90%86%E8%A7%A3%E3%81%99%E3%82%8B)
    - [例：Lambdaの実行ロールの場合](#%E4%BE%8Blambda%E3%81%AE%E5%AE%9F%E8%A1%8C%E3%83%AD%E3%83%BC%E3%83%AB%E3%81%AE%E5%A0%B4%E5%90%88)
- [Lambda × S3：バケットポリシーによるリソース側集中管理の設定手順](#lambda-%C3%97-s3%E3%83%90%E3%82%B1%E3%83%83%E3%83%88%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E3%81%AB%E3%82%88%E3%82%8B%E3%83%AA%E3%82%BD%E3%83%BC%E3%82%B9%E5%81%B4%E9%9B%86%E4%B8%AD%E7%AE%A1%E7%90%86%E3%81%AE%E8%A8%AD%E5%AE%9A%E6%89%8B%E9%A0%86)
  - [✅ ステップ 1：Lambda実行ロールの作成（例：LambdaThumbRole）](#-%E3%82%B9%E3%83%86%E3%83%83%E3%83%97-1lambda%E5%AE%9F%E8%A1%8C%E3%83%AD%E3%83%BC%E3%83%AB%E3%81%AE%E4%BD%9C%E6%88%90%E4%BE%8Blambdathumbrole)
    - [🔸 信頼ポリシー（Trust Policy）](#-%E4%BF%A1%E9%A0%BC%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BCtrust-policy)
    - [🔸 最小権限のIAMポリシー（ログ出力用）](#-%E6%9C%80%E5%B0%8F%E6%A8%A9%E9%99%90%E3%81%AEiam%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E3%83%AD%E3%82%B0%E5%87%BA%E5%8A%9B%E7%94%A8)
  - [✅ ステップ 2：S3バケット側にBucket Policyを設定](#-%E3%82%B9%E3%83%86%E3%83%83%E3%83%97-2s3%E3%83%90%E3%82%B1%E3%83%83%E3%83%88%E5%81%B4%E3%81%ABbucket-policy%E3%82%92%E8%A8%AD%E5%AE%9A)
  - [✅ ステップ 3（任意）：S3イベント通知でLambdaをトリガー](#-%E3%82%B9%E3%83%86%E3%83%83%E3%83%97-3%E4%BB%BB%E6%84%8Fs3%E3%82%A4%E3%83%99%E3%83%B3%E3%83%88%E9%80%9A%E7%9F%A5%E3%81%A7lambda%E3%82%92%E3%83%88%E3%83%AA%E3%82%AC%E3%83%BC)
- [✅ Lambda実行ロールとS3バケットポリシーの権限評価のポイントまとめ](#-lambda%E5%AE%9F%E8%A1%8C%E3%83%AD%E3%83%BC%E3%83%AB%E3%81%A8s3%E3%83%90%E3%82%B1%E3%83%83%E3%83%88%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E3%81%AE%E6%A8%A9%E9%99%90%E8%A9%95%E4%BE%A1%E3%81%AE%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88%E3%81%BE%E3%81%A8%E3%82%81)
  - [💡 前提の構成（今回の例）](#-%E5%89%8D%E6%8F%90%E3%81%AE%E6%A7%8B%E6%88%90%E4%BB%8A%E5%9B%9E%E3%81%AE%E4%BE%8B)
  - [💡 それでもなぜ Lambda から S3 にアクセスできるのか？](#-%E3%81%9D%E3%82%8C%E3%81%A7%E3%82%82%E3%81%AA%E3%81%9C-lambda-%E3%81%8B%E3%82%89-s3-%E3%81%AB%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%81%A7%E3%81%8D%E3%82%8B%E3%81%AE%E3%81%8B)
  - [✅ 結論：S3など一部のサービスでは、リソースポリシーだけでアクセスが許可される](#-%E7%B5%90%E8%AB%96s3%E3%81%AA%E3%81%A9%E4%B8%80%E9%83%A8%E3%81%AE%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E3%81%A7%E3%81%AF%E3%83%AA%E3%82%BD%E3%83%BC%E3%82%B9%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E3%81%A0%E3%81%91%E3%81%A7%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%81%8C%E8%A8%B1%E5%8F%AF%E3%81%95%E3%82%8C%E3%82%8B)
  - [🎯 ポイントまとめ](#-%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88%E3%81%BE%E3%81%A8%E3%82%81)
  - [🔐 セキュリティ的なメリット](#-%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3%E7%9A%84%E3%81%AA%E3%83%A1%E3%83%AA%E3%83%83%E3%83%88)
  - [✅ この構成は「リソース側集中管理型」の代表例です](#-%E3%81%93%E3%81%AE%E6%A7%8B%E6%88%90%E3%81%AF%E3%83%AA%E3%82%BD%E3%83%BC%E3%82%B9%E5%81%B4%E9%9B%86%E4%B8%AD%E7%AE%A1%E7%90%86%E5%9E%8B%E3%81%AE%E4%BB%A3%E8%A1%A8%E4%BE%8B%E3%81%A7%E3%81%99)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# AWS Lambda × S3 ：サーバーレス画像処理システムの安全設計

## 📘 Scenario（シナリオ）

企業が、**AWS Lambda 関数を使用して大きな画像からサムネイル画像を生成**するシステムを構築しています。  
Lambda 関数は、**同一 AWS アカウント内の S3 バケットにアクセス**し、元画像を読み込み、生成されたサムネイルを保存します。

この構成は、**S3 のアップロードイベントによって Lambda をトリガーし、処理結果を再び S3 に保存する**という、典型的な AWS のイベント駆動型サーバーレスアーキテクチャです。

このような場合、**Lambda が S3 にアクセスするための適切で安全な認可設定**が重要なポイントとなります。

---

## 🎯 Test Points（考察ポイント）

- Lambda から S3 への安全なアクセス権限の設定方法
- IAM の基本理解（IAM ユーザー、IAM ロール、IAM ポリシーの違いと適用）
- S3 バケットポリシーの仕組みと IAM ポリシーとの連携
- 長期クレデンシャルを避ける AWS のセキュリティベストプラクティス
- IAM/バケットポリシー と Security Group などのネットワーク制御の違いの理解

---

## 🧠 Related Knowledge（関連知識）

### IAM ユーザー vs. IAM ロール

- IAM ユーザーは人間またはアプリケーション向けで、静的なアクセスキーを持つ。
- IAM ロールは AWS サービス（Lambda、EC2 など）に一時的なクレデンシャルを提供し、**サービス間のアクセスに最適**。
- Lambda には IAM ロールを割り当てることで、他の AWS サービス（例：S3）へのアクセスを安全に許可可能。

### IAM ポリシー

- IAM ポリシーは「どの操作を、どのリソースに対して許可するか」を定義する文書。
- IAM ロールにアタッチされたポリシーによって、Lambda が `s3:GetObject` や `s3:PutObject` などの操作を実行可能にする。

### S3 バケットポリシー

- S3 に直接設定できるリソースベースのポリシー。
- IAM ロールを Principal として明示的に許可することで、Lambda 側での IAM 設定とは別にバケット側からアクセスを管理可能。
- IAM ポリシーと併用することで、アクセス制御の粒度を細かく調整できる。

### AWS Secrets Manager（参考）

- パスワードや API キーなどの機密情報を安全に管理するためのサービス。
- **EC2 キーペアを保存して S3 にアクセスする用途では適切でない**。

### EC2 Key Pair

- SSH で EC2 にログインするための認証情報。
- **S3 API へのアクセスには使用されない**。

### Security Group

- ネットワークレベル（L3/L4）でのアクセス制御（IP/ポート単位）。
- **S3 の API アクセス権限には関与せず、IAM やバケットポリシーで制御する必要がある**。

---

## ✅ Summary（まとめ）

このシナリオは、サーバーレスアーキテクチャにおける代表的なデータ処理パターンであり、  
**Lambda が S3 に安全かつ確実にアクセスするための適切な IAM 設計が問われるケース**です。

### 推奨されるアプローチ

- ✅ Lambda に IAM ロールを割り当て、そのロールに S3 アクセス権限を付与（IAM ポリシー使用）
- ✅ S3 バケットにバケットポリシーを設定し、Lambda の IAM ロールを Principal としてアクセス許可

### 避けるべきアプローチ

- ❌ IAM ユーザーのアクセスキーを Lambda に埋め込む（長期クレデンシャルの使用）
- ❌ Secrets Manager に EC2 キーペアを保存して S3 アクセスに使用
- ❌ Security Group による S3 アクセス制御の誤用

Lambda と S3 の統合においては、**IAM ロールを中心に設計するのが AWS のベストプラクティス**です。


## IAMロールにおける「信頼ポリシー」と「権限ポリシー」の違い

| 種類             | 対象             | 内容の役割                                      | 誰がこのロールを引き受けられるか       | このロールは何ができるか               |
|------------------|------------------|--------------------------------------------------|----------------------------------------|----------------------------------------|
| **信頼ポリシー** | IAMロール自身     | **どのサービス/ユーザーがこのロールを引き受けられるか**を定義 | ✅ 例：Lambdaサービスがロールを引き受ける | ❌ リソースへのアクセス権限は含まない     |
| **権限ポリシー** | IAMロール自身     | **このロールがどのAWSリソースに何の操作ができるか**を定義     | ❌ 引き受ける相手は制御しない             | ✅ 例：S3バケットの読み書き、ログ出力など |

### 💡 イメージで理解する

- **信頼ポリシー**：このロールを「誰が着る（Assume）ことができるか」の設定（＝ユニフォームの貸出対象）
- **権限ポリシー**：このロールを着た人が「何ができるか」の設定（＝社員証のアクセス範囲）

---

### 例：Lambdaの実行ロールの場合

- 信頼ポリシーで `lambda.amazonaws.com` に引き受けを許可
- 権限ポリシーで CloudWatch Logs と S3バケットのアクセスを許可

両方の設定が揃って初めて、安全かつ正しくリソースにアクセスできます。

# Lambda × S3：バケットポリシーによるリソース側集中管理の設定手順

## ✅ ステップ 1：Lambda実行ロールの作成（例：LambdaThumbRole）

### 🔸 信頼ポリシー（Trust Policy）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
### 🔸 最小権限のIAMポリシー（ログ出力用）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```
このロールに付与するIAMポリシーは、CloudWatch Logsへの出力に必要な最小限の権限のみを持たせます。
※ S3アクセス権限はバケットポリシーで制御します。

## ✅ ステップ 2：S3バケット側にBucket Policyを設定

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowLambdaThumbRoleAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/LambdaThumbRole"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-image-bucket",
        "arn:aws:s3:::my-image-bucket/*"
      ]
    }
  ]
}
```
## ✅ ステップ 3（任意）：S3イベント通知でLambdaをトリガー

S3の「イベント通知」設定で、ObjectCreated イベント発生時に Lambda 関数をトリガーするよう設定します。
➡ ファイルがアップロードされたら自動的にLambdaが呼び出され、サムネイル生成を実行。


# ✅ Lambda実行ロールとS3バケットポリシーの権限評価のポイントまとめ

## 💡 前提の構成（今回の例）

- Lambda関数の実行ロール（LambdaThumbRole）には **ログ出力の権限のみ** が付与されている：
  ```json
  {
    "Effect": "Allow",
    "Action": [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents"
    ],
    "Resource": "*"
  }
  ```
S3アクセス権限（GetObject / PutObject）は一切付与されていない。
  ## 💡 それでもなぜ Lambda から S3 にアクセスできるのか？

  S3バケットに以下のような バケットポリシー（リソースポリシー） を追加しているため：

```json
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/LambdaThumbRole"
  },
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Effect": "Allow",
  "Resource": "arn:aws:s3:::my-image-bucket/*"
}
 ```    

- つまり、**リソース側でアクセスを許可しているので、IAMロール側でS3権限がなくてもアクセスできる**という構成。

---

## ✅ 結論：S3など一部のサービスでは、リソースポリシーだけでアクセスが許可される

| 状況                                           | 結果             |
|------------------------------------------------|------------------|
| IAMロールにS3権限なし、バケットポリシーが許可     | ✅ アクセス可能     |
| IAMロールがDenyしている                         | ❌ アクセス拒否     |
| バケットポリシーがPrincipalを許可していない       | ❌ アクセス拒否     |

---

## 🎯 ポイントまとめ

- LambdaのIAMロールには **S3のアクセス権限をあえて持たせない**（最小権限）
- S3リソース側（バケットポリシー）で **アクセス制御を集中管理**
- AWSでは「IAMポリシー ∩ リソースポリシー」で評価されるが、**S3のようなリソースポリシー対応サービスでは、リソース側のみの許可で動くケースがある**

---

## 🔐 セキュリティ的なメリット

- **意図しないS3バケットへのアクセスを防止**
- **リソース管理者側でアクセス制御を一元化できる**
- 監査・可視化・共有環境でのアクセスコントロールに向いている

---

## ✅ この構成は「リソース側集中管理型」の代表例です

Lambda関数から特定のS3バケットへアクセスさせたいとき、  
IAMロールにはログ権限だけを持たせ、S3側で明示的に許可することで、  
**より安全で明確なアクセス管理が実現できます。**

