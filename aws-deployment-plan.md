---
title: AWS移行設計案：3段Proxy・認証・ログ基盤
layout: default
---

# AWS移行設計案：3段Proxy・認証・ログ基盤

Version: 2026-04-29
Author: gan2

> **本ページは、オンプレ / Docker Composeで構築した3段Proxy・認証・ログ基盤を、AWSへ移行するための設計案です。現時点ではAWS上での構築は未実施であり、次フェーズではTerraform / CloudFormation / Systems Manager Automation等を用いて、最小構成のIaC化および構築検証を進める予定です。**

---

## 0. このページの位置づけ

*1行サマリ: 設計段階の AWS 移行案。実装は次フェーズで検証予定。*

- 現行のオンプレ / Docker Compose 構成を AWS 上に再構成する **設計案**
- AWS 上での構築は **未実施**。次フェーズで Terraform / CloudFormation / SSM Automation により **最小構成のIaC化と検証** を進める
- 中小企業（50〜300名）向け **B案（推奨：バランス型）** を採用候補として提示
- 構成判断には **可用性・セキュリティ・コスト・運用性のトレードオフ** を併記
- 表現原則：実装完了を断定する語は使わず、「設計案」「検証予定」「次フェーズで構築予定」を用いる

---

## 1. 図で見る設計概要

*1行サマリ: 4 枚の図面で全体像を把握する。文章は図に合わせて記述する。*

### 1-1. 構成比較図

![AWS構成比較 - 案A 単純移植 / 案B 推奨（バランス型）/ 案C 監査強化](./images/01_architecture_decision_comparison.png)

- **案A：単純移植（EC2で全再現）** — 低コスト・高運用負荷。シンプル構成だが運用は手動中心
- **案B：推奨構成（バランス型）** — オンプレ構成を維持しつつ、AWS で可用性・運用性・監視性を最適化
- **案C：監査強化（高セキュリティ）** — Network Firewall / OpenSearch / Security Hub / AWS Config / CloudTrail / S3 Log Archive を組み合わせ。高コスト・高機能
- **採用：案 B** — 認証はオンプレ Samba AD/DC を正本、AWS Client VPN 経由のみ受付、PAC 配布で経路選択、3 段 Proxy で責務分離、AWS マネージドを必要なところに留める
- **将来検討**：AWS Network Firewall / Managed Microsoft AD / Auto Scaling

### 1-2. AWS全体構成図（B案）

![B案 AWS全体構成図 - 中小企業向け 3段Proxy・認証・ログ基盤](./images/02_architecture_overview.png)

図に存在する構成のみを記述します。

- **AWS Client VPN（必須）** — オンプレ環境（Samba AD/DC、dnsmasq、GPO、Windows Client）から VPC への唯一の到達経路
- **PAC（CloudFront + S3）** — 経路選択ファイルを CDN 配布、WPAD/PAC として配布
- **Private Subnet（2AZ）** に **3段Proxy** を配置：
  - **Proxy3：認証・入口制御** — Kerberos / LDAP 認証、入口 ACL、アクセス管理
  - **Proxy2：経路制御・ICAP連携** — 中継、DIRECT、TLS 再暗号化、検査ポイント
  - **Proxy1：出口制御** — HTTP / HTTPS（80 / 443）、出口 ACL
- **内部通信は Squid TLS を採用**（stunnel は使用しない）
- **ICAP / ClamAV** — Proxy2 と連携してコンテンツ検査
- **AD Connector** — VPN 経由でオンプレ Samba AD/DC を参照（認証の正本はオンプレに残置）
- **NAT Gateway（AZ ごと配置）** — Private Subnet からインターネットへの出口集約
- **運用・監視・ログ基盤**：
  - **Amazon CloudWatch Logs**（Subscription Filter）
  - **CloudWatch Alarm**（異常検知）
  - **CloudWatch Agent**
  - **Amazon S3**（ライフサイクル管理）
  - **Amazon OpenSearch Service**（ログ分析・全文検索・ダッシュボード）
  - **Athena**（S3 への SQL 分析）
  - **AWS Systems Manager Session Manager**（踏み台不要）
