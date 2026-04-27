---
title: AWS 展開案（移行設計の考え方）
layout: default
---

# AWS 展開案（移行設計の考え方）

Version: 2026-04-27
Author: gan2

> **このページの位置づけ**: 本プロジェクトで OSS／オンプレ前提に構築した多段プロキシ・認証・ログ基盤を、
> AWS 上に再配置する場合の **設計の考え方を整理した計画ドキュメント** です。
> 実機検証はまだ未実施のため、「実装済み」ではなく **設計判断の整理段階** として記載しています。

---

## 0. このページで伝えたいこと

- オンプレ／OSS 環境で構築した責務分離（経路制御・認証・暗号化・検査・ログ）を **AWS のマネージドサービスに置き換える際の対応関係** を整理しています
- 「OSS から AWS への置き換え一覧」だけでなく、**AWS に持っていくと再検証が必要になる箇所**（DNS／認証／NAT 境界）も明示します
- AWS 経験はまだ実務での設計・構築に踏み込めていない段階のため、本ページは **学習ロードマップを兼ねた計画** という位置づけです

---

## 1. オンプレ／OSS 構成 ↔ AWS サービス対応（案）

| 区分 | 現構成（OSS） | AWS 展開案（候補） | 想定されるポイント |
|---|---|---|---|
| プロキシ（入口／分岐／出口） | Squid 3段 | EC2（Squid を AMI 化） / ALB + NLB / 必要に応じて Network Firewall | クラウドでは「単一プロキシ + 出口制御」が主流。多段にする場合は責務を再整理 |
| 中継暗号化 | stunnel | VPC 内通信は Security Group や Private Subnet で到達範囲を制御し、必要に応じて ACM や TLS 終端で通信暗号化を設計する（VPC Flow Logs で可視化） | stunnel 相当の責務は VPC 設計／SG／ACM の組合せで再設計する想定 |
| コンテンツ検査 | ICAP / ClamAV | サードパーティ AMI（CloudGuard 等） / GuardDuty / Inspector | 「マルウェア検査」と「脅威検知」を分けて再設計する必要あり |
| 認証 | OpenLDAP / Samba AD/DC / Kerberos | AWS Directory Service（AD Connector / Managed AD） / IAM Identity Center | Kerberos 前提（SPN／時刻／逆引き）は Managed AD で多くが吸収される |
| DNS / 経路制御 | dnsmasq（Split DNS） / PAC | Route 53（Private Hosted Zone） / Route 53 Resolver / ALB のホストベースルーティング | AWS 自体が PAC を直接提供するわけではないため、S3 / CloudFront での配布やクライアント設定との組合せで代替を検討する |
| ログ収集 | Promtail | CloudWatch Agent / Kinesis Data Firehose / OpenTelemetry Collector | エージェント選定は「集約先」で決まる |
| ログ集約・検索 | Loki / Graylog / OpenSearch | CloudWatch Logs / OpenSearch Service / Athena（S3 直接検索） | Loki 相当は CloudWatch Logs Insights、Graylog 相当は OpenSearch Service |
| 監視 | Zabbix（TLS-PSK + Sidecar） | CloudWatch Metrics + Alarms / SNS / EventBridge | 「メトリクス／アラート／通知」を AWS 内で分担 |
| 自動化 | Bash スクリプト群（STEP0〜17） | CloudFormation / Terraform / Systems Manager Automation | 宣言的構成へ書き換えることで再現性をさらに上げる |
| 実行環境 | WSL2 + Docker + VMware | EC2 / ECS（Fargate） / EKS | 単純移植は EC2、責務単位の再設計時は ECS／EKS |

---

## 2. ネットワーク境界の再設計ポイント

オンプレ前提で組んだ構成を AWS に移すと、**境界の前提が変わる**ため、再検証が必要な箇所が出てきます。
本プロジェクトで責務分離した観点を、そのまま AWS に持ち込むときに気をつけるべき点を整理しました。

### 2-1. VPC / サブネット設計
- 現構成の「Proxy1（入口）／Proxy2（分岐）／Proxy3（出口）」は、**Public / Private サブネット境界**にどう配置するかを再検討
- Proxy3（出口）は NAT Gateway もしくは VPC Endpoint との関係を整理
- Proxy 間通信の暗号化境界（現構成では stunnel）は、VPC 内では SG＋ACM の組合せに置き換えられる可能性あり

