# Phase12-A12 作業ログ

| # | 作業内容 |
|---|---|
| 1 | Golden Path Kyverno監視設定の実装済み確認・持ち越し課題から削除 |
| 2 | Backstage Teardown Template 設計書の更新 |
| 3 | Backstage Teardown Template 実装 |
| 4 | `github:actions:dispatch` 403 の調査・修正 |
| 5 | GitHub リポジトリ削除 403 の調査・原因特定 |
| 6 | GitHub App 設計思想の整理・teardown.yaml の修正 |
| 7 | 秘密鍵フォーマットエラーの修正 |
| 8 | 動作確認完了・設計書最終更新 |
| 9 | PROJECT.md 更新・不要タスクの削除判断 |
| 10 | Grafana ダッシュボード作成（sample-backend 開発者向け） |
| 11 | Loki ラベル不一致の修正 |
| 12 | Tempo トレース動作確認 |
| 13 | Platform Team ダッシュボード作成 |
| 14 | cert-manager / ArgoCD / ESO の ServiceMonitor 有効化 |
| 15 | Platform Team ダッシュボード Pod Restart Count パネル追加 |
| 16 | ダッシュボード JSON の GitOps 管理（ConfigMap 化） |
| 17 | Loki / Tempo datasource UID 固定化 |
| 18 | Backstage への Grafana ダッシュボード統合 |
| 19 | Backstage Home ページ整理 |
| 20 | apps-gitops リポジトリ分離 |
| 21 | README.md 更新 |

---

## 1. Golden Path Kyverno監視設定の実装済み確認・持ち越し課題から削除

持ち越し課題「アプリ開発者が使用するNSに `policy-enforced: "true"` ラベルを自動付与して Kyverno 監視対象にする」の実装状況を確認。

確認の結果、以下がすべて実装済みであることを確認した。

| 確認項目 | 状況 |
|---|---|
| Kyverno ClusterPolicy 3本（`require-resource-limits` / `add-default-labels` / `disallow-latest-tag`）の `namespaceSelector.matchLabels: policy-enforced: "true"` 設定 | 実装済み |
| `apps/sample-app-ns.yaml` へのラベル付与 | 実装済み |
| Scaffolder fullstack テンプレートの `namespace.yaml` へのラベル付与 | 実装済み |

`PROJECT.md` の持ち越し課題から当該項目を削除した。

---

## 2. Backstage Teardown Template 設計書の更新

前セッションで作成した設計書（`~/internal/claude/tmp_backstage_teardown_template.md`）を実装前にレビューし、以下を修正した。

| 修正箇所 | 内容 |
|---|---|
| Scaffolderステップ | `workflowInputs` に `confirmName` が含まれていなかったため追加 |
| teardown.yaml サンプル | トークン生成ステップが抜けていた・順序ミスを修正。`confirmName` チェックステップ・空コミット対策（`git diff --cached --quiet \|\|`）を追加 |
| 実装時の考慮事項 | GitHub App 権限の懸念項目を「確認済み」に更新 |
| 残課題セクション | 両項目解決済みのため削除 |

---

## 3. Backstage Teardown Template 実装

設計書に従い以下3ファイルを実装した。

**新規作成:**
- `platform-gitops/backstage/templates/teardown/template.yaml` — Scaffolder テンプレート（`github:actions:dispatch` で teardown.yaml を dispatch）
- `platform-gitops/.github/workflows/teardown.yaml` — 削除 GitHub Actions ワークフロー

**変更:**
- `platform-gitops/platform/backstage/values.yaml` — teardown テンプレートの catalog location を追加

```yaml
# platform/backstage/values.yaml に追加
- type: url
  target: https://github.com/ccl-labs/platform-gitops/blob/main/backstage/templates/teardown/template.yaml
  rules:
    - allow: [Template]
```

---

## 4. `github:actions:dispatch` 403 の調査・修正

### 問題

Backstage Scaffolderから Teardown テンプレートを実行すると以下のエラーが発生した。

```
Failed: dispatching workflow 'teardown.yaml' on repo: 'platform-gitops',
Resource not accessible by integration
```

### 原因

`github:actions:dispatch` アクションは Backstage の統合 GitHub App（`ccl-labs-platform-backstage`）のトークンを使って `workflow_dispatch` イベントを送信する。この App に `Actions: write` 権限がなかった。

`publish:github:pull-request` は動作していたため `pull-requests: write` はあったが、ワークフロー dispatch には別途 `Actions: write` が必要だった。