- **将来検討**：AWS Network Firewall / Managed Microsoft AD / Auto Scaling

通信フロー（図下部の流れ）:
`Client → VPN接続 → PAC設定 → Proxy3（認証） → Proxy2（経路） → ICAP/ClamAV（検査） → Proxy1（出口） → NAT Gateway → Internet`

### 1-3. 通信フロー図

![通信フロー図 - B案 3段Proxy + PAC制御](./images/03_network_flow.png)

下から上へ通信が進みます。

1. **Client（Windows PC、GPO 配布済み）**
2. **AWS Client VPN** — VPN 以外の通信を遮断、認証 ID / MFA、ルートテーブルで Proxy3 方向へ中継
3. **PAC（経路選択）** — CloudFront + S3 から `proxy.pac` を取得、DIRECT or Proxy 振分
4. **AD認証（Kerberos / LDAP）** — AD Connector → Samba AD/DC 参照、アクセス管理
5. **Proxy3：認証・入口制御** — ユーザ認証、入口 ACL
6. **Proxy2：経路制御** — 中継、DIRECT、ICAP サービスへ転送、内部通信は Squid TLS で再暗号化
7. **ICAP / ClamAV：検査ポイント** — 不正検査・ウイルス検査・ファイルチェック
8. **Proxy1：出口制御** — HTTP / HTTPS（80 / 443）、出口 ACL
9. **NAT Gateway（Public Subnet）** — 外部接続点、出口を集約
10. **Internet** へ HTTP / HTTPS で到達

横断の **運用・監視・ログ基盤**：CloudWatch Logs（収集）→ CloudWatch Alarm（異常検知）→ S3（永続保管）+ SSM Session Manager（踏み台不要）。
**将来検討（段階導入）**：AWS Network Firewall / Managed Microsoft AD。

設計ポイント（図下部）：
- 通信は段階的に制御され、各レイヤで責務分離されている
- 障害発生時は Proxy ログを CloudWatch で段階的に切り分け可能

### 1-4. レイヤー分解図

![アーキテクチャ - 3段Proxy レイヤー分解図](./images/04_architecture_layers.png)

責務を **7 レイヤ + Logging Layer（横断）** に分離します。

- **① Egress Layer（外部通信）** — NAT Gateway（パブリックサブネット内）→ Internet
- **② Inspection Layer（検査）** — ICAP / ClamAV、HTTP / HTTPS（80 / 443）、Squid TLS
- **③ Proxy Layer（通信制御）** — Proxy1（出口制御）→ Proxy2（経路制御・ICAP連携）→ Proxy3（認証・入口制御）。**内部通信は Squid TLS を採用（stunnel は使用しない）**
- **④ Auth Layer（認証）** — AD Connector が VPN 経由で Samba AD/DC（認証の正本）を参照、Kerberos / LDAP
- **⑤ Control Layer（経路制御）** — PAC（CloudFront + S3）、GPO / 政策ファイル、Split tunnel / Full tunnel
- **⑥ Access Layer（接続）** — AWS Client VPN
- **⑦ Client Layer（利用者）** — Windows PC、GPO 配布済み
- **Logging Layer（横断）** — CloudWatch Logs / S3 / SSM Session Manager

> 各レイヤで責務を分離し、セキュリティと運用性を両立する設計です。

---

## 2. 現行オンプレ構成と AWS 移行案の対応関係

*1行サマリ: 図に存在する要素のみを対応表で示す。*

