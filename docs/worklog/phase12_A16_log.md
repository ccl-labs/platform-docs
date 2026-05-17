# Phase12-A16 作業ログ

| # | 作業内容 |
|---|---|
| 1 | CoreDNS rewrite の事前確定（Envoy Gateway SVC 名固定） |
| 2 | DOCS_RULE.md 修正（設計書の保存場所ルール更新） |
| 3 | bootstrap 設計書の新規作成 |
| 4 | bootstrap 単一ルート集約（wave 番号付け替え・root.yaml 作成・旧3ルート削除・Makefile 更新） |

---

## 1. CoreDNS rewrite の事前確定

### 背景

bootstrap 単一ルート集約の前提として、`*.platform.local` の CoreDNS rewrite ルールを bootstrap 前の時点で確定できるようにする必要があった。

従来の `bootstrap-sync` では、Envoy Gateway が Gateway リソースの apply 後に動的生成する Service 名（`envoy-envoy-gateway-system-eg-<ハッシュ>`）を `kubectl get svc -l owning-gateway-name=eg` で取得してから CoreDNS を更新していた。この動的取得処理は ArgoCD の wave 自動進行に割り込めないため、単一ルート化の障害になっていた。

Envoy Gateway v1.1 以降で利用可能な `EnvoyProxy` リソースの `spec.provider.kubernetes.envoyService.name` を使い、Service 名を `envoy-eg` に固定することで解決した。

### 実施内容

`platform/gateway/config/envoy-proxy-config.yaml` を新規作成し、`gateway.yaml` に `infrastructure.parametersRef` を追加した。

```yaml
# 新規: envoy-proxy-config.yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: eg-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        name: envoy-eg
```

```yaml
# gateway.yaml に追加
spec:
  infrastructure:
    parametersRef:
      group: gateway.envoyproxy.io
      kind: EnvoyProxy
      name: eg-proxy-config
```

`k3d/Makefile` の `fix-coredns` ターゲットに `*.platform.local` rewrite ルールを追加し、`bootstrap-sync` の動的 CoreDNS パッチ処理を削除した。

```makefile
cf=d['data']['Corefile']; \
rule='    rewrite name keycloak.platform.local envoy-eg.envoy-gateway-system.svc.cluster.local\n...'; \
d['data']['Corefile']=cf.replace('ready\n', 'ready\n'+rule) if 'rewrite name keycloak.platform.local' not in cf else cf; \
```

push 後に ArgoCD が gateway-config を自動 sync し、旧 SVC（ハッシュ付き）が削除されて `envoy-eg` が作成された。`make fix-coredns` を手動実行して現クラスタの CoreDNS を更新し、`argocd.platform.local` / `keycloak.platform.local` の疎通を確認した。

---

## 2. DOCS_RULE.md 修正

設計書の保存場所ルールが実態（`platform-docs/docs/design/`）と乖離していたため修正した。

- 旧: 設計ドキュメントは `~/internal/claude/tmp_<テーマ>.md`
- 新: 正式設計書は `platform-docs/docs/design/<テーマ>-design.md`。設計メモ（tmp ファイル）のセクションは削除。

章立て書式（`## 1.` / `### 1.1`）と、ステータス行は作成中のときのみ記載するルールを追記した。

---

## 3. bootstrap 設計書の新規作成

改修後の bootstrap 構成を `platform-docs/docs/design/bootstrap-design.md` として新規作成した。

記載内容：

- **ルート App 構成**：単一 Application の YAML 定義・ディレクトリ構造
- **Wave 設計**：全 37 Application の wave 番号一覧（0–4 / 10–15 / 20–22）
- **カスタムヘルスチェック**：定義一覧と各チェックが保証する内容
- **Bootstrap フロー**：`make bootstrap` のステップ表と `bootstrap-sync` の詳細
- **CoreDNS 設計**：`*.platform.local` 事前解決の仕組み

---

## 4. bootstrap 単一ルート集約

### wave 番号付け替え

グループ間を 10 刻みで分離し、root-2-auth（旧 1–6 → 新 10–15）・root-3-others（旧 1–3 → 新 20–22）を更新した。`user-apps.yaml` / `user-apps-project.yaml` には wave アノテーション自体がなかったため wave 20 を追加した。

