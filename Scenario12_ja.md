<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [EC2 Auto Scaling × ログ長期保存設計（新しいリージョン）](#ec2-auto-scaling-%C3%97-%E3%83%AD%E3%82%B0%E9%95%B7%E6%9C%9F%E4%BF%9D%E5%AD%98%E8%A8%AD%E8%A8%88%E6%96%B0%E3%81%97%E3%81%84%E3%83%AA%E3%83%BC%E3%82%B8%E3%83%A7%E3%83%B3)
  - [📘 シナリオ](#-%E3%82%B7%E3%83%8A%E3%83%AA%E3%82%AA)
  - [🧠 設計観点](#-%E8%A8%AD%E8%A8%88%E8%A6%B3%E7%82%B9)
  - [🛠️ 構成要素と推奨実装](#-%E6%A7%8B%E6%88%90%E8%A6%81%E7%B4%A0%E3%81%A8%E6%8E%A8%E5%A5%A8%E5%AE%9F%E8%A3%85)
    - [1. EC2 インスタンスに CloudWatch エージェントを導入](#1-ec2-%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E3%81%AB-cloudwatch-%E3%82%A8%E3%83%BC%E3%82%B8%E3%82%A7%E3%83%B3%E3%83%88%E3%82%92%E5%B0%8E%E5%85%A5)
    - [2. CloudWatch Logs 側でロググループの保存期間を明示設定](#2-cloudwatch-logs-%E5%81%B4%E3%81%A7%E3%83%AD%E3%82%B0%E3%82%B0%E3%83%AB%E3%83%BC%E3%83%97%E3%81%AE%E4%BF%9D%E5%AD%98%E6%9C%9F%E9%96%93%E3%82%92%E6%98%8E%E7%A4%BA%E8%A8%AD%E5%AE%9A)
    - [3. IAM ロールを通じたログ転送権限の付与](#3-iam-%E3%83%AD%E3%83%BC%E3%83%AB%E3%82%92%E9%80%9A%E3%81%98%E3%81%9F%E3%83%AD%E3%82%B0%E8%BB%A2%E9%80%81%E6%A8%A9%E9%99%90%E3%81%AE%E4%BB%98%E4%B8%8E)
  - [🔐 セキュリティと運用の考慮点](#-%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3%E3%81%A8%E9%81%8B%E7%94%A8%E3%81%AE%E8%80%83%E6%85%AE%E7%82%B9)
  - [✅ 結論](#-%E7%B5%90%E8%AB%96)
  - [📊 Auto Scaling × ログ配信状況のモニタリング設計](#-auto-scaling-%C3%97-%E3%83%AD%E3%82%B0%E9%85%8D%E4%BF%A1%E7%8A%B6%E6%B3%81%E3%81%AE%E3%83%A2%E3%83%8B%E3%82%BF%E3%83%AA%E3%83%B3%E3%82%B0%E8%A8%AD%E8%A8%88)
    - [✅ モニタリング手法](#-%E3%83%A2%E3%83%8B%E3%82%BF%E3%83%AA%E3%83%B3%E3%82%B0%E6%89%8B%E6%B3%95)
      - [1. CloudWatch Logs Insights を用いたログ流入監視](#1-cloudwatch-logs-insights-%E3%82%92%E7%94%A8%E3%81%84%E3%81%9F%E3%83%AD%E3%82%B0%E6%B5%81%E5%85%A5%E7%9B%A3%E8%A6%96)
      - [2. CloudWatch Metric Filter によるログ配信検出](#2-cloudwatch-metric-filter-%E3%81%AB%E3%82%88%E3%82%8B%E3%83%AD%E3%82%B0%E9%85%8D%E4%BF%A1%E6%A4%9C%E5%87%BA)
      - [3. Auto Scaling ライフサイクルイベントのログ化](#3-auto-scaling-%E3%83%A9%E3%82%A4%E3%83%95%E3%82%B5%E3%82%A4%E3%82%AF%E3%83%AB%E3%82%A4%E3%83%99%E3%83%B3%E3%83%88%E3%81%AE%E3%83%AD%E3%82%B0%E5%8C%96)
      - [4. CloudWatch Alarm による通知設定](#4-cloudwatch-alarm-%E3%81%AB%E3%82%88%E3%82%8B%E9%80%9A%E7%9F%A5%E8%A8%AD%E5%AE%9A)
  - [🔁 運用補足](#-%E9%81%8B%E7%94%A8%E8%A3%9C%E8%B6%B3)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  EC2 Auto Scaling × ログ長期保存設計（新しいリージョン）

---

## 📘 シナリオ

グローバル企業が韓国（ap-northeast-2）に新たな拠点と AWS アカウントを設け、3つの Auto Scaling グループを活用して EC2 ベースのワークロードを展開している。  
本リージョンで稼働するすべてのシステム・アプリケーションログは「**7年間の保存**」が求められる。  
インスタンスのスケーリング（起動・終了）により、ログが失われない仕組みが必要である。

---

## 🧠 設計観点

| 要件 | 説明 |
|------|------|
| ✅ ログの永続保存 | インスタンスが終了してもログが消えないよう、外部ストレージに即時転送が必要 |
| ✅ 自動的な保存期間管理 | 7年間の保存期間が終了したログは、自動的に削除される仕組みを導入 |
| ✅ スケーラブルで一貫した仕組み | 全 Auto Scaling グループに適用可能なログ収集・転送・保存構成とする |

---

## 🛠️ 構成要素と推奨実装

### 1. EC2 インスタンスに CloudWatch エージェントを導入

- 起動時に CloudWatch Agent を自動インストール
- システムログおよびアプリケーションログを CloudWatch Logs に転送
- 各 Auto Scaling グループの起動テンプレートまたはユーザーデータで構成ファイルを注入

### 2. CloudWatch Logs 側でロググループの保存期間を明示設定

- ロググループ単位で **7年間（2555日）** の保持期間を設定
- 不要なログの蓄積を防ぎ、コンプライアンス要件に準拠

### 3. IAM ロールを通じたログ転送権限の付与

- Auto Scaling グループで起動する EC2 にアタッチする IAM ロールを定義
- ロールには CloudWatch Logs への書き込み権限（`logs:PutLogEvents` など）を含める

---

## 🔐 セキュリティと運用の考慮点

- ログ改ざん防止のため、CloudWatch Logs Insights や CloudTrail と組み合わせて分析可能性を確保
- Agent や構成ファイルのバージョン管理は S3 や Systems Manager パラメータストアで集中管理
- Auto Scaling イベントとログ連携状況は CloudWatch Metrics や Alarm でモニタリング

---

## ✅ 結論

この構成により、スケーリング環境下でもすべてのログを**確実に収集・転送・7年間保存**することが可能となり、地域・コンプライアンス要件を満たすシステムログ管理が実現できる。


---

## 📊 Auto Scaling × ログ配信状況のモニタリング設計

Auto Scaling によって動的に起動・終了される EC2 インスタンスが、**正しく CloudWatch Logs にログを配信できているかどうかを継続的に監視**することが重要です。

### ✅ モニタリング手法

#### 1. CloudWatch Logs Insights を用いたログ流入監視

- 各ロググループに対して、「一定時間内にログが到着しているか」をクエリで定期確認
- 例（直近30分以内にログが無いインスタンスを検出）：

```sql
fields @timestamp, @logStream
| filter @timestamp > ago(30m)
| stats count() by @logStream
```

- スケジューラー（Amazon EventBridge Scheduler）や Lambda を組み合わせて定期実行＆Slack通知も可

#### 2. CloudWatch Metric Filter によるログ配信検出

- 特定のログパターン（起動ログや “Log agent started”等）に対してメトリクスフィルタを定義
- 該当メトリクスが一定時間更新されない場合、CloudWatch Alarm を発報

#### 3. Auto Scaling ライフサイクルイベントのログ化

- Auto Scaling グループでライフサイクルフックを設定し、起動時の初期化ログ送信をトリガー
- フック通知（SNS → Lambda）で、初期ログ到達状況を記録・監視

#### 4. CloudWatch Alarm による通知設定

- 上記メトリクスに対して「ログが届かない状態が 15分以上続く場合」などの Alarm を作成
- 通知先は SNS（→メール・Slack）や Incident Manager に連携可能

---

## 🔁 運用補足

- メトリクスフィルタやログ監視設定は Infrastructure as Code（CloudFormation / Terraform）化を推奨
- スケーリング頻度が高い環境では、Auto Scaling の `GroupDesiredCapacity` とログストリーム数を比較してログ取りこぼしを検出
- 定期的に「ロググループのログストリーム数」と「Auto Scaling のインスタンス数」を照合する監視スクリプトも有効

---