| 現行構成（オンプレ / Docker Compose） | AWS 構成（B案） | 採用理由 | 検証ポイント |
|---|---|---|---|
| Windows Client（社内端末） | クライアント側はそのまま、AWS には移さない | 既存端末ポリシー・GPO 配布をそのまま流用 | VPN クライアント配布、PAC 取得経路 |
| Samba AD/DC（オンプレ・正本） | **オンプレに正本を残置**、AWS は **AD Connector で VPN 経由参照** | 認証の正本を変えずに段階移行 | DNS / Kerberos / 時刻同期、AD Connector の設置サブネット |
| dnsmasq（社内 DNS / WPAD 補助） | **Route 53 Resolver / DNS Firewall** | マネージドな DNS 解決と DNS レイヤのフィルタ | Resolver Endpoint 配置、社内 DNS との整合 |
| WPAD / PAC（社内配布） | **PAC（CloudFront + S3）** | 静的配信で運用コスト低、CDN で大規模配布も対応 | PAC キャッシュ TTL、HTTPS 配布、社内 DNS との整合 |
| Proxy1（オンプレ・出口役） | **EC2 + Squid（Proxy1：出口制御）** | 出口 ACL と NAT GW 集約 | NAT GW のポート確保、宛先 ACL 反映フロー |
| Proxy2（オンプレ・中継役） | **EC2 + Squid（Proxy2：経路制御・ICAP連携）** | ICAP 連携・経路分岐の責務を維持 | ICAP 待ち時間、Proxy 間 SG 設計 |
| Proxy3（オンプレ・入口認証役） | **EC2 + Squid（Proxy3：認証・入口制御）** | 既存 ACL / 認証ログ仕様を踏襲 | AD Connector 連携、認証失敗ログ集約 |
| stunnel（中継暗号化） | **不採用。Squid TLS で内部通信を暗号化** | TLS 終端を Squid に集約し、運用要素を減らす | TLS バージョン、証明書ローテーション運用 |
| ICAP / ClamAV | **EC2 上の ICAP / ClamAV** | 既存検査ロジックを継続利用 | 定義ファイル更新、メモリ使用率、スキャン遅延 |
| Loki（経路判定） | **CloudWatch Logs（Insights）** | 経路判定（access→cache）を CW Logs Insights で再現 | クエリ性能、ログ保存期間、コスト |
| Graylog（全文検索） | **Amazon OpenSearch Service** | 全文検索とダッシュボード、Athena で S3 への SQL 分析も併用 | ノード数、ストレージ、ログ取り込みパイプライン |
| Zabbix（死活監視） | **CloudWatch Alarm**（CloudWatch Agent でメトリクス収集） | マネージドで運用負荷削減、SNS 通知へ接続容易 | アラーム閾値、誤検知率、通知先 |
| Bash 自動化（STEP0〜17） | **Terraform / CloudFormation + GitHub Actions + SSM Automation** | 宣言的構成で再現性、承認付き展開へ移行 | plan→apply の運用、ロールバック手順、ドリフト検出 |

---

## 3. 構成案 A / B / C の比較

*1行サマリ: 採用は B 案（バランス型）。コスト・可用性・運用負荷の総合判断。*

| 軸 | 案A: 単純移植 | 案B: 推奨（バランス型） | 案C: 監査強化 |
|---|---|---|---|
| Proxy 構成 | 1 ホストへ責務集約 | **3 段（Proxy1 / 2 / 3、2AZ）** | 3 段 + Network Firewall |
| 内部通信暗号化 | TLS（stunnel または Squid TLS） | **Squid TLS（stunnel は使用しない）** | Squid TLS + Network Firewall |
| NAT Gateway | 1 台 | 初期 1 台 / 標準 2 台（AZ ごと配置） | 2 台 |
| ログ集約 | CloudWatch Logs のみ | **CW Logs + S3 + OpenSearch + Athena** | CW Logs + S3 + OpenSearch + Security Hub |
| 認証 | AD Connector | **AD Connector（オンプレ Samba AD/DC を正本参照）** | Managed AD + IAM Identity Center |
| 検査・監査 | なし | 必要に応じ GuardDuty 検討 | Network Firewall + Security Hub + GuardDuty + AWS Config + CloudTrail |
| 可用性 | 中（責務集約で切り分け困難） | **高（2AZ で継続）** | 高 |
| 月額コスト目安 | 中 | **中（バランス）** | 高 |
| 運用負荷 | 中（手動運用ベース） | **中** | 高 |
| 監査性 | 中 | **中（CW Logs + S3 + OpenSearch）** | 高（フル監査ログ + 自動コンプラ評価） |
| 推奨場面 | 検証・学習 | **50〜300 名の標準業務** | 監査・証跡・コンプライアンス要求 |

