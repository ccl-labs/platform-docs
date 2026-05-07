# platform-docs

## このリポジトリについて

ADRや作業手順書など、platformに関係するドキュメント類を保管しています。

---

## ADR一覧

ポートフォリオ全体の技術選定の意思決定を記録しています。
実務経験なども踏まえつつ、「なぜその技術を選んだか」を Context → Options → Decision → Reasons → Consequences の形式で整理し、
構築できるだけでなく設計から判断できることを示す目的で作成しました。

| No. | タイトル | 対象リポジトリ | Status |
|---|---|---|---|
| [ADR-001](docs/adr/ADR-001-gitops-engine.md) | GitOpsエンジンの選択 | platform-gitops / platform-infra | Accepted |
| [ADR-002](docs/adr/ADR-002-bootstrap.md) | bootstrap 順序制御の設計 | platform-gitops / platform-infra | Accepted |
| [ADR-003](docs/adr/ADR-003-helm-library-chart.md) | Kubernetesマニフェスト抽象化方法の選択 | platform-charts / platform-gitops | Accepted |
| [ADR-004](docs/adr/ADR-004-secrets-management.md) | Secret管理戦略の選択 | platform-gitops | Accepted |
| [ADR-005](docs/adr/ADR-005-crossplane.md) | インフラリソース管理の責務分離（Crossplane vs Terraform） | platform-gitops / platform-infra | Draft |
| [ADR-006](docs/adr/ADR-006-postgresql-operator.md) | PostgreSQL Operatorの選択 | platform-gitops / platform-charts | Accepted |
| [ADR-007](docs/adr/ADR-007-mise-tool-sharing.md) | mise ツール定義の共有戦略 | platform-infra / platform-gitops | Draft |

---

## 意思決定の全体像

```
ローカル基盤
└─ ADR-001: ArgoCD を GitOps エンジンに選択
      └─ pull型によるdrift検出・GUI・App of Appsパターン
bootstrap
└─ ADR-002: App-of-Apps 分割と Makefile による順序制御
      └─ CR health check 問題の回避・ArgoCD GUI の早期確保
アプリデプロイ抽象化
└─ ADR-003: Helm Library Chart を抽象化レイヤーに選択
      └─ テンプレート一元管理・ガードレール・Golden Path
セキュリティ
└─ ADR-004: SOPS×Age + ESO を Secret 管理に選択
      └─ Secrets as Code・クラスタ非依存・DR整合性
インフラ管理
└─ ADR-005: Terraform と Crossplane を用途で使い分け（Draft）
      └─ クラスタ基盤はTerraform / 開発者向けリソースはCrossplane
データ管理
└─ ADR-006: CloudNativePG を PostgreSQL Operator に選択
      └─ ライセンス・k8sネイティブ・クラウド親和性
ツール管理
└─ ADR-007: mise ツール定義の共有戦略（Draft）
      └─ platform-infra を source of truth とする管理方針
```

---

## Runbook一覧

プラットフォーム運用中に発生した既知の問題と対処手順を記録しています。

| No. | タイトル | 関連コンポーネント |
|---|---|---|
| [Runbook-001](docs/runbook/Runbook-001-secrets-management.md) | シークレット追加・更新手順 | SOPS / ESO / ArgoCD |

---

## 作業ログ

[worklog](docs/worklog)：各Phaseごとの作業記録

---

## 関連リポジトリ

| リポジトリ | 役割 |
|---|---|
| [platform-infra](https://github.com/ccl-labs/platform-infra) | k3d クラスタ IaC・mise・Terraform（EKS） |
| [platform-gitops](https://github.com/ccl-labs/platform-gitops) | ArgoCD による GitOps 管理 |
| [platform-charts](https://github.com/ccl-labs/platform-charts) | Helm Library Chart（common-app / common-db） |
| [sample-backend](https://github.com/ccl-labs/sample-backend) | FastAPI + PostgreSQL（Golden Path利用例） |
| [sample-frontend](https://github.com/ccl-labs/sample-frontend) | React + Vite（Golden Path利用例） |

ポートフォリオ全体の概要は [ccl-labs](https://github.com/ccl-labs) を参照してください。
