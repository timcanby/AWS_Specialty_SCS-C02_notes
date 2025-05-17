<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [マルチアカウント認証・認可設計：スケーラブルかつネイティブに構成するための出発点](#%E3%83%9E%E3%83%AB%E3%83%81%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E8%AA%8D%E8%A8%BC%E3%83%BB%E8%AA%8D%E5%8F%AF%E8%A8%AD%E8%A8%88%E3%82%B9%E3%82%B1%E3%83%BC%E3%83%A9%E3%83%96%E3%83%AB%E3%81%8B%E3%81%A4%E3%83%8D%E3%82%A4%E3%83%86%E3%82%A3%E3%83%96%E3%81%AB%E6%A7%8B%E6%88%90%E3%81%99%E3%82%8B%E3%81%9F%E3%82%81%E3%81%AE%E5%87%BA%E7%99%BA%E7%82%B9)
  - [📘 Scenario（シナリオ）](#-scenario%E3%82%B7%E3%83%8A%E3%83%AA%E3%82%AA)
  - [🧠 重要ポイント](#-%E9%87%8D%E8%A6%81%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88)
  - [✅ 対応方法のまとめ（設計出発点）](#-%E5%AF%BE%E5%BF%9C%E6%96%B9%E6%B3%95%E3%81%AE%E3%81%BE%E3%81%A8%E3%82%81%E8%A8%AD%E8%A8%88%E5%87%BA%E7%99%BA%E7%82%B9)
    - [1. IAM Identity Center のデフォルトディレクトリを使用する](#1-iam-identity-center-%E3%81%AE%E3%83%87%E3%83%95%E3%82%A9%E3%83%AB%E3%83%88%E3%83%87%E3%82%A3%E3%83%AC%E3%82%AF%E3%83%88%E3%83%AA%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%99%E3%82%8B)
    - [2. グループと Permission Set を紐付けて各アカウントへ割り当て](#2-%E3%82%B0%E3%83%AB%E3%83%BC%E3%83%97%E3%81%A8-permission-set-%E3%82%92%E7%B4%90%E4%BB%98%E3%81%91%E3%81%A6%E5%90%84%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E3%81%B8%E5%89%B2%E3%82%8A%E5%BD%93%E3%81%A6)
    - [3. ユーザーポータルからのアクセスを推奨](#3-%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E3%83%9D%E3%83%BC%E3%82%BF%E3%83%AB%E3%81%8B%E3%82%89%E3%81%AE%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%82%92%E6%8E%A8%E5%A5%A8)
  - [🚫 避けるべき方法](#-%E9%81%BF%E3%81%91%E3%82%8B%E3%81%B9%E3%81%8D%E6%96%B9%E6%B3%95)
    - [1. オンプレミスまたは外部ディレクトリへの依存](#1-%E3%82%AA%E3%83%B3%E3%83%97%E3%83%AC%E3%83%9F%E3%82%B9%E3%81%BE%E3%81%9F%E3%81%AF%E5%A4%96%E9%83%A8%E3%83%87%E3%82%A3%E3%83%AC%E3%82%AF%E3%83%88%E3%83%AA%E3%81%B8%E3%81%AE%E4%BE%9D%E5%AD%98)
    - [2. IAM ユーザーとの直接マッピング](#2-iam-%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E3%81%A8%E3%81%AE%E7%9B%B4%E6%8E%A5%E3%83%9E%E3%83%83%E3%83%94%E3%83%B3%E3%82%B0)
  - [🛠️ 実装例](#-%E5%AE%9F%E8%A3%85%E4%BE%8B)
    - [IAM Identity Center の初期セットアップ](#iam-identity-center-%E3%81%AE%E5%88%9D%E6%9C%9F%E3%82%BB%E3%83%83%E3%83%88%E3%82%A2%E3%83%83%E3%83%97)
  - [📌 結論](#-%E7%B5%90%E8%AB%96)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# マルチアカウント認証・認可設計：スケーラブルかつネイティブに構成するための出発点

---

## 📘 Scenario（シナリオ）

ある企業は AWS Organizations を使用して複数の AWS アカウントを統合管理しており、すでに IAM Identity Center（旧 AWS SSO）を有効化しています。  
セキュリティエンジニアは、全社的なユーザーアクセス管理を行う必要があります。

- 追加のユーザー管理用インフラ（例：Active Directory）は導入したくない
- 各従業員の職能に応じたアクセス権限を割り当てたい
- スケーラブルかつ一元的に管理できる仕組みを求めている

---

## 🧠 重要ポイント

| 項目 | 説明 |
|------|------|
| ✅ フルマネージド型 | AWS が提供する Identity Center を使うことで、追加構築なく構成可能 |
| ✅ 一元的なユーザー管理 | IAM Identity Center でユーザーとグループを直接管理可能 |
| ✅ ジョブベースの認可 | 各グループに対し Permission Set を定義し、アカウントロールに紐付け |
| ✅ ユーザーポータル | 各ユーザーはポータルから自身に割り当てられたアカウント・ロールへアクセス可能 |
| ✅ AWS Organizations との統合 | アカウント追加時も Identity Center 側で統一的に権限管理ができる |

---

## ✅ 対応方法のまとめ（設計出発点）

### 1. IAM Identity Center のデフォルトディレクトリを使用する

- ユーザーとグループを IAM Identity Center 内で作成・管理
- 外部のディレクトリ（AD Connector、Microsoft ADなど）は不要

### 2. グループと Permission Set を紐付けて各アカウントへ割り当て

- 各グループに対し、ジョブファンクションに基づいた権限セット（Permission Set）を定義
- AWS Organizations のアカウントに対し、グループ単位で権限をマッピング

### 3. ユーザーポータルからのアクセスを推奨

- 従業員は IAM Identity Center のポータルから各 AWS アカウントにアクセス
- ロールの選択、SSO ログイン、セッション管理が簡単に行える

---

## 🚫 避けるべき方法

### 1. オンプレミスまたは外部ディレクトリへの依存

- AD Connector や AWS Managed Microsoft AD を使うと、ディレクトリの保守や接続構成が発生
- 「追加のユーザー管理インフラを使わない」という前提に反する

### 2. IAM ユーザーとの直接マッピング

- Identity Center は IAM ユーザーとの直接リンクを想定しておらず、非効率かつ運用ミスの原因となる
- モダンなアクセス制御には Permission Set + ロール割り当てが基本

---

## 🛠️ 実装例

### IAM Identity Center の初期セットアップ

1. AWS Organizations を有効にしておく
2. IAM Identity Center を有効化し、デフォルトディレクトリを選択
3. IAM Identity Center 上でユーザー・グループを作成
4. 各グループに対し、必要な Permission Set を定義（例: 管理者、開発者、監査者など）
5. Permission Set を AWS アカウントにマッピング

---

## 📌 結論

- ✅ AWS ネイティブな機能（IAM Identity Center）を最大限に活用し、
  **追加構成不要・一元管理・スケーラブル・セキュア** な認証認可基盤を構築可能。
- ✅ 「外部ディレクトリに頼らない構成」として、最小コスト・最小運用負荷で展開できる最良の選択肢。