**B 案を採用する根拠**

- A 案は低コストだが運用負荷が高く、障害切り分けが難しい
- C 案は高機能だがコスト・複雑性が高く、監査要件が明確でない段階では過剰
- B 案は **既存のオンプレ責務分離を維持しつつ AWS マネージドを活用**、段階的に C 案へ拡張できる
- 運用保守経験を AWS 設計に直接接続しやすい

---

## 4. 採用構成：B案

*1行サマリ: 3 段 Proxy + 2AZ + AD Connector + PAC + CloudWatch / S3 / OpenSearch / SSM が骨子。*

| 構成要素 | 推奨値 | 役割 / トレードオフ |
|---|---|---|
| VPC | 1 | シンプル化と運用負荷低減 |
| AZ | 2（AZ-a / AZ-c） | 単一 AZ 障害で業務断を避ける |
| Public Subnet | 2 | NAT Gateway 配置 |
| Private Subnet | 2 | 3 段 Proxy / ICAP / AD Connector を配置 |
| Proxy EC2 | **3 段（Proxy1：出口 / Proxy2：中継 / Proxy3：入口）** | OSS 構成の責務分離を AWS でも維持 |
| ICAP / ClamAV | EC2 | Proxy2 と連携してコンテンツ検査 |
| 内部通信暗号化 | **Squid TLS（stunnel は使用しない）** | TLS 終端を Squid に集約し運用要素を減らす |
| NAT Gateway | 初期 1 台 / 標準 2 台 | コスト優先 vs 可用性優先のトレードオフ |
| AD Connector | small / large | VPN 経由で既存 Samba AD/DC を参照 |
| AWS Client VPN | 利用人数で段階拡張 | オンプレ → VPC の唯一の経路 |
| PAC 配布 | CloudFront + S3 | 静的配信で運用コスト低 |
| DNS 解決 | Route 53 Resolver / DNS Firewall | dnsmasq 相当をマネージドに置換 |
| CloudWatch Logs / Alarm / Agent | 必須 | アクセスログ集約・異常検知・メトリクス収集 |
| S3（Log Archive） | 必須 | ライフサイクル管理付きの長期保管 |
| OpenSearch Service | 採用（B案） | ログ分析・全文検索・ダッシュボード |
| Athena | 採用（B案） | S3 ログの SQL 分析 |
| SSM Session Manager | 必須 | 踏み台不要、IAM 監査が効く |
| Network Firewall / Managed Microsoft AD / Auto Scaling | **将来検討（段階導入）** | 監査強化・ID 基盤刷新・需要増の要件発生時に追加 |

---

## 5. 採用構成の設計判断

*1行サマリ: 図にある構成について「なぜそれを選ぶのか」を 7 観点で整理する。*

### 5-1. なぜ 3 段 Proxy なのか

- **Proxy3：認証・入口制御** — AD Connector 経由で Samba AD/DC を参照、入口 ACL、認証失敗の判定（最初の段階で検出）
- **Proxy2：経路制御・ICAP連携** — 中継、ICAP / ClamAV 連携、TLS 再暗号化、将来の検査機能の追加先
- **Proxy1：出口制御** — 出口 ACL、HTTP / HTTPS（80 / 443）、NAT GW への集約点
- **目的は冗長化ではなく責務分離による障害切り分け** — どの段で問題が起きたかをログで切り分けられる構造を AWS 上でも維持する

### 5-2. なぜ Proxy を Private Subnet に置くのか

