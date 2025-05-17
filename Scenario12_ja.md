#  AWS Security Hub クリティカル検出通知メール送信設計ドキュメント

---

## 📘 Scenario（シナリオ）

ある企業が、**AWS Security Hub によって検出された「重大（CRITICAL）」なセキュリティファインディングに関する通知メール**を受け取りたいと考えています。  
ただし、現在はそのためのインフラや仕組みが未整備です。

---

## 🧠 重要ポイント

| 項目 | 説明 |
|------|------|
| ✅ Security Hub は EventBridge と連携可能 | Security Hub の検出結果（Findings）は EventBridge 経由でリアルタイムに転送可能 |
| ✅ SNS はメール通知に最適な AWS サービス | Amazon SNS を使えば、メールアドレスを直接購読者として設定可能 |
| ✅ コストと構成のシンプルさが重要 | 要件は「メール通知」のみなので、複雑な構成やカスタム処理は不要 |

---

## ✅ 最適な構成（正解）

### C. EventBridge ルールでクリティカル検出を監視し、SNS でメール送信する

| 手順 | 説明 |
|------|------|
| ① | EventBridge ルールを作成し、`SeverityLabel = "CRITICAL"` な Security Hub Findings をフィルター |
| ② | ルールのターゲットとして Amazon SNS トピックを指定 |
| ③ | SNS トピックに通知用メールアドレスを購読（Subscribe）設定 |
| ④ | メールで通知を受信し、初回は確認メールで購読を承認 |

---

### 🎯 EventBridge フィルター例：

```json
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "Severity": {
        "Label": ["CRITICAL"]
      }
    }
  }
}
```

---

## 🚫 他の選択肢の問題点

| 選択肢 | 問題点 |
|--------|--------|
| A. Lambda → SNS | Lambda 関数を使うと構成が複雑になる（EventBridge で直接 SNS を呼び出せるため不要） |
| B. Kinesis Data Firehose | Firehose はログストリーム向けであり、メール送信には適さない |
| D. Amazon SES | SES はメール送信に使えるが、セットアップやドメイン検証が必要で冗長。SNS の方が即時利用可能 |

---

## 📌 結論

- ✅ 最もシンプルかつ実用的な構成は：  
　**EventBridge ルール（CRITICAL を検出） → SNS → メール通知**
- ✅ 追加の開発や運用コストが不要で、Security Hub のネイティブ連携にも対応
