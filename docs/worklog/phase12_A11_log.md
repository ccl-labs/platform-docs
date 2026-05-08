# Phase 12-A11 作業ログ（F-3 Golden Path 着手・ArgoCD GitHub App 認証移行・Kyverno RBAC 修正）

## 概要

| # | 作業内容 |
|---|---|
| 1 | F-3: Kyverno generate で ghcr-pull-secret を NS ラベルトリガーで自動配布（暫定 ExternalSecret を廃止） |
| 2 | ArgoCD: platform-secrets-sources の OutOfSync 修正（ESO デフォルト補完フィールドの ignoreDifferences） |
| 3 | ArgoCD: リポジトリ認証を SSH から GitHub App 方式（HTTPS credential template）に移行 |
| 4 | 【トラブル】全 ArgoCD Application の repoURL を SSH→HTTPS に一括変換 |
| 5 | Runbook-001: 新規シークレット追加手順の誤りを修正（encrypt-first→sops edit 順序） |
| 6 | Kyverno: backgroundController に Secret RBAC を Helm values 経由で追加 |

---

## 1. F-3: Kyverno generate による ghcr-pull-secret 自動配布

### 背景

`sample-app` namespace への `ghcr-pull-secret` 配布は、前フェーズで ExternalSecret を手動追加する暫定対応（`apps/sample-backend/manifests/ghcr-pull-secret.yaml`）で行っていた。F-3 Golden Path の目標は「namespace に特定ラベルを貼るだけで Secret が自動配布される」ことであり、Kyverno の `generate` + `clone` 方式で本対応を実施した。

### 設計判断

Secret の生成元（clone source）を `platform-secrets` namespace の Secret とし、そこへは ExternalSecret 経由で OnePassword から値を注入する構成とした。Kyverno の `clone` は実 Secret を参照するため、`platform-secrets` に実態 Secret が必要。

```
1Password (ghcr-pat)
  ↓ ExternalSecret (platform-secrets/ghcr-pull-secret-source.yaml)
platform-secrets/ghcr-pull-secret  ← clone source
  ↓ Kyverno ClusterPolicy (generate-ghcr-pull-secret)
ghcr-pull-secret: enabled ラベルが付いた各 NS
```

当初「`platform/secrets/sources/` には Kubernetes Secret マニフェストしか置けない」と誤認していたが、実際には ExternalSecret も配置可能。そのため clone source も `platform/secrets/sources/` で管理することにした（kustomization.yaml に追加）。

### 実施内容

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

`platform/secrets/sources/ghcr-pull-secret-source.yaml` を作成（OnePassword → `platform-secrets/ghcr-pull-secret` Secret を生成する ExternalSecret）し、`kustomization.yaml` に追記。

`apps/sample-app-ns.yaml` に `ghcr-pull-secret: enabled` ラベルを追加。

暫定 ExternalSecret（`apps/sample-backend/manifests/ghcr-pull-secret.yaml`）および孤立ファイル（`platform/secrets/external-secrets/namespace-labels.yaml`）を削除。

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

## 2. platform-secrets-sources の OutOfSync 修正

### 背景

`platform-secrets-sources` ArgoCD Application が常に OutOfSync となっていた。原因は ESO controller が ExternalSecret に以下のデフォルトフィールドを自動補完するため：

```
.spec.data[].remoteRef.conversionStrategy
.spec.data[].remoteRef.decodingStrategy
.spec.data[].remoteRef.metadataPolicy
.spec.data[].remoteRef.nullBytePolicy
.spec.target.creationPolicy
.spec.target.deletionPolicy
```

`platform/applications/root-2-auth/platform-secrets-sources.yaml` の `ignoreDifferences` に上記 jqPathExpressions を追加し、`RespectIgnoreDifferences=true` を syncOptions に追加して解消した。

---

## 3. ArgoCD リポジトリ認証を GitHub App 方式に移行

### 背景

ArgoCD GUI で "Unable to load data: error acquiring repo lock..." エラーが継続発生。原因は `okccl` → `ccl-labs` org 移行時に SSH デプロイキーが引き継がれず、ArgoCD が Git リポジトリにアクセスできなかったこと。

GitHub App 認証（HTTPS credential template 方式）に移行することにした。

### 実施手順

**GitHub App 情報の確認**（ArgoCD が Backstage 用に使っている App を共用）：
- App ID: GitHub GUI で確認
- Installation ID: `gh api /orgs/ccl-labs/installations --jq '.installations[] | select(.app_id == <APP_ID>) | .id'`

`secrets/templates/argocd-repo-creds.yaml` に credential template Secret のひな形を作成（.gitignore 対象ディレクトリ）：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-repo-creds-ccl-labs
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  type: git
  url: https://github.com/ccl-labs/
  githubAppID: "3638469"
  githubAppInstallationID: "130426273"
  githubAppPrivateRSAKey: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----