- **外部から直接アクセスさせない** — Proxy にグローバル IP を持たせない
- **VPN 経由でのみ接続** — Client → Proxy3 への到達は AWS Client VPN を経由
- **Internet 向け通信は NAT Gateway に集約** — 出口を一点に絞ることで監査と制御が容易
- **Security Group で通信元・通信先を限定** — Proxy 間 / Proxy → NAT GW / Proxy → AD Connector を最小権限で制御

### 5-3. なぜ PAC なのか

- **社内通信は DIRECT** — VPC 内・社内向けは Proxy を経由せず効率化
- **外部通信は Proxy 経由** — インターネット向けのみ Proxy で集約制御
- **複数 Proxy への振り分け** — 3 段の入口（Proxy3）へ確実に向ける、将来の追加 Proxy へも動的分配
- **障害時・検証時の経路切替** — PAC の差し替えのみで経路を切替可能

### 5-4. なぜ AD Connector なのか

- **認証の正本は Samba AD/DC（オンプレ）に残す** — オンプレ ID 基盤を変更せずに段階移行
- **AWS 側は AD Connector で参照口を提供** — VPN 経由で Samba AD/DC を参照
- **Managed Microsoft AD は将来検討** — ID 基盤を AWS 側に集約する段階で再評価
- **連携検証は DNS / Kerberos / 時刻同期を含めて行う** — Kerberos 前提の SPN・逆引き・時刻ずれを検証対象とする

### 5-5. なぜ Squid TLS（stunnel 不採用）なのか

- **TLS 終端を Squid に集約** — Proxy2 の TLS 再暗号化を Squid 自身で処理し、運用要素を減らす
- **stunnel を運用から外す** — サイドカープロセスの監視・証明書管理・障害時切り分けを 1 段減らせる
- **Squid 単体で観測しやすい** — 内部通信の TLS 状態と access ログを同じ系で扱える
- **将来の AWS マネージド連携の選択肢を残す** — ACM / NLB TLS Listener への移行余地を残す

### 5-6. なぜ CloudWatch / S3 / OpenSearch / SSM なのか

- **CloudWatch Logs：ログ収集・障害調査** — Proxy / ICAP / VPN ログを統一的に集約、Subscription Filter で OpenSearch へ配信
- **CloudWatch Alarm：異常検知** — 認証失敗率、出口エラー率、プロセス停止を即時検知
- **CloudWatch Agent：メトリクス収集** — Zabbix 相当の死活・リソース監視を AWS 側で受ける
- **S3：長期保管・ライフサイクル管理** — Standard → IA → Glacier で監査ログを安価に保管、SSM 操作証跡もここへ
- **OpenSearch / Athena：ログ分析** — Graylog 相当の全文検索・ダッシュボード、Athena は S3 への SQL 分析
- **SSM Session Manager：踏み台不要・SSH 鍵管理削減** — IAM 監査と CloudTrail で操作証跡を残す

### 5-7. なぜ B 案なのか

- **A 案は低コストだが運用負荷が高い** — 1 ホストに責務が集中し、障害切り分けが難しい
- **C 案は高機能だがコストと複雑性が高い** — 監査要件が明確でない段階では過剰
- **B 案は既存の強みを維持しつつ AWS 運用へ最適化できる** — 3 段 Proxy の責務分離を残しつつ、ログ・監視・運用は AWS マネージドへ寄せる
- **運用保守経験を AWS 設計に接続しやすい** — オンプレで身につけた切り分け観点をそのまま面接・実装で語れる

### 5-8. Well-Architected 6 本柱との対応