### 対処

GitHub → Developer Settings → `ccl-labs-platform-backstage` → Permissions & events → Repository permissions → **Actions: Read & write** を追加し、org の Install 画面で Accept した。

---

## 5. GitHub リポジトリ削除 403 の調査・原因特定

### 問題

Actions 権限追加後に再実行すると `github:actions:dispatch` は成功したが、teardown.yaml 内のリポジトリ削除ステップで 403 が発生した。

```
HTTP 403: Resource not accessible by integration
(https://api.github.com/repos/ccl-labs/test-app-backend)
```

### 原因の特定

デバッグステップを追加してトークンの権限を確認した。

```yaml
- name: トークン権限確認（デバッグ用）
  run: |
    gh api /repos/ccl-labs/${{ github.event.inputs.appName }}-backend --jq '.permissions' || true
  env:
    GH_TOKEN: ${{ steps.token.outputs.token }}
```

結果:

```json
{"admin":false,"maintain":false,"pull":false,"push":false,"triage":false}
```

`pull: false` を含む全権限が `false` だった。teardown.yaml はもともと `update-gitops.yaml` のパターンを踏襲して `GITOPS_APP_CLIENT_ID`（`ccl-labs-gitops` App）を使っていたが、この App は `platform-gitops` への push 専用として設計されており、app repos（`test-app-backend` 等）にはインストールされていなかった。

---

## 6. GitHub App 設計思想の整理・teardown.yaml の修正

### 背景

`ccl-labs-gitops` を teardown に流用したことが根本的な設計ミスだった。環境内の GitHub App はアクターで役割分担されている。

| App | アクター | 役割 |
|---|---|---|
| `ccl-labs-gitops` | CI/CD 自動化 | コード変更に反応して継続的に動く。platform-gitops への push のみ |
| `ccl-labs-platform-backstage` | Platform（Backstage） | 開発者向けライフサイクル管理。repo 作成・削除・カタログ・PR |

Teardown は「Platform がアプリのライフサイクルを終わらせる」操作であるため、全ステップ `ccl-labs-platform-backstage` を使うのが設計思想として一貫している。

### 対処

**GitHub Actions org への credential 追加（GUI操作）:**
- Variables に `BACKSTAGE_APP_CLIENT_ID: Iv23li36ymucOKubrBmg` を追加（All repositories）
- Secrets に `BACKSTAGE_APP_PRIVATE_KEY` を追加（All repositories）
  - 値は SOPS 復号して取得: `sops -d ~/platform-gitops/platform/secrets/sources/backstage-github-app-source.yaml`

**teardown.yaml の修正:**

```yaml
# 変更前
client-id: ${{ vars.GITOPS_APP_CLIENT_ID }}
private-key: ${{ secrets.GITOPS_APP_PRIVATE_KEY }}

# 変更後
client-id: ${{ vars.BACKSTAGE_APP_CLIENT_ID }}
private-key: ${{ secrets.BACKSTAGE_APP_PRIVATE_KEY }}
```

デバッグステップも同時に削除した。

---

## 7. 秘密鍵フォーマットエラーの修正

### 問題

`BACKSTAGE_APP_PRIVATE_KEY` を登録して再実行すると、トークン生成ステップで以下のエラーが発生した。

```
Failed to create token for "ccl-labs" (attempt 1): error:1E08010C:DECODER routines::unsupported
```

### 原因

GitHub が生成する秘密鍵は PKCS#8 形式（`-----BEGIN PRIVATE KEY-----`）だが、`actions/create-github-app-token` が期待するのは PKCS#1 形式（`-----BEGIN RSA PRIVATE KEY-----`）だった。

### 対処

以下のコマンドで PKCS#1 形式に変換し、改めて `BACKSTAGE_APP_PRIVATE_KEY` に登録した（GUI操作）。

```bash
sops -d ~/platform-gitops/platform/secrets/sources/backstage-github-app-source.yaml \
  | grep -A 1000 "privateKey:" | tail -n +2 \
  | openssl pkcs8 -nocrypt
```

---

## 8. 動作確認完了・設計書最終更新

再実行後のワークフローログで全ステップが正常終了したことを確認した。

