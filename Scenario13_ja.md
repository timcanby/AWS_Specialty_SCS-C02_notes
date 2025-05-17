
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
