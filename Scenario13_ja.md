
#  EC2 Auto Scaling × ログ長期保存設計（韓国リージョン）

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

