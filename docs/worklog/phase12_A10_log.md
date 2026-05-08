# Phase 12-A10 作業ログ（CI/CD GitHub App移行・クラスター障害対応・ArgoCD OutOfSync修正）

## 概要

| # | 作業内容 |
|---|---|
| 1 | CI/CD: GITOPS_TOKEN を GitHub App トークン方式に移行（3リポジトリ） |
| 2 | sample-frontend: `packages: write` 権限不足の修正 |
| 3 | 【トラブル】クラスター再起動後の Cilium eBPF スタルルーティング障害対応 |
| 4 | sample-backend: Rollout のイメージ更新・カナリアプロモーション |
| 5 | ArgoCD: HTTPRoute・ExternalSecret の OutOfSync 修正 |
| 6 | Makefile: クラスター安全再起動手順の整備 |

---

## 1. CI/CD: GitHub App トークン方式への移行

前セッションで GitHub org を `okccl` → `ccl-labs` に移行した際、CI/CD で使用していた `GITOPS_TOKEN`（fine-grained PAT）が旧 org 向けであり `ccl-labs/platform-gitops` へのアクセス権がなかった。PAT 再発行ではなく GitHub App トークン方式（`actions/create-github-app-token@v3.1.1`）に直接移行した。

**GUI操作（ユーザーが実施）**: GitHub App を新規作成（Repository permissions: Contents/PRs Read & write）し、`ccl-labs` org にインストール。org レベルで `GITOPS_APP_CLIENT_ID`（Variable）と `GITOPS_APP_PRIVATE_KEY`（Secret）を登録。

`sample-backend`・`sample-frontend` の `build.yaml` では `trigger-update` ジョブにトークン生成ステップを追加し、`repository_dispatch` の Authorization ヘッダーを差し替えた。`platform-gitops` の `update-gitops.yaml` では、`actions/checkout` の `token` と `gh pr merge` の `GH_TOKEN` を差し替えた。

```bash
cd ~/sample-backend
git add .github/workflows/build.yaml
git commit -m "ci: GitHub App トークン方式に移行"
git push

cd ~/sample-frontend
git add .github/workflows/build.yaml
git commit -m "ci: GitHub App トークン方式に移行"
git push

cd ~/platform-gitops
git add .github/workflows/update-gitops.yaml
git commit -m "ci: GitHub App トークン方式に移行"
git push
```

---

## 2. sample-frontend: `packages: write` 権限不足の修正

CI/CD を動かしたところ sample-frontend の Actions で `denied: installation not allowed to Create organization package` が発生した。`sample-backend` には `permissions: packages: write` があったが `sample-frontend` には `permissions` ブロック自体が欠落していた。`build.yaml` に追記してコミット。

```bash
cd ~/sample-frontend
git add .github/workflows/build.yaml
git commit -m "ci: packages: write 権限を追加"
git push
```

---

## 3. 【トラブル】クラスター再起動後の Cilium eBPF 障害対応

前夜に `k3d cluster stop dev` でクラスターを停止してから WSL をシャットダウンし、翌朝起動したところ Kyverno Webhook タイムアウト（`context deadline exceeded`）と ArgoCD Redis 接続タイムアウト（`dial tcp 10.43.13.19:6379: i/o timeout`）が発生した。

**根本原因**: `k3d cluster stop` は Docker コンテナを停止するだけで Kubernetes の graceful shutdown を経由しない。Cilium の eBPF ルーティングマップはLinuxカーネルに保持される（コンテナ外）ため、再起動後に pod の veth インターフェース（ifindex）が変わっても古いマップが残存する。ClusterIP 経由の通信が古いエンドポイントにルーティングされてタイムアウトしていた。port-forward は eBPF マップを経由しないため正常動作していた。

`kube-system` Namespace は Kyverno Webhook 対象外のため、Cilium DaemonSet を直接再起動することで eBPF マップが現在のインターフェース情報で再構築された。Cilium 復旧後、ArgoCD Redis・ESO・Kyverno Webhook はすべて自動回復した。

```bash
kubectl rollout restart daemonset cilium -n kube-system
kubectl rollout status daemonset cilium -n kube-system --timeout=120s
```

---

## 4. sample-backend: Rollout イメージ更新・カナリアプロモーション