| ステップ | 結果 | 備考 |
|---|---|---|
| アプリ名確認 | ✅ | |
| GitHub App トークン生成 | ✅ | `ccl-labs-platform-backstage` で取得 |
| platform-gitops チェックアウト | ✅ | |
| GitOps マニフェスト削除 | ✅ | `Everything up-to-date`（前回ランで削除済みのため差分なし） |
| GitHub リポジトリ削除 | ✅ | エラーなし（前回ランで削除済み） |
| GHCR パッケージ削除 | ✅ | `404 Package not found`（テストアプリのため images 未ビルド。`\|\| true` で正常終了） |

設計書（`tmp_backstage_teardown_template.md`）を最終実装と照合して更新した。`PROJECT.md` の持ち越し課題から Teardown Template を削除した。

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-gitops/backstage/templates/teardown/template.yaml` | 新規作成（Scaffolderテンプレート） |
| `platform-gitops/.github/workflows/teardown.yaml` | 新規作成（削除ワークフロー） |
| `platform-gitops/platform/backstage/values.yaml` | teardown テンプレートの catalog location 追加 |
| `~/internal/claude/tmp_backstage_teardown_template.md` | 設計書を最終実装版に更新（GitHub App 設計思想・秘密鍵形式を追記） |
| `~/internal/claude/PROJECT.md` | Golden Path Kyverno監視設定・Teardown Template を持ち越し課題から削除 |
| `~/internal/claude/PROJECT.md` | Phase12-A12 完了更新・2タスク削除（#9） |
| `platform-gitops/platform/applications/root-1-gateway/cert-manager.yaml` | Prometheus ServiceMonitor 有効化（#14） |
| `platform-gitops/platform/argocd/values.yaml` | controller / server / applicationSet の metrics + ServiceMonitor 有効化（#14） |
| `platform-gitops/platform/applications/root-2-auth/external-secrets.yaml` | ServiceMonitor 有効化（#14） |

---

## 9. PROJECT.md 更新・不要タスクの削除判断

Phase12-A12 完了に伴い `Phase12-A11まで完了` → `A12まで完了` に更新。また以下2タスクを削除した。

**GitOps PR ワークフロー検証の削除**

タスクの目的はサブアカウントで Golden Path → PR 作成 → 管理者承認のフローを確認することだったが、検討の結果以下の理由で削除した。

- Scaffolder が PR を作るのは `ccl-labs-platform-backstage` App であり、実行ユーザーの GitHub アカウントは関係しない
- PR の承認制御は GitHub ブランチ保護で行うものであり、Backstage 固有の検証ではない
- ポートフォリオ目的の個人環境でサブアカウント作成コストを払う価値がない

**Backstage Permission Framework の削除**

現状の認証は「Keycloak でログインできる人を絞る」入口制御のみで、細かい操作権限制御（カタログ編集・テンプレート実行制限等）の要件がない。用途のない実装は経験値として薄く、「どういう要件で必要になるか判断できる」ことの方がポートフォリオとして価値があると判断した。

---

## 10. Grafana ダッシュボード作成（sample-backend 開発者向け）

「自分のアプリが今どういう状態か」を確認するための開発者向けダッシュボードを作成した。クラスタ概要は kube-prometheus-stack が自動 provisioning する kubernetes-mixin ダッシュボードが存在したため、追加作業なし。

### 構成

| パネル | タイプ | データソース | クエリ |
|---|---|---|---|
| Request Rate | Time series | Prometheus | `rate(http_requests_total{namespace="sample-app"}[5m])` |
| API Request Rate | Stat | Prometheus | `sum(rate(http_requests_total{namespace="sample-app", endpoint=~"/items.*"}[5m])) or vector(0)` |
| Total Requests | Stat | Prometheus | `sum(http_requests_total{namespace="sample-app"})` |
| Application Logs | Logs | Loki | `{namespace="sample-app", app="sample-backend"} \|~ "(?i)warn\|error"` |

Traces は Grafana Explore（Tempo）で確認する方針とし、ダッシュボードには含めなかった（理由は §12 参照）。

---

## 11. Loki ラベル不一致の修正

### 問題

Logs パネルで `{namespace="sample-app", app_kubernetes_io_name="sample-backend"}` とクエリしたところ No data になった。

### 原因

Alloy が Pod からログ収集する際に付与するラベルは `app`（短縮形）であり、`app_kubernetes_io_name` ではなかった。Loki の実際のラベル一覧を API で確認して判明した。

### 対処

クエリを `{namespace="sample-app", app="sample-backend"}` に修正した。

---

## 12. Tempo トレース動作確認

### Traces パネルの Table view 問題

Grafana の Traces パネルで Service Name を `sample-backend` に指定すると No data になる問題が発生した。Table view を ON にすると表示されるが、ダッシュボードに保存すると Table view の状態が保持されない既知の制約があった。

Traces パネルのデフォルト表示は単一トレース ID のウォーターフォール用であり、Search モード（一覧表示）には Table view が必要なため本質的な解決が難しい。

実務でも Tempo は Explore で参照するのが一般的であるため、**Traces はダッシュボードから除外し Explore で確認する方針**とした。

### Tempo の頻繁な再起動

調査中に Tempo が OOMKill で再起動を繰り返している（22回）ことが判明した。メモリ制限（512Mi）が低いためと推測される。今回はダッシュボード作成の範囲外のため未対応。

---

## 13. Platform Team ダッシュボード作成

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

## 14. cert-manager / ArgoCD / ESO の ServiceMonitor 有効化

ダッシュボード作成中に Prometheus がこれらのコンポーネントを scrape していないことが判明した。各 Helm Chart の values に ServiceMonitor 設定を追加して対応した。

| コンポーネント | 追加した設定 |
|---|---|
| cert-manager | `prometheus.enabled: true` / `prometheus.servicemonitor.enabled: true` |
| ArgoCD | `controller.metrics` / `server.metrics` / `applicationSet.metrics` に `enabled: true` + `serviceMonitor.enabled: true` |
| ESO | `serviceMonitor.enabled: true` |

なお CNPG クラスタ Pod（PostgreSQL インスタンス）のメトリクスも未 scrape であることが判明した。CNPG オペレーターの PodMonitor はあるが、クラスタ Pod（port 9187）を対象にした PodMonitor が存在しない。GitOps 管理での追加方法を検討中のため本セッションでは未対応。

---

## 15. Platform Team ダッシュボード Pod Restart Count パネル追加

Platform Team ダッシュボードに Pod 再起動数パネルを追加した。Tempo OOM（22回再起動）のような問題を早期発見するために有用。

```promql
sum by (pod, namespace) (
  kube_pod_container_status_restarts_total
  unless on(namespace) kube_namespace_labels{label_policy_enforced="true"}
) > 5
```

Transformations: Labels to fields → Merge → Organize fields。閾値 10 以上で赤背景（Cell display mode: Color background）。

---

## 16. ダッシュボード JSON の GitOps 管理（ConfigMap 化）

Grafana で手動作成したダッシュボードを GitOps 管理に移行した。`grafana_dashboard: "1"` ラベルを付与した ConfigMap を platform-gitops に追加し、Grafana sidecar が自動検出する仕組みを利用。

| ダッシュボード | ConfigMap ファイル |
|---|---|
| Platform Quick View | `platform/monitoring/dashboards/platform-quick-view.yaml` |
| sample-app Quick View | `platform/monitoring/dashboards/sample-app-quick-view.yaml` |

ArgoCD Application `grafana-dashboards` を `root-3-others` に追加し、dashboards ディレクトリを管理対象とした。

Certificate Expiry パネルの `legendFormat` を固定文字列 `Days Until Expiry` に変更した。元の Grafana 生成 JSON は pod 名（`cert-manager-67647c794b-2wtt8`）を含む列名だったため pod 再起動後に壊れるパターンだった。

---

## 17. Loki / Tempo datasource UID 固定化

Grafana を再デプロイすると `additionalDataSources` の UID が変わり、ダッシュボードの Loki パネルが壊れる問題を防ぐため、`monitoring/values.yaml` に固定 UID を追加した。

```yaml
- name: Loki
  uid: loki