### 単一ルート root.yaml 作成

`platform/group-roots/root.yaml` を新規作成。`platform/applications` を `recurse: true` で参照し、`vcluster-dev.yaml` のみ exclude する構成。

```yaml
directory:
  recurse: true
  exclude: 'root-3-others/vcluster-dev.yaml'
```

### 移行操作

```bash
kubectl apply -f ~/platform-gitops/platform/group-roots/root.yaml
argocd app delete root-1-gateway --cascade=false
argocd app delete root-2-auth    --cascade=false
argocd app delete root-3-others  --cascade=false
```

`--cascade=false` により子 App（cert-manager・keycloak 等）を残したまま旧3ルートを削除した。root App が自動 sync して Synced + Healthy になったことを確認後、旧3ルートの YAML ファイルを削除した。

### Makefile 更新

`bootstrap-sync` を3ルートの順次 apply から単一ルートの apply + wait に置き換えた。

```makefile
kubectl apply -f ~/platform-gitops/platform/group-roots/root.yaml
argocd app sync root --server-side || true
# ... Envoy Gateway 待機・ログイン切替 ...
argocd app wait root --sync --health --timeout 1800
```

---

---

## 5. bootstrap テスト・タイミング問題の修正

### 問題 1：wave 12 App（external-secrets-config）の初回 sync 失敗

**原因**：ESO の webhook Pod がエンドポイントに登録される前に wave 12 の sync が走り、`ClusterSecretStore` apply 時に `no endpoints available for service "external-secrets-webhook"` が発生。

**対処**：`external-secrets-config` と `kyverno-policies`（ともに wave 12）に retry ポリシーを追加。

```yaml
syncPolicy:
  retry:
    limit: 20
    backoff:
      duration: 30s
      factor: 2
      maxDuration: 10m
```

### 問題 2：keycloak-db の CNPG Cluster が作成されない

**原因**：`ClusterSecretStore` 未作成の状態で wave 13 が走り、ExternalSecret が "could not get secret data from provider" になった。その後 retry ポリシーが 4 回目で成功したが、keycloak App wait の 900 秒 timeout 以内に収まらなかった。

**根本対処**：全 ExternalSecret を `platform/secrets/config/` に集中管理する方針に変更。`external-secrets-config`（wave 12）が ClusterSecretStore と全 ExternalSecret を一括 apply することで、wave 13 以降の App 起動時には必ず Secret が存在する状態を保証する。

同様に `keycloak-db` / `backstage-db` にも retry ポリシーを追加（暫定対処として残存）。

### 問題 3：bootstrap-sync の完了条件が分かりにくい

`argocd app wait root`（全 37 App 待機）から、ArgoCD 起動時・Keycloak 起動時の2段階マイルストーン表示に変更。

```
=== ArgoCD 起動完了 ===
URL: https://argocd.platform.local
Username: admin
Password: xxxxxxxx

=== bootstrap 完了 ===
ArgoCD URL:   https://argocd.platform.local
Keycloak URL: https://keycloak.platform.local
```

---

## 6. applications/ フラット化

`platform/applications/root-1-gateway/`・`root-2-auth/`・`root-3-others/` のサブディレクトリを廃止し、全 38 ファイルを `platform/applications/` 直下にフラット化。

`root.yaml` の exclude を `root-3-others/vcluster-dev.yaml` → `vcluster-dev.yaml` に修正。関連ドキュメント（bootstrap-design.md・DR-design.md・Runbook-001）のパス参照も更新。

---

## 7. ExternalSecret 集中管理とディレクトリ構造整理

### 方針決定

アプリ固有の ExternalSecret をアプリ App に持たせる設計は fresh bootstrap 時のタイミング問題を引き起こすため、全 ExternalSecret を `platform/secrets/config/` に集中管理する方針に変更。

### 移動対象