CI/CD 復旧後、`platform-gitops` の values.yaml は `ghcr.io/ccl-labs/sample-backend:d094371` に更新済みだったが、ArgoCD の server-side apply が Rollout リソースに対してno-op となっており、クラスター上のイメージが旧イメージ `ghcr.io/okccl/sample-backend:ca2e588` のままだった（Cilium 障害中に sync が詰まっていたことが原因）。`kubectl argo rollouts set image` で直接更新し、canary pod が Running になったことを確認してプロモーションした。

```bash
kubectl argo rollouts set image sample-backend -n sample-app \
  sample-backend=ghcr.io/ccl-labs/sample-backend:d094371

kubectl argo rollouts promote sample-backend -n sample-app
```

---

## 5. ArgoCD: HTTPRoute・ExternalSecret の OutOfSync 修正

`sample-backend` / `sample-frontend` の ArgoCD Application が HTTPRoute と ExternalSecret で OutOfSync が続いていた。Gateway API controller と ESO controller がデフォルト値をフィールドに補完するが、git 上のマニフェストにそれらが書かれていなかったことが原因。

まず `ignoreDifferences` で ExternalSecret のデフォルト補完フィールドをカバーし、あわせて HTTPRoute 用のエントリも追加した。`sample-backend.yaml` と `sample-frontend.yaml` を修正してプッシュ。ExternalSecret は Synced になったが HTTPRoute は OutOfSync のままだった。

```bash
cd ~/platform-gitops
git add apps/sample-backend.yaml apps/sample-frontend.yaml
git commit -m "fix: HTTPRoute・ExternalSecret のデフォルト補完フィールドを ignoreDifferences に追加"
git push
```

HTTPRoute は `jqPathExpressions` だけでは `backendRefs` 側の差分がカバーしきれていなかったため、マニフェストにデフォルト値（`parentRefs[].group/kind`・`backendRefs[].group/kind/weight`）を明示して根本解消した。

```bash
git add apps/sample-backend/manifests/httproute.yaml apps/sample-frontend/manifests/httproute.yaml
git commit -m "fix: HTTPRoute に Gateway API のデフォルト補完フィールドを明示"
git push

kubectl annotate application sample-backend sample-frontend \
  -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

ArgoCD GUI で両 Application が Synced / Healthy になったことを確認。

---

## 6. Makefile: クラスター安全再起動手順の整備

Cilium の eBPF 問題を踏まえ、`k3d cluster start` 後に必ず Cilium を再起動する手順を `platform-infra/Makefile` に追加した。以降は `make cluster-start` / `make cluster-restart` を使用する。

```bash
cd ~/platform-infra
git add Makefile
git commit -m "chore: クラスター安全再起動ターゲットを Makefile に追加"
git push
```

---

## 関連ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `sample-backend/.github/workflows/build.yaml` | GitHub App トークン方式に移行 |
| `sample-frontend/.github/workflows/build.yaml` | GitHub App トークン方式に移行・`permissions` 追加 |
| `platform-gitops/.github/workflows/update-gitops.yaml` | GitHub App トークン方式に移行 |
| `platform-gitops/apps/sample-backend.yaml` | HTTPRoute・ExternalSecret の `ignoreDifferences` 追加 |
| `platform-gitops/apps/sample-frontend.yaml` | HTTPRoute の `ignoreDifferences`・`RespectIgnoreDifferences=true` 追加 |
| `platform-gitops/apps/sample-backend/manifests/httproute.yaml` | Gateway API デフォルト値を明示 |
| `platform-gitops/apps/sample-frontend/manifests/httproute.yaml` | Gateway API デフォルト値を明示 |
| `platform-infra/Makefile` | `cluster-start` / `cluster-stop` / `cluster-restart` ターゲット追加 |

---

## 7. F-3: Kyverno generate による ghcr-pull-secret 自動配布

### 背景

`sample-app` namespace への `ghcr-pull-secret` 配布は ExternalSecret を手動追加する暫定対応だった。F-3 Golden Path の目標は「namespace に特定ラベルを貼るだけで Secret が自動配布される」ことであり、Kyverno の `generate` + `clone` 方式で本対応を実施した。

### 設計

```
1Password (ghcr-pat)
  ↓ ExternalSecret (platform-secrets/ghcr-pull-secret-source.yaml)
platform-secrets/ghcr-pull-secret  ← clone source
  ↓ Kyverno ClusterPolicy (generate-ghcr-pull-secret)
