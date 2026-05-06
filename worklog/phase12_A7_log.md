# Phase 12 作業ログ（セッション A7）

## 概要

権限制御の実装（タスク①）。Linux ユーザー分離・K8s RBAC・kubeconfig・AppProject・ArgoCD RBAC・Keycloak グループ/ユーザー管理を一気通貫で実装。

---

## 1. Linux ユーザー分離

```bash
sudo useradd -m -s /bin/bash app-developer
sudo passwd app-developer
```

---

## 2. K8s RBAC の実装

`platform-gitops/apps/app-developer-rbac/` を新規作成し、ArgoCD App として管理。

### ファイル構成

**`clusterrole.yaml`**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: app-developer
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
```

**`serviceaccount.yaml`** — namespace: sample-app  
**`rolebinding.yaml`** — namespace: sample-app、subjects namespace: sample-app  
**`token-secret.yaml`** — type: kubernetes.io/service-account-token

### ArgoCD App（`apps/app-developer-rbac.yaml`）

```yaml
spec:
  project: default
  source:
    path: apps/app-developer-rbac
  destination:
    namespace: sample-app
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

> SA を sample-app NS に集約。専用 NS は無駄なため廃止。

---

## 3. kubeconfig 生成・配置

SAトークンを取得して kubeconfig を生成し、app-developer ユーザーのホームに配置。

```bash
# SAトークン取得
TOKEN=$(kubectl get secret app-developer-token -n sample-app -o jsonpath='{.data.token}' | base64 -d)
CA=$(kubectl get secret app-developer-token -n sample-app -o jsonpath='{.data.ca\.crt}')

# kubeconfig 生成
kubectl config set-cluster dev --server=https://0.0.0.0:6443 --certificate-authority=<(echo $CA | base64 -d) --embed-certs=true --kubeconfig=/tmp/app-developer-kubeconfig
kubectl config set-credentials app-developer --token=$TOKEN --kubeconfig=/tmp/app-developer-kubeconfig
kubectl config set-context default --cluster=dev --user=app-developer --namespace=sample-app --kubeconfig=/tmp/app-developer-kubeconfig
kubectl config use-context default --kubeconfig=/tmp/app-developer-kubeconfig

# 配置
sudo mkdir -p /home/app-developer/.kube
sudo cp /tmp/app-developer-kubeconfig /home/app-developer/.kube/config
sudo chown -R app-developer:app-developer /home/app-developer/.kube
```

---

## 4. AppProject `sample-apps` の作成

**`platform-gitops/apps/sample-apps-project.yaml`**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: sample-apps
  namespace: argocd
spec:
  sourceRepos:
    - git@github.com:okccl/sample-backend.git
    - git@github.com:okccl/sample-frontend.git
    - git@github.com:okccl/platform-charts.git
    - git@github.com:okccl/platform-gitops.git
  destinations:
    - namespace: sample-app
      server: https://kubernetes.default.svc
  clusterResourceWhitelist: []
```

sample-backend / sample-frontend の `spec.project` を `default` → `sample-apps` に変更。

---

## 5. HTTPRoute を sample-app NS に移動

AppProject のデスティネーション制限（namespace: sample-app のみ許可）により、envoy-gateway-system に置いていた HTTPRoute が Permission Error になった。

**変更ファイル**
- `apps/sample-backend/manifests/httproute.yaml` — namespace: envoy-gateway-system → sample-app
- `apps/sample-frontend/manifests/httproute.yaml` — 同上
- `apps/sample-backend/manifests/reference-grant.yaml` — 削除（同一 NS になったため不要）

> Gateway が gatewayClassName で envoy-gateway を参照するため、HTTPRoute は Gateway と別 NS でも動作する。ReferenceGrant は不要になる。

---

## 6. sample-app NS の明示管理

`CreateNamespace=true` を削除し、PE 管理の `apps/sample-app-ns.yaml` で明示管理。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sample-app
  labels:
    goldilocks.fairwinds.com/enabled: "true"
    policy-enforced: "true"
```

---

## 7. app-developer 用 mise.toml

PE 用（`platform-infra/.mise.toml`）と別に、アプリ開発者向けの定義を各リポジトリに配置。

**`sample-backend/.mise.toml`** / **`sample-frontend/.mise.toml`**
```toml
[tools]
kubectl = "1.35.3"
argocd  = "3.2.9"
gh      = "2.87.2"
"github:hatoo/oha" = "1.14.0"
```

> 以前は platform-infra/.mise.toml へのシンボリックリンクだったが、扱うツールが異なるため実ファイルに変換。

app-developer ユーザーでのインストール:
```bash
su - app-developer
cd ~/sample-backend
mise install
```

---

## 8. Keycloak config-cli 更新

### グループ・ユーザー追加

**`platform-gitops/platform/keycloak/config-cli/realm-configmap.yaml`**
- `argocd-admins` グループを `platform-team` / `app-developer` に置き換え
- `platform-admin` ユーザーのグループを `platform-team` に変更
- `app-developer` ユーザーを新規追加（パスワードは `$(APP_DEVELOPER_PASSWORD)` 参照）

### シークレット追加

