# AWS CloudFormation StackSets × セキュリティ通知設計ドキュメント（日本語）

---

## 📘 Scenario（シナリオ）

ある企業は **AWS Organizations** を利用し、**CloudFormation StackSets** を使って組織内の各 AWS アカウントに EC2、ELB、RDS、EKS/ECS を含む設計パターンを展開しようとしています。  
開発者は現在、個別に CloudFormation スタックを作成することで高速なデリバリーを実現しています。  
デプロイは共有サービスアカウント内の中央 CI/CD パイプラインによって実行されます。  
セキュリティチームは、各リソースに関する内部セキュリティ基準を定めており、**その基準に準拠していないリソースがあった場合に通知を受ける必要があります**。  
ただし、**開発速度は維持する必要があります**。

---

## 🧠 重要ポイント

| 項目 | 説明 |
|------|------|
| 📦 **StackSets + Organizations** | 複数アカウントに標準設計を一括展開。 |
| ⚡ **開発速度の維持** | 各開発者がテンプレートを個別に迅速に展開している現状を維持する必要がある。 |
| 🔐 **セキュリティ要件の遵守** | 全リソースは既定のセキュリティポリシーに従う必要がある。 |
| 📨 **違反通知の仕組み** | 違反時にはセキュリティチームに速やかに通知されること。 |
| 🎯 **目標** | 最小の運用負荷でセキュリティ検証と通知を実現すること。 |


---

## 🛠️ 実装例

### CloudFormation Guard ルールの例（例: EC2 インスタンスと S3 バケットの検証）

```hcl
rule ec2_instance_type when resourceType == "AWS::EC2::Instance" {
  properties.instanceType == "t3.micro"
}

rule s3_bucket_encryption when resourceType == "AWS::S3::Bucket" {
  properties.BucketEncryption != null
}
```

### CI/CD 内での実行例（Docker を利用）

```bash
docker run --rm   -v $(pwd)/template:/template   -v $(pwd)/rules:/rules   ghcr.io/aws-cloudformation/cloudformation-guard   cfn-guard validate -d /template/template.yaml -r /rules
```

### 検証失敗時の SNS 通知例（CodePipeline）

```json
{
  "Action": "Publish",
  "Resource": "arn:aws:sns:ap-northeast-1:123456789012:SecurityNotifications",
  "Condition": {
    "StringEquals": {
      "cfn-guard:Result": "FAILED"
    }
  }
}
```

---

## 🔐 セキュリティチェック

| 対策項目 | 説明 |
|----------|------|
| ✅ 自動化されたテンプレート検証 | CloudFormation Guard により、すべてのテンプレートを CI/CD 内でポリシー準拠チェック |
| ✅ 構文と依存関係の検証 | `cfn-lint` による事前チェック |
| ✅ 違反発見時の即時通知 | SNS 経由でセキュリティチームに違反イベントを通知 |
| ✅ デプロイ制限 | Guard で PASS しないテンプレートは本番適用不可にする |
| ✅ 再現性と整合性 | Docker 実行で誰がどこで検証しても同じ結果を得られる構成 |



## 🚀 推奨ソリューション: CloudFormation Guard の活用

### CloudFormation Guard（cfn-guard）とは？

**CloudFormation Guard** は、CloudFormation テンプレートが**企業のポリシーに準拠しているかどうかを検証**するためのポリシー・アズ・コードツールです。

| 特徴 | 説明 |
|------|------|
| 🔍 DSLベースのポリシー記述 | 例：`properties.instanceType == "t3.micro"` |
| 🔁 再利用性 | 一度書けば複数テンプレートに適用可能 |
| ⚙️ CI/CD 統合性 | Docker や CLI で容易に統合できる |
| 🔐 セキュリティのシフトレフト | デプロイ前に検出して未然に防止 |

---

### 代表的な活用例

| シナリオ | ポリシー例 |
|----------|-----------|
| ✅ 暗号化の強制 | S3 バケット、RDS、EBS などで暗号化を必須に |
| ✅ インスタンスタイプ制限 | 許可されたタイプのみ許可（例：`t3.*`） |
| ✅ パブリックアクセスの制限 | `0.0.0.0/0` を拒否するポリシー適用 |
| ✅ IAM ポリシーの検査 | `Action: "*"` を禁止など過剰な権限を検出 |

---

### ポリシー例

```hcl
rule ec2_instance_type when resourceType == "AWS::EC2::Instance" {
  properties.instanceType == "t3.micro"
}

rule s3_encryption when resourceType == "AWS::S3::Bucket" {
  properties.BucketEncryption != null
}
```

---

## 📦 CloudFormation Stack とは？

CloudFormation Stack とは、テンプレートに基づいてデプロイされる AWS リソースの集合体です。

### 利用タイミング

| シナリオ | 説明 |
|----------|------|
| 複数リソース構成 | EC2 + RDS + ELB などを一括管理・展開 |
| インフラの再現性 | テンプレート化 + Git管理で誰でも同じ構成を再現可能 |
| マルチアカウント管理 | StackSets により組織全体で統一インフラ展開が可能 |

---

## 🛡️ 事前防止策（シフトレフトセキュリティ）

| 対策 | 具体例 |
|------|--------|
| 🧾 ルール定義 | Guard で暗号化、タグ付け、公開範囲制限などを強制 |
| 🔄 Lint チェック | `cfn-lint` で構文・依存関係エラーの早期検知 |
| 🛂 IAM 制御 | 本番デプロイには Guard 検査済みテンプレートのみ許可 |
| 🏷️ タグ強制 | `Owner`, `Environment`, `CostCenter` の付与を必須に |

---

## ✅ インフラ・アズ・コード運用のベストプラクティス

| ベストプラクティス | 解説 |
|---------------------|------|
| パラメータ化の徹底 | サブネットや AMI を動的に扱う |
| ネストスタック | 大規模構成は部品化して管理性向上 |
| 変更セット使用 | 適用前に差分確認でミス防止 |
| Git 管理 & PR 実施 | 全変更にレビューと承認を挟む |
| 暗号化の標準化 | Guard で強制ポリシーとして適用 |
| ワイルドカード禁止 | IAM での `"*"` 使用は原則 NG |

---

## 🔔 結論

CloudFormation Guard を CI/CD に組み込み、SNS によってセキュリティチームに通知することで：

- ✅ 各テンプレートの自動検査
- ✅ 開発スピードの維持
- ✅ 違反の即時可視化
- ✅ スケーラブルなセキュリティ統制

