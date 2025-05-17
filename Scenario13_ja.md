<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [IAM MFA制限とセッション時間制御の混同ポイントまとめ](#iam-mfa%E5%88%B6%E9%99%90%E3%81%A8%E3%82%BB%E3%83%83%E3%82%B7%E3%83%A7%E3%83%B3%E6%99%82%E9%96%93%E5%88%B6%E5%BE%A1%E3%81%AE%E6%B7%B7%E5%90%8C%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88%E3%81%BE%E3%81%A8%E3%82%81)
  - [🔄 よくある混同点と補足解説：`aws:MultiFactorAuthAge` と `MaxSessionDuration`](#-%E3%82%88%E3%81%8F%E3%81%82%E3%82%8B%E6%B7%B7%E5%90%8C%E7%82%B9%E3%81%A8%E8%A3%9C%E8%B6%B3%E8%A7%A3%E8%AA%ACawsmultifactorauthage-%E3%81%A8-maxsessionduration)
    - [🔐 `aws:MultiFactorAuthAge`](#-awsmultifactorauthage)
    - [🕒 `MaxSessionDuration`](#-maxsessionduration)
    - [🧩 よくある誤解の例と整理](#-%E3%82%88%E3%81%8F%E3%81%82%E3%82%8B%E8%AA%A4%E8%A7%A3%E3%81%AE%E4%BE%8B%E3%81%A8%E6%95%B4%E7%90%86)
  - [🔎 補足：他の関連する時間条件（参考）](#-%E8%A3%9C%E8%B6%B3%E4%BB%96%E3%81%AE%E9%96%A2%E9%80%A3%E3%81%99%E3%82%8B%E6%99%82%E9%96%93%E6%9D%A1%E4%BB%B6%E5%8F%82%E8%80%83)
  - [✅ 結論：時間制御のキーは目的によって正しく使い分ける](#-%E7%B5%90%E8%AB%96%E6%99%82%E9%96%93%E5%88%B6%E5%BE%A1%E3%81%AE%E3%82%AD%E3%83%BC%E3%81%AF%E7%9B%AE%E7%9A%84%E3%81%AB%E3%82%88%E3%81%A3%E3%81%A6%E6%AD%A3%E3%81%97%E3%81%8F%E4%BD%BF%E3%81%84%E5%88%86%E3%81%91%E3%82%8B)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


#  IAM MFA制限とセッション時間制御の混同ポイントまとめ

---

## 🔄 よくある混同点と補足解説：`aws:MultiFactorAuthAge` と `MaxSessionDuration`

MFA とセッションの制限に関する設定には、**似たような名前で異なる意味を持つ項目**がいくつか存在し、混乱を招きやすいです。以下にそれぞれの違いを整理します。

### 🔐 `aws:MultiFactorAuthAge`

| 項目 | 内容 |
|------|------|
| 種類 | IAM ポリシーの `Condition` で使用するキー |
| 単位 | 秒（例：7200秒 = 2時間） |
| 意味 | MFA が認証された時点から現在までの経過時間を表す |
| 使い方 | `NumericLessThan: { "aws:MultiFactorAuthAge": 7200 }` のように使い、「MFA 認証から 2 時間以内」でなければ拒否可能 |
| 対象 | **MFA を使ってセッションを開始した IAM ユーザー** のみ |

### 🕒 `MaxSessionDuration`

| 項目 | 内容 |
|------|------|
| 種類 | IAM ロールの設定値（ポリシー条件では使用不可） |
| 単位 | 秒（設定時は 3600〜43200 秒の範囲） |
| 意味 | AssumeRole によって生成されるセッションの最大持続時間 |
| 使い方 | ロールそのものに対して `MaxSessionDuration` を設定（IAM ポリシーの `Condition` には使えない） |
| 対象 | **IAM ロールに AssumeRole でアクセスするセッション**に適用される

---

### 🧩 よくある誤解の例と整理

| 混同されやすい項目 | 正しい意味と注意点 |
|------------------|------------------|
| `MaxSessionDuration` を IAM ポリシーの `Condition` に書く | ❌ エラーになる。これはロール設定値であり、ポリシーの条件に使えない。 |
| `aws:MultiFactorAuthAge` は IAM ロールに適用できる？ | ⚠️ 原則として IAM ユーザーに適用される。AssumeRole では MFA コンテキストが引き継がれない場合もあるため注意。 |
| `aws:TokenIssueTime` の使用 | ✅ 代替的に MFA 認証直後の時刻として使えるが、明示的に `MultiFactorAuthAge` を使う方が簡潔 |

---

## 🔎 補足：他の関連する時間条件（参考）

| キー名 | 意味 | 備考 |
|--------|------|------|
| `aws:TokenIssueTime` | セッションが発行された時刻 | `MultiFactorAuthAge` が使えない場合の代替条件に使われることがある |
| `aws:CurrentTime` | 現在の時刻（UTC） | 固定された期間の間だけ許可したい場合に使う |
| `aws:EpochTime` | 現在時刻のUNIXタイムスタンプ | カスタムな時間条件を計算式で使いたいときに便利 |

---

## ✅ 結論：時間制御のキーは目的によって正しく使い分ける

- ✅ **MFA の制限（「MFA後2時間以内」など）→ `aws:MultiFactorAuthAge`**
- ✅ **ロールセッションの制限 → ロールの `MaxSessionDuration` を設定**
- ✅ IAM ポリシーの `Condition` に書けるのは `aws:MultiFactorAuthAge` 等のみ