| 柱 | 設計上の考慮 | 採用 / 検討する AWS サービス | トレードオフ |
|---|---|---|---|
| Operational Excellence | IaC 化・実行前承認・構築後検証 | Terraform / CFn, SSM Automation, GitHub Actions | 自動化整備コスト vs 手動運用負荷 |
| Security | 最小権限 IAM・SG 最小化・SSM Session Manager・CloudTrail | IAM Identity Center, STS AssumeRole, CloudTrail | セキュリティ強化 vs 運用速度 |
| Reliability | Multi-AZ・Proxy 多段冗長・Alarm | Multi-AZ, CloudWatch Alarm | 冗長化コスト vs 単一障害許容 |
| Performance Efficiency | 適切なインスタンス・Squid キャッシュ・スケール余地 | t3 / t3a / m6i, Squid キャッシュ | 過剰スペック vs 性能不足 |
| Cost Optimization | NAT GW 数・OpenSearch サイジング・S3 ライフサイクル | Cost Explorer, Budgets, Compute Savings Plans | 可用性 vs 月額コスト |
| Sustainability | 低消費インスタンス・不要リソース削除 | Graviton (t4g / c7g), Instance Scheduler | 互換性検証コスト vs 消費電力削減 |

---

## 6. 運用・監視・ログ設計

*1行サマリ: CloudWatch Logs を起点に、S3・OpenSearch・SSM で観測と監査を担う。*

- **ログ収集**: Proxy / ICAP / VPN すべて CloudWatch Logs に集約。CloudWatch Agent で OS / プロセスメトリクスも収集
- **ログ相関**: 認証失敗（407 → Proxy3）→ ACL 拒否（403 → Proxy1）→ ICAP 遮断（Proxy2 経由）の順で切り分けるフローを CW Logs Insights で再現
- **長期保管**: Subscription Filter で S3 へ転送し、Standard → IA → Glacier でコスト最適化。SSM 操作の証跡もここへ
- **検索・分析**: OpenSearch Service（全文検索・ダッシュボード）と Athena（S3 への SQL 分析）を併用
- **異常検知**: CloudWatch Alarm（認証失敗率・出口エラー率・プロセス停止）→ SNS 通知
- **運用アクセス**: SSM Session Manager に統一し、踏み台 EC2・SSH 鍵を排除
- **監査**: すべての操作を CloudTrail に記録、S3 アーカイブと CloudTrail Insights を併用
- **将来検討**: AWS Network Firewall（ネットワーク監査強化時）

**補足図：ログ相関フロー**

![ログ相関図 - Proxy/ICAP/VPN/CWAgent → CloudWatch Logs → OpenSearch / Alarm / S3 → Athena](./images/05_log_correlation_flow.svg)

Proxy / ICAP / VPN / CloudWatch Agent からのログを CloudWatch Logs に集約し、Subscription Filter で OpenSearch Service へ配信、CloudWatch Alarm で異常検知、S3 へ長期保管して Athena で SQL 分析、SSM Session Manager の操作証跡は CloudTrail 経由で S3 に集約する流れです。

---

## 7. EC2 インスタンス選定とコスト最適化方針

*1行サマリ: ここはすべて初期検証時の仮説。実測に基づき後で更新する。*

> **本章は初期検証時の仮説であり、CloudWatch 等での監視結果に基づき見直しを行います。**

| コンポーネント | 初期候補 | 起動方針 | 選定理由 | 監視項目 | 見直し条件 |
|---|---|---|---|---|---|
| Proxy3（入口・認証） | t3.small または t3.medium | 常時起動 | 認証・入口制御を担当、AD 連携時の CPU / メモリ想定 | CPU / メモリ / 認証失敗数 | レイテンシ悪化、ピーク時のリソース不足 |
| Proxy2（中継・ICAP連携） | t3.small または t3.medium | 常時起動 | 経路分岐・ICAP 連携、ICAP 待ちが性能要因 | レイテンシ / ICAP 待ち / メモリ | ICAP 連携の応答遅延、検査スループット不足 |
| Proxy1（出口） | t3.small | 常時起動 | 出口制御中心、ACL 評価が主処理 | 通信量 / 拒否ログ / NAT 利用率 | 通信量増加、リソース不足 |
| ICAP / ClamAV | t3.medium 以上 | 常時起動 | ClamAV はメモリを多く使用、定義ファイル更新が走る | メモリ使用率 / 定義ファイル更新 / スキャン遅延 | スキャン遅延が増大した場合 |
| NAT Gateway | 初期 1 台 | 常時起動 | コスト優先、可用性要件に応じ AZ ごと配置 | バイトアウト / エラー率 / ポート使用率 | AZ 障害許容を上げる場合は 2 台化 |
| AD Connector | small / large | 常時起動 | VPN 経由で既存 Samba AD/DC を参照 | 接続失敗 / Kerberos エラー | ユーザー数増、レイテンシ悪化 |
| AWS Client VPN | 利用人数で段階拡張 | 常時起動 | リモート接続の終端 | 同時接続数 / 認証失敗数 | リモート利用者の増減 |
| OpenSearch Service | 採用（B 案） | 常時起動（小規模クラスター） | ログ分析・全文検索・ダッシュボード | 検索レイテンシ / ストレージ使用率 | クエリ性能不足、データ量増 |
| Network Firewall / Managed Microsoft AD / Auto Scaling | 段階導入（将来検討） | - | 監査・ID 基盤・需要増の要件発生時に追加 | - | 要件発生時 |

