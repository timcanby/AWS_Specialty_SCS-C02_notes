<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [AWS Security Hub クリティカル検出通知メール送信設計ドキュメント](#aws-security-hub-%E3%82%AF%E3%83%AA%E3%83%86%E3%82%A3%E3%82%AB%E3%83%AB%E6%A4%9C%E5%87%BA%E9%80%9A%E7%9F%A5%E3%83%A1%E3%83%BC%E3%83%AB%E9%80%81%E4%BF%A1%E8%A8%AD%E8%A8%88%E3%83%89%E3%82%AD%E3%83%A5%E3%83%A1%E3%83%B3%E3%83%88)
  - [📘 Scenario（シナリオ）](#-scenario%E3%82%B7%E3%83%8A%E3%83%AA%E3%82%AA)
  - [🧠 重要ポイント](#-%E9%87%8D%E8%A6%81%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88)
  - [✅ 最適な構成（正解）](#-%E6%9C%80%E9%81%A9%E3%81%AA%E6%A7%8B%E6%88%90%E6%AD%A3%E8%A7%A3)
    - [C. EventBridge ルールでクリティカル検出を監視し、SNS でメール送信する](#c-eventbridge-%E3%83%AB%E3%83%BC%E3%83%AB%E3%81%A7%E3%82%AF%E3%83%AA%E3%83%86%E3%82%A3%E3%82%AB%E3%83%AB%E6%A4%9C%E5%87%BA%E3%82%92%E7%9B%A3%E8%A6%96%E3%81%97sns-%E3%81%A7%E3%83%A1%E3%83%BC%E3%83%AB%E9%80%81%E4%BF%A1%E3%81%99%E3%82%8B)
    - [🎯 EventBridge フィルター例：](#-eventbridge-%E3%83%95%E3%82%A3%E3%83%AB%E3%82%BF%E3%83%BC%E4%BE%8B)
  - [🚫 他の選択肢の問題点](#-%E4%BB%96%E3%81%AE%E9%81%B8%E6%8A%9E%E8%82%A2%E3%81%AE%E5%95%8F%E9%A1%8C%E7%82%B9)
  - [📌 結論](#-%E7%B5%90%E8%AB%96)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

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