- `platform-gitops/platform/keycloak/config-cli/external-secret.yaml` に `APP_DEVELOPER_PASSWORD` 追加
- `platform-gitops/platform/keycloak/config-cli-values.yaml` の extraEnv に `APP_DEVELOPER_PASSWORD` 追加
- `platform-gitops/platform/secrets/sources/secret-generator.yaml` に `keycloak-app-developer-source.yaml` 追加
- `keycloak-app-developer-source.yaml` をユーザーが別ターミナルで SOPS 暗号化して作成

### NPE 対応

config-cli v6.5.0 + Keycloak 26 の UPDATE パスで `ClientScopeRepresentation.getId()` が NPE になる問題が発生。

**原因**: defaultClientScopes / optionalClientScopes の更新時にスコープ解決が失敗する。

**対応**: クライアント定義から `defaultClientScopes` / `optionalClientScopes` を一旦完全削除してレルムデフォルトに委ねたうえで、クリーンな状態から `optionalClientScopes: [groups]` のみを追加。

### ignoreDifferences の整理

ExternalSecret の `force-sync` annotation が OutOfSync を引き起こし Helm PostSync hook が実行されない問題を解消。

**`platform-gitops/platform/applications/root-2-auth/keycloak-config-cli.yaml`**
```yaml
ignoreDifferences:
  - group: external-secrets.io
    kind: ExternalSecret
    jqPathExpressions:
      - .metadata.annotations["force-sync"]
      - .spec.data[].remoteRef.conversionStrategy
      - .spec.data[].remoteRef.decodingStrategy
      - .spec.data[].remoteRef.metadataPolicy
      - .spec.data[].remoteRef.nullBytePolicy
```

### イメージタグ固定

```yaml
image:
  tag: "6.5.0-26.5.4"  # latest-26 から固定
```

---

## 9. ArgoCD RBAC 更新

**`platform-gitops/platform/argocd/values.yaml`**
```yaml
rbac:
  policy.csv: |
    p, role:platform-team, applications, *, */*, allow
    p, role:platform-team, repositories, *, *, allow
    p, role:app-developer, applications, get, sample-apps/*, allow
    g, platform-team, role:platform-team
    g, app-developer, role:app-developer
  policy.default: role:''
  scopes: "[groups]"
```

> `policy.default: role:readonly` では app-developer が全アプリを閲覧できてしまうため `role:''` に変更。

---

## 10. Keycloak SSO 動作確認

- `platform-admin` で ArgoCD にログイン → platform-team ロールで全操作可能 ✅
- `app-developer` で ArgoCD にログイン → sample-apps のアプリのみ閲覧可能、操作不可 ✅

---

## 最終状態

- K8s RBAC: sample-app NS に ClusterRole / SA / RoleBinding / SAToken を配置 ✅
- kubeconfig: app-developer ユーザーのホームに配置済み ✅
- AppProject `sample-apps`: 作成済み ✅
- Keycloak: platform-team / app-developer グループ、各ユーザーを config-cli で自動管理 ✅
- ArgoCD RBAC: グループベースで権限制御 ✅
- SSO ログイン: 動作確認済み ✅

---

## 関連ファイル一覧

| ファイル | 変更内容 |
|---------|---------|
| `platform-gitops/apps/app-developer-rbac.yaml` | 新規（ArgoCD App） |
| `platform-gitops/apps/app-developer-rbac/` | 新規（ClusterRole / SA / RoleBinding / Token） |
| `platform-gitops/apps/sample-apps-project.yaml` | 新規（AppProject） |
| `platform-gitops/apps/sample-app-ns.yaml` | 新規（sample-app NS 明示管理） |
| `platform-gitops/apps/sample-backend.yaml` | project: sample-apps、CreateNamespace 削除 |
| `platform-gitops/apps/sample-frontend.yaml` | project: sample-apps |
| `platform-gitops/apps/sample-backend/manifests/httproute.yaml` | namespace → sample-app |
| `platform-gitops/apps/sample-frontend/manifests/httproute.yaml` | namespace → sample-app |
| `platform-gitops/apps/sample-backend/manifests/reference-grant.yaml` | 削除 |
| `platform-gitops/platform/argocd/values.yaml` | RBAC policy 更新、policy.default → role:'' |
| `platform-gitops/platform/keycloak/config-cli/realm-configmap.yaml` | グループ・ユーザー追加、optionalClientScopes 追加 |
| `platform-gitops/platform/keycloak/config-cli/external-secret.yaml` | APP_DEVELOPER_PASSWORD 追加 |
| `platform-gitops/platform/keycloak/config-cli-values.yaml` | APP_DEVELOPER_PASSWORD extraEnv 追加、imageタグ固定 |
| `platform-gitops/platform/secrets/sources/secret-generator.yaml` | keycloak-app-developer-source 追加 |
| `platform-gitops/platform/secrets/sources/keycloak-app-developer-source.yaml` | 新規（SOPS暗号化・ユーザー作成） |
| `platform-gitops/platform/applications/root-2-auth/keycloak-config-cli.yaml` | ignoreDifferences 整理 |
| `sample-backend/.mise.toml` | symlink → 実ファイル（アプリ開発者向けツール定義） |
| `sample-frontend/.mise.toml` | 同上 |