**コスト最適化方針**

- 初期は **1AZ + NAT Gateway 1 台** で検証
- 要件に応じて **2AZ + NAT 2 台** へ段階拡張
- **S3 ライフサイクル**（Standard → IA → Glacier）でログ保管コストを抑制
- **OpenSearch のサイジング**は小規模クラスターから開始、検索量に応じて拡張
- **Network Firewall / Managed AD は段階導入** — 初期は NACL / AD Connector で代替
- PoC では **夜間停止 / Instance Scheduler** も検討
- ただし Proxy 経路は利用時間帯の通信前提となるため、**標準運用では常時起動を基本とする**

---

## 8. 承認付き IaC 展開フロー（検討）

*1行サマリ: 自動化の目的は楽をすることではなく、安全・レビュー可能・再現性高い展開。*

**設計原則**

- **コード化対象**: VPC / Subnet / Route Table / Security Group / EC2 / IAM / CloudWatch を **Terraform** または **CloudFormation** で記述
- **起動方式**: GitHub Actions の **`workflow_dispatch`** で手動起動
- **plan → review → apply**: `terraform plan` の差分を Artifact / PR コメントで提示し、**人間が差分を確認してから apply**
- **顧客環境への展開**: **Cross Account Role + External ID** を検討、root ユーザや永続アクセスキーは使わない
- **監査**: すべての操作を **CloudTrail** で記録、S3 アーカイブ
- **構築後チェック**: SSM Automation Runbook で疎通・ログ・Alarm 状態を確認し、Markdown レポート化

**ステップ案**

| Step | 操作 | 担当 | 出力 |
|---|---|---|---|
| 1 | workflow_dispatch でジョブ起動 | 依頼者 | Run ID |
| 2 | terraform plan を実行 | GitHub Actions | plan.txt / Markdown 要約 |
| 3 | plan を Artifact / PR コメントへ | GitHub Actions | レビュー対象 |
| 4 | 差分レビュー & 承認 | 人間（複数名推奨） | Approve |
| 5 | terraform apply（AssumeRole + External ID） | GitHub Actions | apply ログ |
| 6 | SSM Automation で構築後チェック | SSM Automation | テスト結果 JSON |
| 7 | CloudWatch Logs / Alarm 状態を確認 | SSM Automation | Markdown レポート |
| 8 | レポートを Artifact / Slack 通知 | GitHub Actions | レビュー証跡 |

**補足図：IaC 展開フロー**

![IaC展開フロー - 依頼者 → GitHub Actions（terraform plan）→ 人間レビュー → apply (AssumeRole + ExternalID) → SSM Automation 構築後チェック → Markdown レポート](./images/06_iac_deployment_flow.svg)

依頼者の `workflow_dispatch` 起動 → `terraform plan` → 人間レビューと承認 → AssumeRole + External ID で `terraform apply` → SSM Automation で構築後チェック → Markdown 検証レポート出力までを、すべて CloudTrail で監査記録します。

---

## 9. 今後の検証ロードマップ

*1行サマリ: Phase 1 の設計整理が完了。次は Terraform 最小構成の検証。*