```

ユーザーがひな形に PEM を転記し、SOPS で暗号化：

```bash
# テンプレートを暗号化対象ディレクトリへコピー後、まず暗号化してから編集
sops --encrypt --in-place secrets/encrypted/argocd-repo-creds.yaml
sops secrets/encrypted/argocd-repo-creds.yaml  # PEM を貼り付けて保存
```

bootstrap Makefile を更新：SSH エージェント設定と `argocd repo add` を廃止し、SOPS 復号 + kubectl apply に差し替え：

```makefile
SOPS_AGE_KEY_FILE=$(SOPS_AGE_KEY) sops decrypt $(SECRETS_DIR)/argocd-repo-creds.yaml \
    | kubectl apply -f -
```

---

## 4. 【トラブル】全 Application の repoURL を SSH→HTTPS に一括変換

### 問題

credential template を HTTPS URL プレフィックス（`https://github.com/ccl-labs/`）で登録したが、ArgoCD Application の `repoURL` が旧来の SSH 形式（`git@github.com:ccl-labs/...`）のままだった。credential template は URL プレフィックスでマッチングするため、SSH URL の Application には適用されずすべてのリポジトリアクセスが失敗した。

### 原因と反省

SSH→GitHub App 移行時に「credential template の URL 形式と、各 Application の repoURL が一致している必要がある」という点を先に確認せずに進めた。影響範囲を事前調査すべきだった。

### 対処

`platform-gitops` 配下の全ファイルを対象に sed で一括変換（32 ファイル）：

```bash
cd ~/platform-gitops
find . -name "*.yaml" -exec grep -l "git@github.com:ccl-labs/" {} \; \
  | xargs sed -i 's|git@github.com:ccl-labs/|https://github.com/ccl-labs/|g'
```

すでに稼働中の 17 Application は `argocd app set` で即時更新：

```bash
for app in $(argocd app list -o name); do
  argocd app set $app --repo https://github.com/ccl-labs/<該当リポジトリ>.git
done
```

```bash
git add -A
git commit -m "fix: ArgoCD Application の repoURL を SSH から HTTPS に移行"
git push
```

---

## 5. Runbook-001 の手順誤りを修正

### 問題

`sops secrets/encrypted/argocd-repo-creds.yaml` を実行したところ `sops metadata not found` エラーが発生。

### 原因

テンプレートをコピーした直後のファイルは SOPS メタデータを持たない。`sops <file>`（編集モード）はメタデータが存在することを前提とするため、このエラーが出る。

### 対処

Runbook-001「新規シークレットの追加」セクションの手順を修正：

- 修正前: `sops <file>` で直接編集
- 修正後: まず `sops --encrypt --in-place <file>` でメタデータを生成、その後 `sops <file>` で編集

注意書きも追加：「テンプレートをコピーした直後は SOPS メタデータがないため `sops <file>` は "sops metadata not found" エラーになる」

---

## 6. Kyverno backgroundController に Secret RBAC を追加

### 背景

`kyverno-policies` ArgoCD Application が `generate-ghcr-pull-secret` ClusterPolicy の apply 時にエラーを出し続けていた。Kyverno の admission webhook が ClusterPolicy を検証する際、`kyverno-background-controller` SA が Secret の `list/get` 権限を持たないと判定してポリシー作成を拒否した。

### 原因

Kyverno の `generate` + `clone` ルールは background-controller が Secret を操作するため、対応する RBAC が必要。デフォルトの Helm chart では Secret へのアクセス権が付与されていない。

### 対処方針の選択

最初に ClusterRole + ClusterRoleBinding を手動で作成したが、ユーザーレビューにより「スコープが広すぎる・ArgoCD 管理外になる」として差し戻し。Helm values 経由が自然と判断し、`kyverno.yaml` の Helm values に `backgroundController.rbac.clusterRole.extraResources` を追加する方法を採用した。

`platform/applications/root-2-auth/kyverno.yaml` の helm.values を更新：

```yaml
backgroundController:
  replicas: 1
  rbac:
    clusterRole:
      extraResources:
        - apiGroups:
            - ""
          resources:
            - secrets
          verbs:
            - get
            - list
            - watch
            - create
            - update
            - patch
            - delete
```

```bash
cd ~/platform-gitops
git add platform/applications/root-2-auth/kyverno.yaml
git commit -m "fix: kyverno backgroundController に Secret RBAC を追加"
git push
```

ArgoCD が `kyverno` app を sync すると background-controller の ClusterRole に権限が追加され、`generate-ghcr-pull-secret` ポリシーのエラーが解消される。

---

## 関連ファイル一覧

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
