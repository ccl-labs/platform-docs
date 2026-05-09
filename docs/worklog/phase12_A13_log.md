# Phase12-A13 作業ログ

| # | 作業内容 |
|---|---|
| 1 | PROJECT.md 更新・不要タスクの削除判断 |
| 2 | Grafana ダッシュボード作成（sample-backend 開発者向け） |
| 3 | Loki ラベル不一致の修正 |
| 4 | Tempo トレース動作確認 |
| 5 | Platform Team ダッシュボード作成 |
| 6 | cert-manager / ArgoCD / ESO の ServiceMonitor 有効化 |

---

## 1. PROJECT.md 更新・不要タスクの削除判断

Phase12-A12 完了に伴い `Phase12-A11まで完了` → `A12まで完了` に更新。また以下2タスクを削除した。

**GitOps PR ワークフロー検証の削除**

タスクの目的はサブアカウントで Golden Path → PR 作成 → 管理者承認のフローを確認することだったが、検討の結果以下の理由で削除した。

- Scaffolder が PR を作るのは `ccl-labs-platform-backstage` App であり、実行ユーザーの GitHub アカウントは関係しない
- PR の承認制御は GitHub ブランチ保護で行うものであり、Backstage 固有の検証ではない
- ポートフォリオ目的の個人環境でサブアカウント作成コストを払う価値がない

**Backstage Permission Framework の削除**

現状の認証は「Keycloak でログインできる人を絞る」入口制御のみで、細かい操作権限制御（カタログ編集・テンプレート実行制限等）の要件がない。用途のない実装は経験値として薄く、「どういう要件で必要になるか判断できる」ことの方がポートフォリオとして価値があると判断した。

---

## 2. Grafana ダッシュボード作成（sample-backend 開発者向け）

「自分のアプリが今どういう状態か」を確認するための開発者向けダッシュボードを作成した。クラスタ概要は kube-prometheus-stack が自動 provisioning する kubernetes-mixin ダッシュボードが存在したため、追加作業なし。

### 構成

| パネル | タイプ | データソース | クエリ |
|---|---|---|---|
| Request Rate | Time series | Prometheus | `rate(http_requests_total{namespace="sample-app"}[5m])` |
| API Request Rate | Stat | Prometheus | `sum(rate(http_requests_total{namespace="sample-app", endpoint=~"/items.*"}[5m])) or vector(0)` |
| Total Requests | Stat | Prometheus | `sum(http_requests_total{namespace="sample-app"})` |
| Application Logs | Logs | Loki | `{namespace="sample-app", app="sample-backend"} \|~ "(?i)warn\|error"` |

Traces は Grafana Explore（Tempo）で確認する方針とし、ダッシュボードには含めなかった（理由は §4 参照）。

---

## 3. Loki ラベル不一致の修正

### 問題

Logs パネルで `{namespace="sample-app", app_kubernetes_io_name="sample-backend"}` とクエリしたところ No data になった。

### 原因

Alloy が Pod からログ収集する際に付与するラベルは `app`（短縮形）であり、`app_kubernetes_io_name` ではなかった。Loki の実際のラベル一覧を API で確認して判明した。

### 対処

クエリを `{namespace="sample-app", app="sample-backend"}` に修正した。

---

## 4. Tempo トレース動作確認

### Traces パネルの Table view 問題

Grafana の Traces パネルで Service Name を `sample-backend` に指定すると No data になる問題が発生した。Table view を ON にすると表示されるが、ダッシュボードに保存すると Table view の状態が保持されない既知の制約があった。

Traces パネルのデフォルト表示は単一トレース ID のウォーターフォール用であり、Search モード（一覧表示）には Table view が必要なため本質的な解決が難しい。

実務でも Tempo は Explore で参照するのが一般的であるため、**Traces はダッシュボードから除外し Explore で確認する方針**とした。

### Tempo の頻繁な再起動

調査中に Tempo が OOMKill で再起動を繰り返している（22回）ことが判明した。メモリ制限（512Mi）が低いためと推測される。今回はダッシュボード作成の範囲外のため未対応。

---

## 5. Platform Team ダッシュボード作成

Platform チームが「ミドルウェアが落ちていないか」を確認するためのダッシュボードを作成した。

### 構成

| パネル | タイプ | クエリ概要 |
|---|---|---|
| Unhealthy Deployments | Table | `kube_deployment_status_replicas_unavailable > 0` からアプリ NS を除外 |
| Problem ArgoCD Applications | Table | Synced でないまたは Healthy でないアプリ |
| Certificate Expiry | Table | cert-manager 証明書の残り日数 |
| ESO Sync Errors | Stat | ESO の reconcile エラー増加数（直近1時間） |

**アプリ NS の除外方法**

namespace を列挙するのはメンテナンス性が悪いため、Golden Path で作成した namespace に付与されている `policy-enforced: "true"` ラベルを利用して除外した。新しいアプリが追加されても自動的に除外対象になる。

```promql
kube_deployment_status_replicas_unavailable > 0
unless on(namespace)
kube_namespace_labels{label_policy_enforced="true"}
```

cert-manager の証明書は `*.platform.local` ワイルドカード1枚のみで全サービスを共有しているため、表示は1行になる。複数証明書が存在する場合は1行ずつ表示される。

---

## 6. cert-manager / ArgoCD / ESO の ServiceMonitor 有効化

ダッシュボード作成中に Prometheus がこれらのコンポーネントを scrape していないことが判明した。各 Helm Chart の values に ServiceMonitor 設定を追加して対応した。

| コンポーネント | 追加した設定 |
|---|---|
| cert-manager | `prometheus.enabled: true` / `prometheus.servicemonitor.enabled: true` |
| ArgoCD | `controller.metrics` / `server.metrics` / `applicationSet.metrics` に `enabled: true` + `serviceMonitor.enabled: true` |
| ESO | `serviceMonitor.enabled: true` |

なお CNPG クラスタ Pod（PostgreSQL インスタンス）のメトリクスも未 scrape であることが判明した。CNPG オペレーターの PodMonitor はあるが、クラスタ Pod（port 9187）を対象にした PodMonitor が存在しない。GitOps 管理での追加方法を検討中のため本セッションでは未対応。

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `~/internal/claude/PROJECT.md` | Phase12-A12 完了更新・2タスク削除 |
| `platform-gitops/platform/applications/root-1-gateway/cert-manager.yaml` | Prometheus ServiceMonitor 有効化 |
| `platform-gitops/platform/argocd/values.yaml` | controller / server / applicationSet の metrics + ServiceMonitor 有効化 |
| `platform-gitops/platform/applications/root-2-auth/external-secrets.yaml` | ServiceMonitor 有効化 |
