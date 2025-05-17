<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [クロスアカウント S3 アクセス設計ドキュメント（Account A → Account B）](#%E3%82%AF%E3%83%AD%E3%82%B9%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88-s3-%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E8%A8%AD%E8%A8%88%E3%83%89%E3%82%AD%E3%83%A5%E3%83%A1%E3%83%B3%E3%83%88account-a-%E2%86%92-account-b)
  - [📘 Scenario（シナリオ）](#-scenario%E3%82%B7%E3%83%8A%E3%83%AA%E3%82%AA)
  - [🧠 重要ポイント](#-%E9%87%8D%E8%A6%81%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88)
  - [✅ 解決策：バケットポリシーによるクロスアカウント許可](#-%E8%A7%A3%E6%B1%BA%E7%AD%96%E3%83%90%E3%82%B1%E3%83%83%E3%83%88%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E3%81%AB%E3%82%88%E3%82%8B%E3%82%AF%E3%83%AD%E3%82%B9%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E8%A8%B1%E5%8F%AF)
    - [Account B に設定するバケットポリシー（例）](#account-b-%E3%81%AB%E8%A8%AD%E5%AE%9A%E3%81%99%E3%82%8B%E3%83%90%E3%82%B1%E3%83%83%E3%83%88%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E4%BE%8B)
  - [📘 関連知識：クロスアカウントアクセスを成立させる 2 つの条件](#-%E9%96%A2%E9%80%A3%E7%9F%A5%E8%AD%98%E3%82%AF%E3%83%AD%E3%82%B9%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%82%92%E6%88%90%E7%AB%8B%E3%81%95%E3%81%9B%E3%82%8B-2-%E3%81%A4%E3%81%AE%E6%9D%A1%E4%BB%B6)
  - [🚫 他の方法とその限界](#-%E4%BB%96%E3%81%AE%E6%96%B9%E6%B3%95%E3%81%A8%E3%81%9D%E3%81%AE%E9%99%90%E7%95%8C)
  - [🛡️ ベストプラクティス：S3 クロスアカウント設計のポイント](#-%E3%83%99%E3%82%B9%E3%83%88%E3%83%97%E3%83%A9%E3%82%AF%E3%83%86%E3%82%A3%E3%82%B9s3-%E3%82%AF%E3%83%AD%E3%82%B9%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E8%A8%AD%E8%A8%88%E3%81%AE%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88)
  - [✅ 結論](#-%E7%B5%90%E8%AB%96)
  - [🧾 Account A 側の IAM ポリシー例（アクセス元ユーザーに付与）](#-account-a-%E5%81%B4%E3%81%AE-iam-%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E4%BE%8B%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E5%85%83%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E3%81%AB%E4%BB%98%E4%B8%8E)
  - [📝 補足：トラブルが発生したときの確認ポイント](#-%E8%A3%9C%E8%B6%B3%E3%83%88%E3%83%A9%E3%83%96%E3%83%AB%E3%81%8C%E7%99%BA%E7%94%9F%E3%81%97%E3%81%9F%E3%81%A8%E3%81%8D%E3%81%AE%E7%A2%BA%E8%AA%8D%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  クロスアカウント S3 アクセス設計ドキュメント（Account A → Account B）

---

## 📘 Scenario（シナリオ）

- 企業 A（Account A）は、企業 B（Account B）を買収。
- Account B の S3 バケットに保存されたファイルに対して、Account A のユーザーにフルアクセスを与えたい。
- Account A 側の IAM ポリシーは設定済みだが、アクセスは拒否されている。

---

## 🧠 重要ポイント

| 項目 | 内容 |
|------|------|
| ✅ IAM ポリシーのみでは不十分 | IAM は「誰に何を許可するか」を定義するが、S3 バケット側の「受け入れ設定」が必要 |
| ✅ S3 のリソースベースポリシー | S3 バケット側で「どの外部アカウントのどのユーザーにアクセスを許可するか」を明示する必要がある |
| ✅ Principal と Resource の両方を明示 | 相互の ARN を明記し、双方向に権限が成立するように構成 |
| ❌ ACL は推奨されない | ACL（アクセスコントロールリスト）は細かい制御が難しく、現在は非推奨機能 |
| ❌ Account B 内の IAM ユーザーに対するポリシーは意味がない | Account A のユーザーには作用しない |

---

## ✅ 解決策：バケットポリシーによるクロスアカウント許可

### Account B に設定するバケットポリシー（例）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:user/TargetUser"
      },
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::example-bucket",
        "arn:aws:s3:::example-bucket/*"
      ]
    }
  ]
}
```

- `Principal`: アクセスを許可したいユーザーの ARN（Account A 側）
- `Resource`: 対象の S3 バケットとその中のオブジェクトすべて
- `Action`: どの操作を許可するか（例：`s3:GetObject`, `s3:PutObject`, `s3:*` など）

---

## 📘 関連知識：クロスアカウントアクセスを成立させる 2 つの条件

| 要件 | 説明 |
|------|------|
| ① アクセス元ユーザーの IAM ポリシー | Account A のユーザーに `s3:*` など適切な操作を許可 |
| ② バケット側のリソースベースポリシー | Account B の S3 バケットに外部ユーザーのアクセスを明示的に許可するバケットポリシー |

---

## 🚫 他の方法とその限界

| 方法 | 問題点 |
|------|--------|
| バケット ACL | 操作粒度が粗く、ユーザー指定が困難。監査性・管理性が低いため非推奨。 |
| オブジェクト ACL | 各オブジェクトごとに設定が必要。スケールしない。 |
| IAM ポリシー（Account B 側） | Account B の IAM ユーザーやロールにしか適用できないため、Account A のユーザーには効果がない。 |

---

## 🛡️ ベストプラクティス：S3 クロスアカウント設計のポイント

- **Resourceベースのバケットポリシーを使う**
- **ARN（アカウント番号 + ユーザー/ロール名）を明示的に書く**
- **最小権限の原則を守る（例：s3:GetObject のみ許可など）**
- **ログ記録の有効化（S3 Access Logs や CloudTrail）**

---

## ✅ 結論

- IAM ユーザー間のクロスアカウントアクセスには、**アクセス元の IAM ポリシーと、S3 バケット側のバケットポリシーの両方が必要**
- このような設計では、**Account B 側のバケットポリシーで明示的に許可を与えることが必須**
- よって、**正しい対応策は「選択肢 C：バケットポリシーの追加」**である

---


---

## 🧾 Account A 側の IAM ポリシー例（アクセス元ユーザーに付与）

Account A の IAM ユーザー（例: TargetUser）には、以下のような IAM ポリシーをアタッチする必要があります：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::example-bucket",
        "arn:aws:s3:::example-bucket/*"
      ]
    }
  ]
}
```

- `Action`：必要に応じて `s3:GetObject`, `s3:PutObject` などに制限可能
- `Resource`：Account B の S3 バケットの ARN を指定
- ※ Account A 側でこのポリシーが適用されていても、**Account B 側のバケットポリシーがないとアクセスは拒否される**点に注意

---

## 📝 補足：トラブルが発生したときの確認ポイント

1. バケットポリシーに指定された `Principal` が正しいか（ARN ミス）
2. IAM ポリシーとバケットポリシーの両方で必要な `Action` が許可されているか
3. S3 バケットに「バケットオーナー強制設定（ACL 無効化）」が有効になっていないか
4. アクセス拒否エラー発生時は IAM Policy Simulator や S3 アクセスアナライザーで検証

---