- name: Tempo
  uid: tempo
```

ダッシュボード JSON 側も `"uid": "P8E80F9AEF21F6940"` → `"uid": "loki"` に更新した。

---

## 18. Backstage への Grafana ダッシュボード統合

### 構成

- **Home ページ**：`GrafanaDashboardWidget` コンポーネントを追加。Platform Quick View を iframe で表示。
- **EntityPage**：`GrafanaDashboardCard` コンポーネントを追加。`grafana/dashboard-url` アノテーションが付いた service エンティティに「Monitoring」タブを表示。

sample-backend の `catalog-info.yaml` にアノテーションを追加した。

```yaml
grafana/dashboard-url: https://grafana.platform.local/d/adm2mk6/sample-app-quick-view
```

### Grafana 設定

iframe 埋め込みのため `monitoring/values.yaml` に以下を追加した。

```yaml
grafana.ini:
  security:
    allow_embedding: true
  auth.anonymous:
    enabled: true
    org_role: Viewer
```

### Backstage CSP 修正

Backstage のデフォルト CSP が `frame-src` を `default-src 'self'` から継承して外部 iframe を全ブロックしていた。`backstage/values.yaml` に以下を追加して解決した。

```yaml
csp:
  frame-src: ["'self'", 'https://grafana.platform.local']
