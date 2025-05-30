<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [AWS EC2 × SSM：セキュリティインシデント対応プロセスの安全設計（メモリ保持と隔離）](#aws-ec2-%C3%97-ssm%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3%E3%82%A4%E3%83%B3%E3%82%B7%E3%83%87%E3%83%B3%E3%83%88%E5%AF%BE%E5%BF%9C%E3%83%97%E3%83%AD%E3%82%BB%E3%82%B9%E3%81%AE%E5%AE%89%E5%85%A8%E8%A8%AD%E8%A8%88%E3%83%A1%E3%83%A2%E3%83%AA%E4%BF%9D%E6%8C%81%E3%81%A8%E9%9A%94%E9%9B%A2)
  - [📘 Scenario（シナリオ）](#-scenario%E3%82%B7%E3%83%8A%E3%83%AA%E3%82%AA)
  - [🔎 Investigation Workflow（調査フローの全体像）](#-investigation-workflow%E8%AA%BF%E6%9F%BB%E3%83%95%E3%83%AD%E3%83%BC%E3%81%AE%E5%85%A8%E4%BD%93%E5%83%8F)
  - [🛡️ Isolation Strategy（隔離戦略）](#-isolation-strategy%E9%9A%94%E9%9B%A2%E6%88%A6%E7%95%A5)
    - [Termination Protection とは？](#termination-protection-%E3%81%A8%E3%81%AF)
  - [🔧 Security Group 更新ポイント](#-security-group-%E6%9B%B4%E6%96%B0%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88)
  - [🚀 Systems Manager Run Command で揮発性データ収集](#-systems-manager-run-command-%E3%81%A7%E6%8F%AE%E7%99%BA%E6%80%A7%E3%83%87%E3%83%BC%E3%82%BF%E5%8F%8E%E9%9B%86)
    - [スクリプト例（Linux）](#%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%97%E3%83%88%E4%BE%8Blinux)
    - [実行手順](#%E5%AE%9F%E8%A1%8C%E6%89%8B%E9%A0%86)
      - [✅ SSM Run Command が SSH/RDP より優れる点](#-ssm-run-command-%E3%81%8C-sshrdp-%E3%82%88%E3%82%8A%E5%84%AA%E3%82%8C%E3%82%8B%E7%82%B9)
  - [💾 EBS ボリュームのスナップショット & タグ付け](#-ebs-%E3%83%9C%E3%83%AA%E3%83%A5%E3%83%BC%E3%83%A0%E3%81%AE%E3%82%B9%E3%83%8A%E3%83%83%E3%83%97%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88--%E3%82%BF%E3%82%B0%E4%BB%98%E3%81%91)
    - [スナップショット作成コマンド](#%E3%82%B9%E3%83%8A%E3%83%83%E3%83%97%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88%E4%BD%9C%E6%88%90%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89)
    - [タグ例](#%E3%82%BF%E3%82%B0%E4%BE%8B)
  - [🛠️ 侵害シナリオの具体例と対処](#-%E4%BE%B5%E5%AE%B3%E3%82%B7%E3%83%8A%E3%83%AA%E3%82%AA%E3%81%AE%E5%85%B7%E4%BD%93%E4%BE%8B%E3%81%A8%E5%AF%BE%E5%87%A6)
  - [🔐 セキュリティチェックリスト](#-%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3%E3%83%81%E3%82%A7%E3%83%83%E3%82%AF%E3%83%AA%E3%82%B9%E3%83%88)
  - [❓ なぜ **Systems Manager Run Command** を使用して揮発性データを収集するのか](#-%E3%81%AA%E3%81%9C-systems-manager-run-command-%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%97%E3%81%A6%E6%8F%AE%E7%99%BA%E6%80%A7%E3%83%87%E3%83%BC%E3%82%BF%E3%82%92%E5%8F%8E%E9%9B%86%E3%81%99%E3%82%8B%E3%81%AE%E3%81%8B)
    - [Run Command のメリット](#run-command-%E3%81%AE%E3%83%A1%E3%83%AA%E3%83%83%E3%83%88)
    - [D. なぜ **SSH / RDP セッション** ではないのか](#d-%E3%81%AA%E3%81%9C-ssh--rdp-%E3%82%BB%E3%83%83%E3%82%B7%E3%83%A7%E3%83%B3-%E3%81%A7%E3%81%AF%E3%81%AA%E3%81%84%E3%81%AE%E3%81%8B)
  - [❓ なぜ **F. State Manager Association** を使わず、**E. create-snapshot** を採用するのか](#-%E3%81%AA%E3%81%9C-f-state-manager-association-%E3%82%92%E4%BD%BF%E3%82%8F%E3%81%9Ae-create-snapshot-%E3%82%92%E6%8E%A1%E7%94%A8%E3%81%99%E3%82%8B%E3%81%AE%E3%81%8B)
  - [❌ なぜオプション **B / D / F** は最適ではないのか？](#-%E3%81%AA%E3%81%9C%E3%82%AA%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3-b--d--f-%E3%81%AF%E6%9C%80%E9%81%A9%E3%81%A7%E3%81%AF%E3%81%AA%E3%81%84%E3%81%AE%E3%81%8B)
    - [オプション B: インスタンスを隔離サブネットに移動](#%E3%82%AA%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3-b-%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E3%82%92%E9%9A%94%E9%9B%A2%E3%82%B5%E3%83%96%E3%83%8D%E3%83%83%E3%83%88%E3%81%AB%E7%A7%BB%E5%8B%95)
    - [オプション D: SSH / RDP セッションでスクリプト実行](#%E3%82%AA%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3-d-ssh--rdp-%E3%82%BB%E3%83%83%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%A7%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%97%E3%83%88%E5%AE%9F%E8%A1%8C)
    - [オプション F: State Manager Association でスナップショット生成](#%E3%82%AA%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3-f-state-manager-association-%E3%81%A7%E3%82%B9%E3%83%8A%E3%83%83%E3%83%97%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88%E7%94%9F%E6%88%90)
  - [📚 補足：RDP セッションとは？Run Command / State Manager の代表的ユースケース](#-%E8%A3%9C%E8%B6%B3rdp-%E3%82%BB%E3%83%83%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%A8%E3%81%AFrun-command--state-manager-%E3%81%AE%E4%BB%A3%E8%A1%A8%E7%9A%84%E3%83%A6%E3%83%BC%E3%82%B9%E3%82%B1%E3%83%BC%E3%82%B9)
    - [🔹 RDP セッションとは](#-rdp-%E3%82%BB%E3%83%83%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%A8%E3%81%AF)
    - [🔹 Run Command が活躍するその他シナリオ](#-run-command-%E3%81%8C%E6%B4%BB%E8%BA%8D%E3%81%99%E3%82%8B%E3%81%9D%E3%81%AE%E4%BB%96%E3%82%B7%E3%83%8A%E3%83%AA%E3%82%AA)
    - [🔹 Systems Manager **State Manager** の主な用途](#-systems-manager-state-manager-%E3%81%AE%E4%B8%BB%E3%81%AA%E7%94%A8%E9%80%94)
      - [Association を作成するとは？](#association-%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B%E3%81%A8%E3%81%AF)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# AWS EC2 × SSM：セキュリティインシデント対応プロセスの安全設計（メモリ保持と隔離）

---

## 📘 Scenario（シナリオ）

AWS Systems Manager（SSM Agent）がインストールされている Amazon EC2 インスタンスが侵害された可能性がある。  
すべての EC2 は Amazon EBS バックエンドで動作しており、インスタンスを停止せずに **以下の要件** を満たしながらインシデント対応を行う必要がある：

1. 🔒 **インスタンスの揮発性メモリ（RAM）と非揮発性メモリ（EBS）のデータを保持**すること  
2. 🏷️ **インスタンスメタデータにインシデントチケット情報をタグとして追加**すること  
3. 🔗 **インスタンスはオンライン状態を維持しながらも、マルウェア拡散を防ぐために隔離**すること  
4. 📝 **RAM 取得など調査アクティビティを完全にログ記録**すること  

---

## 🔎 Investigation Workflow（調査フローの全体像）

1. **検知**  
   - GuardDuty / CloudTrail / EDR などで不審な挙動を検出  
2. **初期トリアージ**  
   - 該当インスタンス ID と VPC／サブネットを確認  
   - 重要度・影響範囲を判断し、インシデントチケットを発行  
3. **隔離 & 保全**  
   - セキュリティグループ更新 + ASG/ELB からデタッチ  
   - Termination Protection を有効化して誤削除を予防  
4. **揮発性データ取得**  
   - SSM Run Command で RAM/ネットワーク/プロセス情報を収集  
5. **ディスク保全**  
   - EBS スナップショットを取得し、タグを付与  
6. **詳細解析**  
   - Snapshot を別アカウント／別リージョンへコピーし、検体をマウント  
7. **根本原因特定 & 封じ込め**  
   - IAM キー漏洩・脆弱性・マルウェア種類を特定  
8. **復旧 & 事後対応**  
   - AMI 再構築、パッチ適用、検出ルール改善  

---

## 🛡️ Isolation Strategy（隔離戦略）

| 隔離対象 | 目的 | 具体的アクション |
|---------|------|----------------|
| **ネットワーク通信** | 感染拡大・C2 通信を遮断 | 制限専用 SG へ付替え or 送受信ゼロの隔離サブネットへ移動 |
| **Auto Scaling Group** | 自動復旧による汚染拡散を防止 | `detach-instances --should-decrement-desired-capacity` |
| **Elastic Load Balancer** | 汚染トラフィックを顧客へ流さない | `deregister-instances-from-load-balancer` |
| **Lifecycle 操作** | 誤停止・誤終了を防ぐ | **Termination Protection** を有効化 (`modify-instance-attribute --disable-api-termination false`) |

### Termination Protection とは？

EC2 インスタンスの **誤った Stop や Terminate を防止** するフラグ。  
- 調査中に他チームが誤ってシャットダウンし証拠が失われるリスクを回避。  
- 解除には明示的な属性変更が必要で、監査ログが残る。

---

## 🔧 Security Group 更新ポイント

| ポート | デフォルト | 隔離後の推奨設定 |
|-------|-----------|----------------|
| 22 / 3389 | 社内 IP など許可 | **全拒否** （SSM は 443/TCP をアウトバウンドで利用） |
| 80 / 443 | ユーザ向け公開 | **全拒否** |
| その他アプリ | 要件に応じ許可 | **全拒否** |

> **ポイント:** SSM Agent はアウトバウンド HTTPS(443) で Systems Manager エンドポイントへ接続するため、**インバウンドはすべて 0** にしても管理操作は可能。

---

## 🚀 Systems Manager Run Command で揮発性データ収集

### スクリプト例（Linux）

```bash
commands=(
  "echo '### Date' && date -u"
  "echo '### Uptime' && uptime"
  "echo '### Memory' && cat /proc/meminfo"
  "echo '### Netstat' && netstat -pantue"
  "echo '### Process' && ps aux --sort=-%mem | head -n 30"
)
```

### 実行手順

1. **S3 バケット** `forensic-logs-bucket` を事前作成し、暗号化とバージョニングを有効化  
2. **Run Command 発行**

```bash
aws ssm send-command   --instance-ids "i-xxxxxxxx"   --document-name "AWS-RunShellScript"   --comment "Forensic RAM dump INC-123456"   --parameters "{"commands":["${commands[*]}"]}"   --output-s3-bucket-name "forensic-logs-bucket"   --output-s3-key-prefix "INC-123456/volatile"
```

3. **CloudWatch Logs** or **S3** で出力確認  
4. 取得コマンド・実行者・タイムスタンプは SSM / CloudTrail に自動記録

#### ✅ SSM Run Command が SSH/RDP より優れる点

| 項目 | SSM Run Command | 手動 SSH/RDP |
|------|-----------------|--------------|
| 監査ログ | 自動で CloudTrail 記録 | OS ログ不十分／別途設定 |
| ネットワーク要件 | インバウンド 0、アウトバウンド 443 のみ | ポート開放が必要 |
| スクリプト自動化 | パラメータ化・再利用可能 | 手動コピー・実行ミスリスク |
| IAM 統合 | Least Privilege CLI 権限 | 鍵管理・パスワード共有が課題 |
| 運用オーバーヘッド | 低（数クリック） | 高（踏み台設定、鍵配布） |

---

## 💾 EBS ボリュームのスナップショット & タグ付け

### スナップショット作成コマンド

```bash
SNAP_ID=$(aws ec2 create-snapshot   --volume-id vol-xxxxxxxx   --description "INC-123456 Evidence Snapshot"   --query 'SnapshotId' --output text)
```

### タグ例

```bash
aws ec2 create-tags --resources "$SNAP_ID" i-xxxxxxxx vol-xxxxxxxx   --tags Key=IncidentTicket,Value=INC-123456          Key=Owner,Value=SecurityTeam          Key=Evidence,Value=Yes
```

> **実例:**  
> - **ランサムウェア感染**：攻撃者の暗号化ツールをディスク内で検出  
> - **コインマイナー**：不正バイナリや crontab に残る C2 情報をスナップショットから解析  

スナップショットを **別アカウント / 別リージョン** にコピーすれば、攻撃者による削除リスクも軽減。

---

## 🛠️ 侵害シナリオの具体例と対処

| 侵害タイプ | 兆候 | 隔離理由 | 追加対策 |
|-----------|------|----------|---------|
| **CoinMiner** | CPU 使用率急上昇、未知プロセス | マイニング拡大を防止 | VPC Flow Logs で C2 を確認 |
| **ランサムウェア** | ファイル暗号化、拡張子変更 | 同一 VPC 内共有ストレージ汚染防止 | S3 バケット版 Versioning & MFA Delete |
| **Credential Dumping** | 異常な IAM API コール | 資格情報再利用防止 | IAM Access Analyzer で権限確認 |

---

## 🔐 セキュリティチェックリスト

- [ ] Termination Protection が有効  
- [ ] 隔離 SG にインバウンド 0.0.0.0/0 Deny 相当を設定  
- [ ] Run Command 実行ログが CloudTrail に記録されている  
- [ ] スナップショットが暗号化され、タグで検索可能  
- [ ] 証拠ファイルを S3 バージョニング + イミュータブルレプリケーションで保護  

---

このプロセスにより、**EC2 インスタンスを停止せず**に証拠保全・隔離・調査を迅速かつ再現性高く実行できます。  
Security チームは最小の運用負荷でインシデント対応を標準化し、監査にも耐えうる証跡を残せます。💡

---

## ❓ なぜ **Systems Manager Run Command** を使用して揮発性データを収集するのか

### Run Command のメリット
| ポイント | 理由 |
|----------|------|
| **インバウンドポート不要** | SSM Agent はアウトバウンド 443 のみを使用するため、侵害インスタンスに新たな穴を開けずに済む。 |
| **完全な監査証跡** | Run Command の実行内容・パラメータ・結果は CloudTrail と SSM Execution Logs に自動で残る。後から“誰が何をしたか”を証明しやすい。 |
| **自動化・再現性** | スクリプトをパラメータ化して再利用でき、オペレーターの手作業ミスを排除。 |
| **IAM Least Privilege** | `ssm:SendCommand` 等、必要最小限の API 権限のみ付与すればよい。 |
| **多 OS 対応** | Linux／Windows／macOS（SSM Agent 対応）を同一インタフェースで操作。 |

### D. なぜ **SSH / RDP セッション** ではないのか
| 課題 | SSH/RDP が不適切な理由 |
|------|----------------------|
| **追加のポート開放** | インバウンド 22/3389 を一時でも開けると、攻撃者が横展開するリスクが増大。 |
| **証跡の抜け漏れ** | シェル操作は OS ログに残らないコマンドもあり、別途セッション録画を用意しないと完全な監査が難しい。 |
| **手作業ミス** | オペレーターのタイポや誤操作が揮発性メモリを上書きし、証拠汚染の恐れ。 |
| **鍵・パスワード管理** | 侵害済みサーバに新鍵を送り込む手間、あるいは既存鍵が盗まれている可能性。 |
| **運用コスト** | Bastion 構築や踏み台経由などネットワーク設計が複雑化する。 |

---

## ❓ なぜ **F. State Manager Association** を使わず、**E. create-snapshot** を採用するのか

| 観点 | E. 直ちに snapshot 作成 (CLI/Run Command) | F. State Manager Association |
|------|------------------------------------------|------------------------------|
| **目的適合性** | インシデント時の “一回限り” の証拠保全に最適 | 定期的な設定ドリフト修正やパッチ適用向け |
| **作業スピード** | 数秒で実行可。トリアージ中に即時スナップショット | Association 作成 → 適用までタイムラグがある |
| **設定オーバーヘッド** | CLI ワンライナー or Run Command で完結 | ドキュメント作成、ターゲット指定、Schedule 設定が必要 |
| **侵害リスク** | Snapshot API だけの最小権限で済む | `ssm:CreateAssociation` など追加権限が必要 |
| **運用対象** | インシデント対応チームだけが実行 | Ops チームの System 管理フローと衝突の可能性 |

> **結論**:  
> - **Run Command** は「一時的・緊急のコマンド実行」に最適。  
> - **State Manager** は「継続的・ポリシーベース構成管理」に適し、今回の用途ではオーバースペック。  

---

これで、なぜ選択肢 **C (Run Command)** が推奨され、**D と F が採用されない**かまで含めて、設計意図を網羅的に文書化できました。  

---

## ❌ なぜオプション **B / D / F** は最適ではないのか？

### オプション B: インスタンスを隔離サブネットに移動
- 目的は「オンラインを維持しつつ隔離」。  
- **セキュリティグループを更新 (オプション A)** するだけなら、IP・EIP・ルート設定を変えずに済み、変更範囲が狭くオーバーヘッドが最小。  
- サブネット移動は **プライベート IP の変更**、Elastic IP 再割当て、ルートテーブル見直し等が発生し、ネットワーク依存のアプリではダウンタイムや設定修正が必要になる。  
- Auto Scaling / ELB 組み込みの場合、移動後に再登録が必要になるなど手順が煩雑。  
- **最小の運用負荷** という要求からは、オプション B より A が優れる。

### オプション D: SSH / RDP セッションでスクリプト実行
- 会社はすでに **AWS Systems Manager** を導入しており、SSM Agent が全インスタンスで稼働中。  
- 隔離後はインバウンドは閉じるため、SSH/RDP ポートを再度開けるのは本末転倒。  
- キー／パスワード管理、踏み台ホスト、セッション録画の整備など **セキュリティと運用コスト** が高い。  
- Run Command は **アウトバウンド 443 のみ** で済み、CloudTrail にフル監査証跡が残る。  
- よって、同じ目的であれば **C (Run Command)** が D よりも安全かつ低コスト。

### オプション F: State Manager Association でスナップショット生成
- **State Manager** は“持続的構成管理”用サービス。定期パッチ適用やドリフト修正が主用途。  
- インシデント対応は **単発・即時実行** が求められる。Association を作成→適用→削除の流れは冗長。  
- 追加 IAM 権限 (`ssm:CreateAssociation` 等) が必要で攻撃面を広げてしまう。  
- **E (直接 create-snapshot + タグ付け)** は API ワンショットで完結し、チケット番号タグを即時に残せる。  
- したがって F は本シナリオでは過剰設計であり、運用上のオーバーヘッドが大きい。

---

> **結論:**  
> - **最小の運用負荷と既存 SSM 活用** という観点から、推奨は **A + C + E**。  
> - **B / D / F** は目的は満たせるものの、ネットワークリスク・監査性・手順の複雑さの点で相対的に劣る。

---

## 📚 補足：RDP セッションとは？Run Command / State Manager の代表的ユースケース

### 🔹 RDP セッションとは
- **Remote Desktop Protocol (RDP)** は、Windows OS に標準搭載されている GUI ベースのリモート接続プロトコル。  
- **用途例**  
  1. Windows サーバの GUI 設定変更やアプリケーション操作  
  2. ヘルプデスクがエンドユーザ PC を遠隔サポート  
  3. グラフィカルツールを使ったログ/イベントビューワ確認  

> インシデント対応では CLI より操作が重く、ポート 3389/TCP を開ける必要があるため隔離中のマシンには推奨されない。

---

### 🔹 Run Command が活躍するその他シナリオ
| シナリオ | 例 |
|----------|----|
| **継続的なパッチ適用前テスト** | 選択したインスタンスにだけパッチ検証スクリプトを即時実行 |
| **ログ収集 / ファイル転送** | `aws s3 cp` コマンドで大量ログを S3 へアップロード |
| **ユーザ定義メトリクス収集** | カスタムシェルで CPU/IO 詳細を計測し CloudWatch へ送信 |
| **アプリケーションリスタート** | Web サーバプロセスを全リージョンで一斉に再起動 |
| **インベントリチェック** | インストール済みパッケージを Greengrass 風に一覧化 |

---

### 🔹 Systems Manager **State Manager** の主な用途
| 用途 | 具体例 |
|------|--------|
| **構成ドリフト防止** | `AWS-ApplyPatchBaseline` ドキュメントを毎日 2:00 JST に適用し、最新パッチを強制 |
| **エージェント常駐** | セキュリティエージェントが削除されたら自動再インストール |
| **ファイル/レジストリ監視** | 特定設定が変更された際に自動で元に戻す |
| **構成ベースライン遵守** | CIS Benchmarks 設定をドキュメント化し、違反があれば自動修復 |

#### Association を作成するとは？
- **Association = ドキュメント + ターゲット + スケジュール + パラメータ** のセット。  
- “この Runbook を○時に / このタグのインスタンスへ / この頻度で実行” という **ポリシー** を定義するもの。  
- → **一度きりのタスク** ではなく、**継続的かつ自動で所定の状態に保つ** ためのメカニズム。

---