### 2-2. 認証境界（Kerberos / AD）
- Kerberos が前提とする **時刻同期・SPN・逆引き DNS** は、AWS Managed Microsoft AD で多くが吸収される
- ただし「Linux クライアントから AD Kerberos で SSO」のシナリオは、**Managed AD への参加方法・SPN 登録**を再検証する必要がある
- 本プロジェクトで整理した「認証失敗を発生レイヤで切り分ける観点」は AWS でも有効

### 2-3. 経路制御（PAC の置き換え）
- AWS 自体が PAC を直接提供するわけではないため、運用方式は環境に応じて設計する必要がある
- 代替として以下を検討:
  - **PAC 配布**: S3 / CloudFront でホスティングし、クライアント設定と組み合わせて配布する設計
  - **クライアント側設定**: 端末管理（Intune／Jamf 等）で Proxy 設定を配布
  - **DNS ベース**: Route 53 Resolver の DNS Firewall でドメイン単位の経路制御を検討
  - **ALB / NLB**: ホストベースルーティングで宛先を分岐
- 「経路をログで判別できる」という観点は、**VPC Flow Logs ＋ ALB アクセスログ** などの組合せで再現できないかを検討する

### 2-4. ログ・監視の責務再分担
- 現構成は「Loki＝経路の判定」「Graylog＝詳細検索」「Zabbix＝死活監視」と役割を分けている
- AWS では:
  - 経路判定 → CloudWatch Logs Insights / Athena
  - 詳細検索 → OpenSearch Service
  - 死活監視 → CloudWatch Metrics + Alarms
- ポイントは **「結果（HTTP コード）→ 原因（DENIED／TLS）」の切り分けフローを、AWS のサービス組合せで再現できるかどうか**

---

## 3. 段階的な移行ステップ（学習・検証順序）

実機検証を進める場合、以下の順序を想定しています。

| Step | 内容 | 学べること |
|---|---|---|
| 1 | 単一 EC2 上で Squid を AMI 化し、ALB 配下で動作させる | EC2／ALB／SG／ACM の基本 |
| 2 | Private Hosted Zone（Route 53）で内部 FQDN を整備 | DNS 境界設計 |
| 3 | CloudWatch Agent でアクセスログを CloudWatch Logs に集約 | ログ収集設計 |
| 4 | Managed AD を立て、EC2（Linux）から Kerberos で参加 | 認証境界の再現 |
| 5 | 多段化（入口／分岐／出口）を ECS（Fargate）で再構成 | 責務分離のクラウド再現 |
| 6 | OpenSearch Service と CloudWatch Logs Insights で切り分けフローを再構築 | 可観測性の再設計 |
| 7 | CloudFormation／Terraform で IaC 化 | 再現性の宣言的担保 |

> このステップは「AWS 完全対応」を主張するものではなく、**現構成の責務分離を AWS でも維持できるかを段階的に検証する学習計画** です。

---

## 4. 想定される設計上の論点（面接で聞かれそうな点）

| 論点 | 整理している考え |
|---|---|
| なぜ多段プロキシを AWS で作るのか | AWS 単独で代替できる部分が多い。多段化は「責務分離の理解」と「経路ログの追跡性」を残す目的で位置づける |
| Squid をマネージドに置き換えないのか | アクセスログ仕様や PAC 配布など Squid 固有の挙動を残したい場合は EC2／コンテナで動かす選択肢が現実的 |
| Kerberos を AWS でどう扱うか | Managed AD で SPN／時刻同期は吸収。Linux 側の `krb5.conf`／keytab 運用は本プロジェクトの整理がそのまま活きる |
| PAC は AWS でどう代替するか | クライアント側配布／Route 53 Resolver／ALB のホストルーティングへ役割分散 |
| 切り分けフロー（access→cache）は AWS でどう再現するか | CloudWatch Logs Insights ＋ OpenSearch Service ＋ VPC Flow Logs の組合せで近い情報量に到達できる想定 |

---

## 5. 現状の到達点と次のステップ

- **到達点**: AWS 上での「責務分離の維持」を考えるための **対応表と論点整理は完了**
- **次のステップ**: 単一 EC2 + Squid + CloudWatch Logs から始め、段階的に責務を分散させる検証
- **正直な書き方**: 本ページは設計の考え方を整理した段階であり、AWS 上での **実機検証はまだこれから** です

---

## 関連ドキュメント

- [Index（設計意図・全体構成）](./index.html)
- [Verification（動作証跡）](./verification.html)
- [Automation（自動化・再現性）](./automation.html)
- [面接用ピッチ](./interview-pitch.html)