| Phase | 内容 | 成果物 |
|---|---|---|
| 1 | 設計図・対応関係表・判断ドキュメントの整備 | 本ページ（aws-deployment-plan.md） |
| 2 | Terraform 最小構成（VPC / Subnet / Proxy EC2 / NAT GW）を **次フェーズで構築予定** | terraform/ ディレクトリ |
| 3 | CloudWatch Logs / Alarm / SSM 接続検証 | 検証スクショ + Markdown |
| 4 | GitHub Actions workflow_dispatch + plan / apply 分離 | .github/workflows/aws-deploy.yml |
| 5 | クロスアカウント Role + External ID の検証 | role-trust-policy.json + 検証ログ |
| 6 | SSM Automation Runbook で構築後チェックを自動化 | runbook.yaml + サンプルレポート |
| 7 | EC2 インスタンスサイズの実測検証・コストレポート | CloudWatch Dashboard + 月次レポート |
| 8 | OpenSearch / Athena でのログ分析パイプライン検証 | クエリ集 + ダッシュボード |
| 9 | 面接用 1 枚図 / 10 分説明資料化 | interview-pitch.md / PPTX |

> 各 Phase は **次フェーズで検証予定** であり、現時点では Phase 1 完了段階。

---

## 10. 面接での説明ポイント

*1行サマリ: 図 4 枚を順に示し、責務分離と AWS 運用最適化の判断軸を語る。*

このAWS移行設計案で示したいのは、単に AWS サービス名を知っていることではなく、既存の OSS 3 段 Proxy で分離した「通信・認証・暗号化・ログ・監視」の責務を、AWS 上でも **可用性・セキュリティ・コスト・運用性** の観点で再設計できることです。

顧客アカウントへの展開についても、**最小権限・承認・監査・実行後検証** を含む **承認付き IaC 展開フロー** として安全な導入を目指しています。

**話す順序の例（面接 5 分）**

| # | 話す内容 | 所要時間 | 着地点 |
|---|---|---|---|
| 1 | このページは「設計案」「AWS 上の構築は未実施・次フェーズで検証予定」 | 30秒 | 実装段階ではなく **設計判断を整理した段階** であることを明示 |
| 2 | 構成比較図（01）→ B 案を採用した理由 | 1分 | A：単純移植 / C：監査強化との対比でコスト・可用性・運用負荷の判断軸 |
| 3 | 全体構成図（02）→ 3段Proxy で OSS の責務分離を AWS でも維持 | 1分 | VPC / 2AZ / Proxy3→Proxy2→Proxy1 / ICAP / NAT / CW / AD Connector / PAC を一枚図で説明 |
| 4 | レイヤー分解図（04）→ Client / Access / Control / Auth / Proxy / Inspection / Egress / Logging | 45秒 | レイヤ別責務分離と Logging Layer の横断観測 |
| 5 | 設計判断（5-1〜5-7）から 1〜2 点を深掘り | 1分15秒 | 例：「なぜ 3 段を維持するのか」「なぜ Squid TLS で stunnel を外したのか」 |
| 6 | 承認付き IaC 展開フロー | 30秒 | 自動化＝楽ではなく、レビュー可能で再現性の高い展開を目指す |

**SAA / SOA 観点で語れること**

- **SAA（設計）** — VPC / Subnet / SG / NACL の責務分離、IAM Role 最小権限、Multi-AZ、NLB / Route 53 ルーティング選択、CloudWatch / OpenSearch の使い分け、Cost Optimization
- **SOA（運用）** — CloudWatch Logs / Metrics / Alarm を起点とした切り分けフロー、SSM Automation Runbook、Patch / Inventory による運用標準化、S3 ログ保管、障害時の確認手順、権限・監査の運用設計

---

## 関連ドキュメント

- [Index（設計意図・全体構成）](./index.html)
- [Verification（動作証跡）](./verification.html)
- [Automation（自動化・再現性）](./automation.html)
- [面接用ピッチ](./interview-pitch.html)
