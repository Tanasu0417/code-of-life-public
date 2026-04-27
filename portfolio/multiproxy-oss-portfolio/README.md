# 多段フォワードプロキシシステム構築ポートフォリオ

金融系ネットワーク基盤の運用経験をもとに、**通信経路・認証・暗号化・ログ・監視を OSS で再設計し、障害時の切り分け観点を説明できる形に整理したポートフォリオ**です。

具体的には、多段プロキシ・認証・暗号化・ログ基盤を OSS で一から構築し、
**設計意図と動作証跡をログ根拠で説明できる状態**までまとめています。

▶ **公開ページ**: <https://tanasu0417.github.io/code-of-life-public/portfolio/multiproxy-oss-portfolio/>

---

## 採用担当の方へ：閲覧時間別ガイド

| 時間 | 見ていただきたい場所 | 何が分かるか |
|---|---|---|
| 30秒 | この README の下記「ひと目で分かる構成」 | 何を作ったか・どんな技術スタックか |
| 3分 | [Verification（動作証跡）](https://tanasu0417.github.io/code-of-life-public/portfolio/multiproxy-oss-portfolio/verification.html) | 設計どおりに動いている証拠（スクショ／ログ） |
| 10分 | [Index（設計意図・全体構成）](https://tanasu0417.github.io/code-of-life-public/portfolio/multiproxy-oss-portfolio/) | 設計判断の理由・苦労した点・解決アプローチ |
| 追加 | [AWS 展開案](https://tanasu0417.github.io/code-of-life-public/portfolio/multiproxy-oss-portfolio/aws-deployment-plan.html) | オンプレ／OSS から AWS への移行設計の考え方 |
| 面接 | [面接用ピッチ](https://tanasu0417.github.io/code-of-life-public/portfolio/multiproxy-oss-portfolio/interview-pitch.html) | 10分プレゼン用の構成と話す順序 |

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

- **[Index](https://tanasu0417.github.io/code-of-life-public/portfolio/multiproxy-oss-portfolio/)** — 設計意図・全体構成・苦労した点
- **[Verification](https://tanasu0417.github.io/code-of-life-public/portfolio/multiproxy-oss-portfolio/verification.html)** — 動作証跡（スクショ／ログによる裏取り）
- **[Automation](https://tanasu0417.github.io/code-of-life-public/portfolio/multiproxy-oss-portfolio/automation.html)** — 構築・検証・復旧の自動化（再現性）
- **[AWS 展開案](https://tanasu0417.github.io/code-of-life-public/portfolio/multiproxy-oss-portfolio/aws-deployment-plan.html)** — クラウド移行設計の考え方（計画段階）
- **[面接用ピッチ](https://tanasu0417.github.io/code-of-life-public/portfolio/multiproxy-oss-portfolio/interview-pitch.html)** — 10分プレゼン用の構成

---

## 注記

- 本ポートフォリオは **個人OSS開発（学習・検証目的）** です
- 商用設定のコピーは行わず、**構造と責務から自作** しています
- 機密情報・顧客名・実環境情報は一切含みません
