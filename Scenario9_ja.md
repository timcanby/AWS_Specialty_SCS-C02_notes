<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [GuardDuty 「Impact:IAMUser/AnomalousBehavior」インシデント初動調査設計ドキュメント](#guardduty-impactiamuseranomalousbehavior%E3%82%A4%E3%83%B3%E3%82%B7%E3%83%87%E3%83%B3%E3%83%88%E5%88%9D%E5%8B%95%E8%AA%BF%E6%9F%BB%E8%A8%AD%E8%A8%88%E3%83%89%E3%82%AD%E3%83%A5%E3%83%A1%E3%83%B3%E3%83%88)
  - [📘 Scenario（シナリオ）](#-scenario%E3%82%B7%E3%83%8A%E3%83%AA%E3%82%AA)
  - [🧠 重要ポイント](#-%E9%87%8D%E8%A6%81%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88)
  - [✅ 最適なアプローチ](#-%E6%9C%80%E9%81%A9%E3%81%AA%E3%82%A2%E3%83%97%E3%83%AD%E3%83%BC%E3%83%81)
    - [B. 読み取り専用（ReadOnly）で GuardDuty の Finding を確認し、Amazon Detective で API コールの文脈を分析する](#b-%E8%AA%AD%E3%81%BF%E5%8F%96%E3%82%8A%E5%B0%82%E7%94%A8readonly%E3%81%A7-guardduty-%E3%81%AE-finding-%E3%82%92%E7%A2%BA%E8%AA%8D%E3%81%97amazon-detective-%E3%81%A7-api-%E3%82%B3%E3%83%BC%E3%83%AB%E3%81%AE%E6%96%87%E8%84%88%E3%82%92%E5%88%86%E6%9E%90%E3%81%99%E3%82%8B)
  - [🚫 他の選択肢の問題点](#-%E4%BB%96%E3%81%AE%E9%81%B8%E6%8A%9E%E8%82%A2%E3%81%AE%E5%95%8F%E9%A1%8C%E7%82%B9)
  - [📌 結論](#-%E7%B5%90%E8%AB%96)
  - [🔎 Amazon Detective と AWS CloudTrail Insights の比較と使い分け](#-amazon-detective-%E3%81%A8-aws-cloudtrail-insights-%E3%81%AE%E6%AF%94%E8%BC%83%E3%81%A8%E4%BD%BF%E3%81%84%E5%88%86%E3%81%91)
    - [🧭 Amazon Detective の役割とユースケース](#-amazon-detective-%E3%81%AE%E5%BD%B9%E5%89%B2%E3%81%A8%E3%83%A6%E3%83%BC%E3%82%B9%E3%82%B1%E3%83%BC%E3%82%B9)
      - [🔍 主な機能：](#-%E4%B8%BB%E3%81%AA%E6%A9%9F%E8%83%BD)
      - [💡 適した場面：](#-%E9%81%A9%E3%81%97%E3%81%9F%E5%A0%B4%E9%9D%A2)
    - [📘 AWS CloudTrail Insights & CloudTrail Lake の役割とユースケース](#-aws-cloudtrail-insights--cloudtrail-lake-%E3%81%AE%E5%BD%B9%E5%89%B2%E3%81%A8%E3%83%A6%E3%83%BC%E3%82%B9%E3%82%B1%E3%83%BC%E3%82%B9)
      - [🔍 主な機能：](#-%E4%B8%BB%E3%81%AA%E6%A9%9F%E8%83%BD-1)
      - [💡 適した場面：](#-%E9%81%A9%E3%81%97%E3%81%9F%E5%A0%B4%E9%9D%A2-1)
  - [✅ まとめ：どちらを使うべきか？](#-%E3%81%BE%E3%81%A8%E3%82%81%E3%81%A9%E3%81%A1%E3%82%89%E3%82%92%E4%BD%BF%E3%81%86%E3%81%B9%E3%81%8D%E3%81%8B)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  GuardDuty 「Impact:IAMUser/AnomalousBehavior」インシデント初動調査設計ドキュメント

---

## 📘 Scenario（シナリオ）

ある企業の AWS アカウントで、本番アプリケーションをホストしています。  
Amazon GuardDuty より以下の検出通知が届きました：

> **Impact:IAMUser/AnomalousBehavior**（IAMユーザーによる異常な動作）

セキュリティエンジニアは、**本番環境に影響を与えることなく、迅速に調査を開始**し、攻撃の兆候を確認・分析する必要があります。

---

## 🧠 重要ポイント

| 項目 | 内容 |
|------|------|
| 🔍 検出内容 | 通常と異なる API 呼び出しやリージョンへのアクセスなど、IAM ユーザーの不審な動作 |
| ✅ アプリ影響の回避 | 本番システムを停止させずに調査を行う必要がある |
| 🔐 最小権限での調査 | 調査は ReadOnly 権限で実施すべき |
| 📊 文脈付き分析 | API ログだけでなく、「なぜ・どのように異常か」を理解する必要がある |
| ⏱️ スピード | もっとも迅速にインシデント対応を始められる方法が求められる |

---

## ✅ 最適なアプローチ

### B. 読み取り専用（ReadOnly）で GuardDuty の Finding を確認し、Amazon Detective で API コールの文脈を分析する

| 理由 | 説明 |
|------|------|
| ✅ 調査中にアカウントへ影響を与えない | ReadOnly IAM ユーザーで安全に確認可能 |
| ✅ Amazon Detective の活用 | GuardDuty の Finding と連携し、詳細な行動分析が可能 |
| ✅ API 呼び出しの相関可視化 | 攻撃者の行動や影響範囲が時系列や依存関係で見える |

---

## 🚫 他の選択肢の問題点

| 選択肢 | 問題点 |
|--------|--------|
| A. IAM に DenyAll ポリシー付与 | 調査前に通信を遮断すると誤検知だった場合に正しい動作まで止める恐れがある |
| C. 管理者として即時遮断 | 影響の大きい操作となるため、事前調査なしではリスクが高い |
| D. CloudTrail Insights と Lake を使用 | 有効だが事前準備が必要で、Detective に比べ即時性に欠ける |

---

## 📌 結論

- ✅ **Amazon Detective を用いることで、即時に・影響を与えずに・深い調査が可能**
- ✅ ReadOnly アクセスで開始でき、GuardDuty との統合により操作もスムーズ
- ✅ 今後の封じ込めや修復アクションの前に、正確な判断材料を得るのに最適な方法


---

## 🔎 Amazon Detective と AWS CloudTrail Insights の比較と使い分け

本件のようなセキュリティインシデント調査においては、**異常行動の原因追跡と全体像の把握**が重要です。  
ここでは、それを支援する 2 つの代表的な AWS サービスの役割と活用シーンを整理します。

---

### 🧭 Amazon Detective の役割とユースケース

**Amazon Detective** は、GuardDuty・CloudTrail・VPC Flow Logs などのデータを自動的に関連付けて、**疑わしい動作の前後関係・影響範囲を可視化**するサービスです。

#### 🔍 主な機能：

- アクティビティタイムライン（時系列）
- 関連ユーザー / IP / インスタンスとの相関分析
- 自動グラフ生成による調査効率の向上

#### 💡 適した場面：

- GuardDuty による異常検出後の初動調査
- 「この IAM ユーザーは他に何をしていたか？」を文脈で把握したいとき
- 誰が、いつ、どの IP から、何をしたかを横断的に見たいとき

---

### 📘 AWS CloudTrail Insights & CloudTrail Lake の役割とユースケース

**AWS CloudTrail Insights** は、CloudTrail の通常パターンからの逸脱を検出するサービスです。  
**CloudTrail Lake** は、CloudTrail データを長期間蓄積・分析できるクエリベースのデータレイクです。

#### 🔍 主な機能：

- API 呼び出しの頻度や量の異常を Insights で自動検出
- SQL ライクなクエリで CloudTrail ログを時系列で検索・集計（Lake）

#### 💡 適した場面：

- GuardDuty に検出されていない微細な傾向を分析したいとき
- 管理者操作や設定変更のトレンドを把握したいとき
- 長期的な API 利用パターンや統計的分析が必要なとき

---

## ✅ まとめ：どちらを使うべきか？

| 調査目的 | 適したサービス |
|-----------|----------------|
| GuardDuty の finding をすぐに分析 | ✅ Amazon Detective |
| 通常と異なるアクティビティの自動検出 | ✅ CloudTrail Insights |
| クエリベースで詳細にログ分析したい | ✅ CloudTrail Lake |
| リージョン・アカウントをまたぐ文脈の把握 | ✅ Amazon Detective |

---