ghcr-pull-secret: enabled ラベルが付いた各 NS
```

`platform/policy/policies/generate-ghcr-pull-secret.yaml` を作成：

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-ghcr-pull-secret
spec:
  generateExisting: true
  rules:
    - name: generate-ghcr-pull-secret
      match:
        any:
          - resources:
              kinds:
                - Namespace
              selector:
                matchLabels:
                  ghcr-pull-secret: enabled
      generate:
        apiVersion: v1
        kind: Secret
        name: ghcr-pull-secret
        namespace: "{{request.object.metadata.name}}"
        synchronize: true
        clone:
          namespace: platform-secrets
          name: ghcr-pull-secret
```

clone source 用 ExternalSecret（`platform/secrets/sources/ghcr-pull-secret-source.yaml`）を作成し `kustomization.yaml` に追記。`apps/sample-app-ns.yaml` に `ghcr-pull-secret: enabled` ラベルを追加。暫定 ExternalSecret と孤立ファイルを削除。

```bash
cd ~/platform-gitops
git add platform/policy/policies/generate-ghcr-pull-secret.yaml \
        platform/secrets/sources/ghcr-pull-secret-source.yaml \
        platform/secrets/sources/kustomization.yaml \
        apps/sample-app-ns.yaml
git rm apps/sample-backend/manifests/ghcr-pull-secret.yaml \
       platform/secrets/external-secrets/namespace-labels.yaml
git commit -m "feat: Kyverno generate で ghcr-pull-secret を NS ラベルトリガーで自動配布"
git push
```

---

## 8. platform-secrets-sources の OutOfSync 修正

ESO controller が ExternalSecret にデフォルトフィールドを自動補完することで常に OutOfSync となっていた。`platform/applications/root-2-auth/platform-secrets-sources.yaml` に該当フィールドの `ignoreDifferences` と `RespectIgnoreDifferences=true` を追加して解消した。

---

## 9. 【トラブル】ArgoCD リポジトリ認証を GitHub App 方式に移行

### 問題

ArgoCD GUI で "Unable to load data: error acquiring repo lock..." エラーが継続発生。`okccl` → `ccl-labs` org 移行時に SSH デプロイキーが引き継がれず、ArgoCD が Git リポジトリにアクセスできなくなっていた。

### 対処

GitHub App 認証（HTTPS credential template 方式）に移行した。`secrets/templates/argocd-repo-creds.yaml` にひな形を作成し、ユーザーが PEM を転記して SOPS 暗号化。bootstrap Makefile を SSH エージェント不要の構成に更新。

```bash
# Makefile の bootstrap-argocd で GitHub App credential template を適用
SOPS_AGE_KEY_FILE=$(SOPS_AGE_KEY) sops decrypt $(SECRETS_DIR)/argocd-repo-creds.yaml \
    | kubectl apply -f -
```

### 派生トラブル: 全 Application の repoURL を SSH→HTTPS に一括変換

credential template は URL プレフィックスマッチで動作するため、HTTPS template に対して SSH URL の Application は認証されず全リポジトリアクセスが失敗した。影響範囲を先に確認すべきだった（反省）。

`platform-gitops` 配下 32 ファイルを sed で一括変換し、稼働中 17 Application は `argocd app set` で即時更新：

```bash
find . -name "*.yaml" -exec grep -l "git@github.com:ccl-labs/" {} \; \
  | xargs sed -i 's|git@github.com:ccl-labs/|https://github.com/ccl-labs/|g'

git add -A
git commit -m "fix: ArgoCD Application の repoURL を SSH から HTTPS に移行"
git push
```

---

## 10. Runbook-001: 新規シークレット追加手順の誤りを修正

### 問題

`sops secrets/encrypted/argocd-repo-creds.yaml` を実行したところ `sops metadata not found` エラーが発生。

### 原因と対処

テンプレートをコピーした直後のファイルは SOPS メタデータを持たないため、`sops <file>`（編集モード）は動作しない。正しい手順は「`sops --encrypt --in-place` でメタデータを生成してから `sops <file>` で編集」。Runbook-001 を修正し注意書きを追加した。

---

## 11. Kyverno backgroundController に Secret RBAC を追加

### 背景

`kyverno-policies` Application が `generate-ghcr-pull-secret` ClusterPolicy の apply 時にエラーを出し続けた。Kyverno admission webhook が ClusterPolicy 検証時に `kyverno-background-controller` SA の Secret 権限不足を検出して拒否していた。

### 対処

最初に ClusterRole + ClusterRoleBinding を手動作成したが「スコープが広すぎる・ArgoCD 管理外」としてユーザーレビューで差し戻し。Helm values 経由（`backgroundController.rbac.clusterRole.extraResources`）で追加することにした：