| 元パス | 移動先 |
|---|---|
| `keycloak/db-config/external-secret.yaml` | `secrets/config/keycloak-db-credentials.yaml` |
| `keycloak/db-config/minio-backup-secret.yaml` | `secrets/config/keycloak-minio-backup.yaml` |
| `keycloak/config-cli/external-secret.yaml` | `secrets/config/keycloak-config-cli-secret.yaml` |
| `backstage/db-config/external-secret.yaml` | `secrets/config/backstage-db-credentials.yaml` |
| `backstage/db-config/minio-backup-secret.yaml` | `secrets/config/backstage-minio-backup.yaml` |
| `backstage/app-config/external-secret.yaml` | `secrets/config/backstage-secret.yaml` |
| `backstage/app-config/ghcr-pull-secret.yaml` | `secrets/config/backstage-ghcr-pull-secret.yaml`・`secrets/generators/backstage-ghcr-token.yaml` に分割 |
| `monitoring/alerts/discord-webhook-secret.yaml` | `secrets/config/discord-webhook-secret.yaml` |

ExternalSecret の wave は 4（ClusterSecretStore wave 3 の後）に統一。Namespace も `secrets/namespaces/` に分離。

### secrets/ ディレクトリ最終構造

```
secrets/
  config/      # ClusterSecretStore + ExternalSecret（wave 3–4）
  generators/  # ESO Generator（GithubAccessToken 等）（wave 5–6）
  namespaces/  # wave 12 に必要な事前 Namespace（README で意図を説明）
  sources/     # SOPS 暗号化 source secrets
```

`external-secrets-config` App を multi-source（config・generators・namespaces の3パス）に変更。

### 実装方針として PROJECT.md に記録

- **1 ファイル 1 リソース種別**（`---` 複数定義は同一 kind のみ）
- **ExternalSecret 集中管理**（`platform/secrets/config/` に一元化）
- **Generator 集中管理**（`platform/secrets/generators/` に一元化）

---

## 8. kind 混在 YAML のチェックと分離

全リポジトリ（platform-gitops・platform-infra・apps-gitops・platform-charts）を対象に「1ファイル1リソース種別」方針に違反する YAML を検索し、対象3ファイルを分離した。

| ファイル | 混在 kind | 対処 |
|---|---|---|
| `crossplane/config/provider.yaml` | `DeploymentRuntimeConfig` + `Provider` | `provider-helm-runtime.yaml` に分離 |
| `cert-manager/config/cluster-issuer.yaml` | `ClusterIssuer` + `Certificate` | `certificate.yaml` に分離 |
| `crossplane/config/rbac.yaml` | `ServiceAccount` + `ClusterRoleBinding` | `service-account.yaml` + `cluster-role-binding.yaml` に分割・旧ファイル削除 |

`catalog-info.yaml`・`backstage/org.yaml`（Backstage Entity）および `gateway/envoy-gateway/manifests.yaml`（vendor manifest）は対象外と判断。

---

## 9. apps-root.yaml の整理

`platform-gitops/bootstrap/apps-root.yaml` の問題を発見・修正した。

- **repoURL バグ**：`platform-gitops.git` の `apps/` を参照していたが、該当ディレクトリは存在しない。正しくは `apps-gitops.git`
- **配置場所**：`bootstrap-apps` ターゲットから apply するため `platform-infra/k3d/` に移動
- `platform-gitops/bootstrap/` ディレクトリは削除（`.gitkeep` のみだったため）

---

## 10. bootstrap-apps の統合と user-apps 整理

### bootstrap-apps を make bootstrap に統合

`bootstrap-sync` が Keycloak 起動まで待機するようになったため、`bootstrap-apps` を `bootstrap` target に追加。ESO・CNPG が確実に稼働済みの状態でアプリ層を展開できる。

`user-apps-infra`（`sample-app` Namespace + `sample-apps` AppProject）の稼働確認後に `apps-root` を apply するよう `bootstrap-apps` に `argocd app wait user-apps-infra` を追加。

### user-apps / apps-root の一本化

`user-apps.yaml`（wave 20）が `apps-gitops/apps/*/application.yaml` を管理しており、`apps-root.yaml` と役割が重複していた。また wave 20 でアプリが展開されるため「platform 安定後にアプリを展開する」方針が実現できていなかった。

- `platform/applications/user-apps.yaml` / `user-apps-project.yaml` を削除（`user-apps` AppProject はどのアプリも参照していないことを確認済み）
- `apps-root.yaml` を `recurse: true` + `include: '*/application.yaml'` に修正（`user-apps` と同等の動作）
- `user-apps-infra.yaml` は platform 側リソース（Namespace ラベル・AppProject）のため platform-gitops に残存

