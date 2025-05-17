<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [AWS Organizations × マルチアカウントセキュリティ監視・自動修復設計ドキュメント](#aws-organizations-%C3%97-%E3%83%9E%E3%83%AB%E3%83%81%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3%E7%9B%A3%E8%A6%96%E3%83%BB%E8%87%AA%E5%8B%95%E4%BF%AE%E5%BE%A9%E8%A8%AD%E8%A8%88%E3%83%89%E3%82%AD%E3%83%A5%E3%83%A1%E3%83%B3%E3%83%88)
  - [📘 シナリオ](#-%E3%82%B7%E3%83%8A%E3%83%AA%E3%82%AA)
  - [🧠 設計要件と検討軸](#-%E8%A8%AD%E8%A8%88%E8%A6%81%E4%BB%B6%E3%81%A8%E6%A4%9C%E8%A8%8E%E8%BB%B8)
  - [✅ 最適な構成](#-%E6%9C%80%E9%81%A9%E3%81%AA%E6%A7%8B%E6%88%90)
    - [**D. AWS Security Hub + EventBridge + Lambda + SNS の構成**](#d-aws-security-hub--eventbridge--lambda--sns-%E3%81%AE%E6%A7%8B%E6%88%90)
  - [🔍 GuardDuty 単独構成の限界と盲点](#-guardduty-%E5%8D%98%E7%8B%AC%E6%A7%8B%E6%88%90%E3%81%AE%E9%99%90%E7%95%8C%E3%81%A8%E7%9B%B2%E7%82%B9)
    - [❗ GuardDuty の主な盲点と補足](#-guardduty-%E3%81%AE%E4%B8%BB%E3%81%AA%E7%9B%B2%E7%82%B9%E3%81%A8%E8%A3%9C%E8%B6%B3)
  - [🧭 なぜ Security Hub との統合が必要か](#-%E3%81%AA%E3%81%9C-security-hub-%E3%81%A8%E3%81%AE%E7%B5%B1%E5%90%88%E3%81%8C%E5%BF%85%E8%A6%81%E3%81%8B)
  - [✅ 推奨構成：GuardDuty + Security Hub + EventBridge + Lambda + SNS](#-%E6%8E%A8%E5%A5%A8%E6%A7%8B%E6%88%90guardduty--security-hub--eventbridge--lambda--sns)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  AWS Organizations × マルチアカウントセキュリティ監視・自動修復設計ドキュメント

---

## 📘 シナリオ

企業は AWS Organizations を使用し、複数の AWS アカウントに分散された本番ワークロードを運用している。  
セキュリティエンジニアは、以下のようなセキュリティ監視基盤を設計する必要がある：

- 🔍 **すべての本番アカウントでの不審な動作を事前に検知（プロアクティブ監視）**
- ⚙️ **インシデントが発生した場合に自動で修復を実行**
- 📩 **重大なセキュリティファインディング発生時に SNS 経由で通知**
- 📁 **すべてのセキュリティログを専用のログ収集アカウントに送信**

---

## 🧠 設計要件と検討軸

| 要件 | 説明 |
|------|------|
| ✅ 複数アカウント対応 | AWS Organizations による一元管理・連携 |
| ✅ 統合ログ管理 | 専用ログ収集アカウントにセキュリティデータを集約 |
| ✅ 自動修復 | Lambda や Systems Manager を使ったインシデント対応 |
| ✅ 重大度の高い検出に対する通知 | SNS による即時通知と外部システム連携可能性 |
| ✅ 拡張性 | アカウント追加や検出ロジック変更に柔軟に対応可能であること |

---

## ✅ 最適な構成

### **D. AWS Security Hub + EventBridge + Lambda + SNS の構成**

| 要素 | 説明 |
|------|------|
| 🔒 **Security Hub** | 各本番アカウントに有効化。脅威の統合検出・管理を実現。 |
| 🔁 **集約アカウントでの統合** | セキュリティファインディングを集約用アカウントに統合管理 |
| ⚡ **EventBridge + Lambda** | Security Hub の検出イベントをトリガーに Lambda を実行し自動修復 |
| 🔔 **SNS 通知** | Lambda 内で SNS に通知を送信し、即時にセキュリティ担当者へアラート |

---

## 🔍 GuardDuty 単独構成の限界と盲点

GuardDuty は非常に強力な脅威検出サービスですが、**対応できない領域や盲点**が存在するため、それ単独ではセキュリティ監視が不十分となるケースがあります。

### ❗ GuardDuty の主な盲点と補足

| 分類 | 説明 |
|------|------|
| IAM 設定ミスの検出 | GuardDuty は IAM ポリシーの過剰許可や誤設定などを直接検出できない |
| S3 パブリック設定 | バケットがパブリックになっている場合も、GuardDuty は通知しない（Security Hub + S3 保護ルールが必要） |
| OS レベルの異常 | GuardDuty は OS 内部の不審なプロセスやログイン履歴などは検出できない（Inspector や CloudWatch Logs が補完） |
| 長期的傾向の異常検出 | GuardDuty は主にリアルタイムな振る舞いを検知するが、行動傾向やポリシー継続的な逸脱には弱い（Security Hub や Config と連携必要） |
| アプリケーション層攻撃 | HTTP レイヤーの脆弱性攻撃などは GuardDuty では検知できない（WAF や Inspector Advanced の活用が必要） |

---

## 🧭 なぜ Security Hub との統合が必要か

Security Hub は以下のような役割で GuardDuty を補完・拡張します：

- ✅ **Security Best Practices の定期評価（CIS など）**
- ✅ **S3 / IAM / EBS / VPC に関する構成リスク検出**
- ✅ **AWS Config ルールや Inspector、Macie など他サービスとの統合**
- ✅ **重大度別の一元ビューと EventBridge アクション連携**

---

## ✅ 推奨構成：GuardDuty + Security Hub + EventBridge + Lambda + SNS

このような複合構成により、**リアルタイム検知（GuardDuty）と構成監査・リスクスコアリング（Security Hub）を統合**し、マルチアカウント環境におけるセキュリティギャップを最小限に抑えることができます。

