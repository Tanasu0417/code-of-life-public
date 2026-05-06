# multiproxy-oss-portfolio

金融系ネットワーク運用で扱った多段プロキシ・認証・ログ基盤を、OSSで再設計し、約25分で再構築可能な検証環境として実装しました。

通信経路・認証・暗号化・ログのどこで問題が起きているかを、ログ根拠で切り分けて説明できる構成です。

▶ 公開ページ:
https://tanasu0417.github.io/code-of-life-public/

---

## 閲覧時間別ガイド

| 時間 | 見ていただきたい場所 | 何が分かるか |
|---|---|---|
| 30秒 | この README の下記「ひと目で分かる構成」 | 何を作ったか・どんな技術スタックか |
| 3分 | [Verification（動作証跡）](https://tanasu0417.github.io/code-of-life-public/verification.html) | 設計どおりに動いている証拠（スクショ／ログ） |
| 10分 | [Index（設計意図・全体構成）](https://tanasu0417.github.io/code-of-life-public/) | 設計判断の理由・苦労した点・解決アプローチ |
| 追加 | [AWS 移行設計案](https://tanasu0417.github.io/code-of-life-public/aws-deployment-plan.html) | OSS 基盤を AWS へ移行する場合の設計案（Phase 1/2/3 段階導入・実機未実施） |
| 補足 | [ポートフォリオ説明用スクリプト](https://tanasu0417.github.io/code-of-life-public/interview-pitch.html) | 口頭で説明する際のスピーカーノート |

### 発表資料（PPTX）

短時間で全体像を把握できる発表資料を同梱しています。

| 見たい内容 | 資料 | 目安 |
|---|---|---|
| まず概要を知りたい | [5分概要版](./Mulutiproxy_System_AWS_5min%20_ver.2.pptx) | 全体像・代表的なログ切り分け・再現性を短く確認 |
| 詳細まで確認したい | [10分詳細版](./Mulutiproxy_System_AWS_10min_ver3.pptx) | 設計判断・3段Proxy責務分離・認証/DNS/PAC・ログ/監視・AWS移行設計案まで |

---

## ひと目で分かる構成

- **題材**: 実務で扱う多段プロキシ構成（入口／分岐／出口の3段）を OSS で再現
- **目的**: 「動かす」ではなく「**通信・認証・暗号化・ログをレイヤ単位で切り分けて説明できる**」ことを目指す
- **使用OSS**: Squid / stunnel / OpenLDAP / Samba AD/DC / Kerberos / dnsmasq / Promtail / Loki / Graylog / OpenSearch / Zabbix / ICAP / ClamAV
- **環境**: WSL2（mirrored mode）× Docker × VMware × Ubuntu 24.04
- **再現性**: 環境破棄から構築・検証完了まで **約25分** で再現できる自動化スクリプト群あり

---

## このポートフォリオで示したいこと

実務では障害解析がベンダー回答に依存しやすく、設計意図や原因を自分の言葉で説明しづらい場面があります。
そこで、**入口〜出口までを OSS で構築し、挙動をログ根拠で説明できる状態**を作りました。

具体的に強化したのは以下の観点です。

- **責務分離**: 入口／分岐／出口・認証・暗号化・検査・ログを分解
- **設計判断**: SSLBump の制約を踏まえた復号ポイントの分離、stunnel による中継暗号化責務の分離
- **可観測性**: access.log（経路）→ cache.log（原因）の切り分けフローを Loki / Graylog で実装
- **再現性**: STEP0〜17 の自動化スクリプトで環境を破棄しても同等状態へ復元

---

## 主要ドキュメント

- **[Index](https://tanasu0417.github.io/code-of-life-public/)** — 設計意図・全体構成・苦労した点
- **[Verification](https://tanasu0417.github.io/code-of-life-public/verification.html)** — 動作証跡（スクショ／ログによる裏取り）
- **[Automation](https://tanasu0417.github.io/code-of-life-public/automation.html)** — 構築・検証・復旧の自動化（再現性）
- **[AWS 移行設計案](https://tanasu0417.github.io/code-of-life-public/aws-deployment-plan.html)** — OSS 基盤を AWS へ移行する場合の設計案。中小規模を想定した **Phase 1（最小）→ Phase 2（B案標準）→ Phase 3（将来拡張）** の段階導入計画、Well-Architected 6 本柱、承認付き IaC 展開フローまで整理（**設計段階・AWS 上の実機構築は未実施**。次フェーズで Terraform / CloudFormation / SSM Automation による IaC 化を検証予定）
- **[ポートフォリオ説明用スクリプト](https://tanasu0417.github.io/code-of-life-public/interview-pitch.html)** — 5分概要版・10分詳細版に対応するスピーカーノート

---

## 注記

- 本ポートフォリオは **個人OSS開発（学習・検証目的）** です
- 商用設定のコピーは行わず、**構造と責務から自作** しています
- 機密情報・顧客名・実環境情報は一切含みません