Makefile のターゲット定義順も実行順（bootstrap-argocd → bootstrap-sync → bootstrap-apps）に整列。

---

## 11. Trivy 修正

### Helm values キー名バグ修正

`excludeNamespaces` と `concurrentScanJobsLimit` のキー名が誤っており、設定が一切反映されていなかった。

| 修正前 | 修正後 |
|---|---|
| `operator.excludeNamespaces` | `excludeNamespaces`（トップレベル） |
| `operator.concurrentScanJobsLimit: 1` | `operator.scanJobsConcurrentLimit: 1` |
| `operator.concurrentNodeScanners: 1` | 削除（存在しないキー） |

### wave 20 → 23 に変更

他コンポーネント起動後にスキャンが始まるよう wave を後退させた。

---

## 12. ESO webhook 競合対策

### 問題

`external-secrets-config`（wave 12）の retry policy が `duration: 30s, factor: 2` の指数バックオフであり、ESO webhook の endpoint 登録が間に合わない場合に最大数分待機が発生していた。

### 対処

wave を1つずらして `kyverno-policies`（wave 12）を自然なバッファとして活用し、retry を線形短縮した。

| App | 変更前 | 変更後 |
|---|---|---|
| `external-secrets-config` | wave 12、30s/factor:2/max:10m/limit:20 | wave 13、10s/factor:1/max:10s/limit:60 |
| `keycloak-db` | wave 13 | wave 14 |
| `keycloak` | wave 14 | wave 15 |
| `keycloak-config-cli` / `keycloak-routes` | wave 15 | wave 16 |

retry limit を 20（200秒）から 60（600秒）に増加。初回 bootstrap テストで limit 20 到達による sync error が発生したため修正。

---

## 13. bootstrap テスト結果

2回のbootstrap実行で以下を確認・修正した。

- `external-secrets-config` sync error → retry limit 不足（200秒）が原因。limit を 60（600秒）に修正
- `backstage` Pod が起動時に `backstage-db-rw` を DNS 解決できず 503 → 手動 restart で解消。根本対応（Dockerfile に `pg_isready` wait を追加）は次セッションへ持ち越し
- `argocd` App の OutOfSync（ServiceMonitor 未作成）は automated sync で自己解決

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-gitops/platform/gateway/config/envoy-proxy-config.yaml` | 新規作成：EnvoyProxy リソース（SVC 名 `envoy-eg` に固定） |
| `platform-gitops/platform/gateway/config/gateway.yaml` | `infrastructure.parametersRef` 追加 |
| `platform-gitops/platform/applications/root-2-auth/*.yaml` | sync-wave: 1–6 → 10–15 |
| `platform-gitops/platform/applications/root-3-others/*.yaml` | sync-wave: 1–3 → 20–22、user-apps 2ファイルに wave 20 追加 |
| `platform-gitops/platform/group-roots/root.yaml` | 新規作成：単一ルート App-of-Apps |
| `platform-gitops/platform/group-roots/root-{1,2,3}-*.yaml` | 削除（単一ルート移行完了） |
| `platform-infra/k3d/Makefile` | fix-coredns に rewrite 追加、bootstrap-sync を単一ルート対応・マイルストーン表示に更新 |
| `platform-gitops/platform/applications/external-secrets-config.yaml` | retry ポリシー追加・multi-source 化 |
| `platform-gitops/platform/applications/kyverno-policies.yaml` | retry ポリシー追加 |
| `platform-gitops/platform/applications/keycloak-db.yaml` | retry ポリシー追加・ExternalSecret ignoreDifferences 削除 |
| `platform-gitops/platform/applications/backstage-db.yaml` | retry ポリシー追加・ExternalSecret ignoreDifferences 削除 |
| `platform-gitops/platform/applications/*.yaml`（38 ファイル） | サブディレクトリから直下にフラット化 |
| `platform-gitops/platform/secrets/config/`（新規 10 ファイル） | ExternalSecret 集中管理 |
| `platform-gitops/platform/secrets/generators/` | GithubAccessToken を分離 |
| `platform-gitops/platform/secrets/namespaces/` | Namespace を分離・README 追加 |