```

---

## 19. Backstage Home ページ整理

Grafana ダッシュボードで ArgoCD 異常検知が確認できるようになったため、Home ページから以下を削除した。

- `ArgoCDWidget`
- `HomePageRequestedReviewsCard`（GitHub PR レビュー依頼）
- `HomePageYourOpenPullRequestsCard`（自分の PR 一覧）

Home ページは「クイックリンク + Grafana Platform Quick View」のみのシンプルな構成になった。

---

## 20. apps-gitops リポジトリ分離

`platform-gitops` に混在していたミドルウェア設定とアプリ GitOps マニフェストを分離した。

| リポジトリ | 管理対象 |
|---|---|
| `platform-gitops` | ミドルウェア（ArgoCD・cert-manager・monitoring 等） |
| `apps-gitops`（新規） | アプリ GitOps マニフェスト（Scaffolder 生成・Teardown 対象） |

**分離の理由**：Scaffolder が環境を払い出すたびに `platform-gitops` のコミット履歴が汚れる。アクセス制御・sync ポリシーも分離できる。

**変更ファイル：**

| ファイル | 変更内容 |
|---|---|
| `platform/applications/root-3-others/user-apps.yaml` | `repoURL` を `apps-gitops` に変更 |
| `backstage/templates/fullstack/template.yaml` | PR 送信先を `apps-gitops` に変更 |
| `.github/workflows/teardown.yaml` | チェックアウト先を `apps-gitops` に変更（`repository: ccl-labs/apps-gitops` 追加） |
| `apps/`（削除） | `apps-gitops` リポジトリへ移行 |

---

## 21. README.md 更新

`ccl-labs/.github` の profile README に `apps-gitops` リポジトリを追加。CI/CD フロー図を `platform-gitops`（ミドルウェア）と `apps-gitops`（アプリ）に分離した構成に更新した。

---

## 変更ファイル一覧（#15〜#21）

| ファイル | 変更内容 |
|---|---|
| `platform-gitops/platform/monitoring/dashboards/platform-quick-view.yaml` | 新規作成（Platform Quick View ConfigMap） |
| `platform-gitops/platform/monitoring/dashboards/sample-app-quick-view.yaml` | 新規作成（sample-app Quick View ConfigMap） |
| `platform-gitops/platform/applications/root-3-others/grafana-dashboards.yaml` | 新規作成（grafana-dashboards ArgoCD Application） |
| `platform-gitops/platform/monitoring/values.yaml` | Grafana allow_embedding・anonymous auth・Loki/Tempo UID 固定化 |
| `platform-gitops/platform/backstage/values.yaml` | CSP に frame-src 追加 |
| `platform-gitops/platform/applications/root-3-others/user-apps.yaml` | repoURL を apps-gitops に変更 |
| `platform-gitops/backstage/templates/fullstack/template.yaml` | PR 送信先を apps-gitops に変更 |
| `platform-gitops/.github/workflows/teardown.yaml` | チェックアウト先を apps-gitops に変更 |
| `platform-gitops/.github/workflows/update-gitops.yaml` | チェックアウト先を apps-gitops に変更 |
| `platform-gitops/apps/`（削除） | apps-gitops へ移行 |
| `apps-gitops/.github/workflows/auto-merge-app-pr.yaml` | 新規作成（platform-gitopsからコピー） |
| `backstage/packages/app/src/components/home/GrafanaDashboardWidget.tsx` | 新規作成 |
| `backstage/packages/app/src/components/home/HomePage.tsx` | Grafana ウィジェット追加・不要ウィジェット削除 |
| `backstage/packages/app/src/components/catalog/GrafanaDashboardCard.tsx` | 新規作成 |
| `backstage/packages/app/src/components/catalog/EntityPage.tsx` | Monitoring タブ追加 |
| `sample-backend/catalog-info.yaml` | grafana/dashboard-url アノテーション追加 |
| `ccl-labs/.github/profile/README.md` | apps-gitops 追加・CI/CD フロー図更新 |