```yaml
# platform/applications/root-2-auth/kyverno.yaml
backgroundController:
  replicas: 1
  rbac:
    clusterRole:
      extraResources:
        - apiGroups: [""]
          resources: [secrets]
          verbs: [get, list, watch, create, update, patch, delete]
```

```bash
cd ~/platform-gitops
git add platform/applications/root-2-auth/kyverno.yaml
git commit -m "fix: kyverno backgroundController に Secret RBAC を追加"
git push
```

---

## 12. 【トラブル】bootstrap 再実行時の ESO webhook 未起動問題

### 問題

bootstrap 再実行で `external-secrets-config`（wave 3）が sync しようとした際、ESO の validation webhook が「no endpoints available」で応答できず `ClusterSecretStore` の作成に失敗。Keycloak の ExternalSecret が全滅し Pod が `CreateContainerConfigError` で起動不能になった。

### 原因

ESO の cert-controller が webhook TLS 証明書を非同期で生成するため、ESO Deployment が `Available` になっても webhook がまだ呼び出せない状態がある。ArgoCD のデフォルト Deployment ヘルスチェックはこのラグをカバーできず、wave 1 完了と判定して wave 3 に進んでしまった。

コミュニティ調査（GitHub issues #2273, #4540 等）で広く知られた問題と確認。ESO 公式の ArgoCD 向けガイドは存在しない。コミュニティの推奨は `ValidatingWebhookConfiguration` の `caBundle` が cert-controller によって注入済みかを確認するカスタムヘルスチェックで、webhook を持つ全 Operator（ESO・Kyverno・CNPG 等）に一本で適用できる。

### 手動復旧

```bash
argocd app sync external-secrets-config --server-side
argocd app sync keycloak-db --server-side
argocd app sync keycloak-config-cli --server-side
kubectl rollout restart statefulset keycloak-keycloakx -n keycloak
```

### 根本対処

`platform/argocd/values.yaml` に `ValidatingWebhookConfiguration` のカスタムヘルスチェックを追加。全 webhook の caBundle 注入を確認してから次の wave に進むようになった。

```yaml
resource.customizations.health.admissionregistration.k8s.io_ValidatingWebhookConfiguration: |
  hs = {}
  if obj.webhooks ~= nil then
    for i, webhook in ipairs(obj.webhooks) do
      if webhook.clientConfig ~= nil then
        if webhook.clientConfig.caBundle == nil or webhook.clientConfig.caBundle == "" then
          hs.status = "Progressing"
          hs.message = "Waiting for CA bundle injection by cert-controller"
          return hs
        end
      end
    end
    hs.status = "Healthy"
    return hs
  end
  hs.status = "Progressing"
  return hs
```

---

## 追加関連ファイル一覧（7〜12）

| ファイル | 変更内容 |
|---|---|
| `platform/policy/policies/generate-ghcr-pull-secret.yaml` | 新規作成: Kyverno generate ClusterPolicy |
| `platform/secrets/sources/ghcr-pull-secret-source.yaml` | 新規作成: clone source 用 ExternalSecret |
| `platform/secrets/sources/kustomization.yaml` | ghcr-pull-secret-source.yaml を resources に追加 |
| `apps/sample-app-ns.yaml` | `ghcr-pull-secret: enabled` ラベルを追加 |
| `apps/sample-backend/manifests/ghcr-pull-secret.yaml` | 削除（暫定対応を廃止） |
| `platform/secrets/external-secrets/namespace-labels.yaml` | 削除（孤立ファイル） |
| `platform/applications/root-2-auth/platform-secrets-sources.yaml` | ESO デフォルト補完フィールドの ignoreDifferences を追加 |
| `platform/applications/root-2-auth/kyverno.yaml` | backgroundController Secret RBAC を Helm values に追加 |
| `secrets/templates/argocd-repo-creds.yaml` | 新規作成: GitHub App credential template ひな形 |
| 全 Application yaml（32 ファイル） | repoURL を SSH→HTTPS に一括変換 |
| `platform-infra/k3d/Makefile` | SSH エージェント廃止、GitHub App credential template を apply するよう変更 |
| `platform-docs/docs/runbook/Runbook-001-secrets-management.md` | 新規シークレット追加手順を修正（encrypt-first→sops edit 順序） |
| `platform/argocd/values.yaml` | `ValidatingWebhookConfiguration` ヘルスチェック追加 |
