# Phase 2: GitOps & Secrets 作業ログ

## 完了の定義
- ArgoCD が稼働し、`platform-gitops` リポジトリを監視している
- App of Apps パターンで `platform/` 以下が自動同期される
- External Secrets Operator が稼働し、ClusterSecretStore が Valid になっている
- Secret の値が Git に直接置かれていない

---

## Step 2-1: platform-gitopsリポジトリの作成

### 方針
- Privateリポジトリとして作成（後で公開予定）
- ArgoCDからのアクセスはSSH鍵で設定

### 実施内容

```bash
# clone & 初期ディレクトリ構成の作成
git clone git@github.com:okccl/platform-gitops.git
cd platform-gitops
mkdir -p bootstrap platform/ingress platform/monitoring platform/logging platform/secrets platform/policy apps
touch bootstrap/.gitkeep platform/ingress/.gitkeep platform/monitoring/.gitkeep platform/logging/.gitkeep platform/secrets/.gitkeep platform/policy/.gitkeep apps/.gitkeep

cat > README.md << 'EOF'
# platform-gitops
Platform Engineering portfolio — GitOps management repository.
EOF

git add .
git commit -m "feat: initial repository structure"
git push origin main
```

---

## Step 2-2: ArgoCDのインストール

### トラブル: ディレクトリ移動時にhelmが見つからない

**現象：** `platform-infra` から `platform-gitops` に移動すると `helm` が見つからなくなる。

```
direnv: unloading
Command 'helm' not found
```

**原因：**
- miseはリポジトリの `.mise.toml` を見てツールを有効化する設計
- `platform-gitops` には `.mise.toml` がないため、helmが有効にならない
- `.envrc` の `mise hook-env` も `.mise.toml` がない場合は空を返す

**解決策：** `platform-infra` の `.mise.toml` と `.envrc` を `platform-gitops` にコピーし、miseに信頼させる。

```bash
cp ~/platform-infra/.mise.toml ~/platform-gitops/.mise.toml
cp ~/platform-infra/.envrc ~/platform-gitops/.envrc
direnv allow ~/platform-gitops
mise trust ~/platform-gitops
```

### ArgoCDのインストール

```bash
# valuesファイルの作成
mkdir -p platform/argocd
cat > platform/argocd/values.yaml << 'EOF'
global:
  domain: argocd.localhost

configs:
  params:
    # HTTP通信を許可（Ingress経由のアクセスのため）
    server.insecure: true

server:
  ingress:
    enabled: true
    ingressClassName: nginx
    hostname: argocd.localhost
EOF
```

```bash
# Helmリポジトリの追加
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# ArgoCDのインストール
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values ~/platform-gitops/platform/argocd/values.yaml \
  --wait
```

### コミット

```bash
git add .mise.toml .envrc platform/argocd/values.yaml
git commit -m "feat: add ArgoCD and mise/direnv config"
git push origin main
```

---

## Step 2-3: App of Appsパターンの構築

### ArgoCDへのSSH鍵登録（Privateリポジトリのため必要）

```bash
# ArgoCDにログイン
argocd login localhost:8080 \
  --username admin \
  --password $(argocd admin initial-password -n argocd | head -1) \
  --insecure

# SSH鍵を登録
argocd repo add git@github.com:okccl/platform-gitops.git \
  --ssh-private-key-path ~/.ssh/id_ed25519

# 登録確認
argocd repo list
```

### Root Appの作成

```bash
cat > ~/platform-gitops/bootstrap/root.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:okccl/platform-gitops.git
    targetRevision: HEAD
    path: platform
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

```bash
# Root AppをArgoCDに登録
kubectl apply -f ~/platform-gitops/bootstrap/root.yaml

# 同期確認
argocd app get root
```

**期待する出力：**
```
Sync Status:   Synced to HEAD
Health Status: Healthy
```

### コミット

```bash
git add bootstrap/root.yaml
git commit -m "feat: add ArgoCD root app (App of Apps)"
git push origin main
```

---

## Step 2-4: External Secrets Operatorの導入

### ESOのインストール

```bash
mkdir -p ~/platform-gitops/platform/secrets
cat > ~/platform-gitops/platform/secrets/values.yaml << 'EOF'
installCRDs: true
EOF
```

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --values ~/platform-gitops/platform/secrets/values.yaml \
  --wait
```

### トラブル: ClusterSecretStore適用時にAPIバージョンエラー

**現象：**

```
error: resource mapping not found for name: "fake-store" namespace: ""
no matches for kind "ClusterSecretStore" in version "external-secrets.io/v1beta1"
```

**原因：** ESOの新バージョンでAPIバージョンが `v1beta1` から `v1` に変更された。

**確認コマンド：**

```bash
kubectl api-resources | grep clustersecretstore
# → external-secrets.io/v1
```

**解決策：** `apiVersion` を `external-secrets.io/v1` に修正。

### ClusterSecretStoreの作成

```bash
cat > ~/platform-gitops/platform/secrets/cluster-secret-store.yaml << 'EOF'
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: fake-store
spec:
  provider:
    fake:
      data:
        - key: /demo/password
          value: supersecret
          version: v1
EOF
```

```bash
kubectl apply -f ~/platform-gitops/platform/secrets/cluster-secret-store.yaml
kubectl get clustersecretstore
```

**期待する出力：**
```
NAME         AGE   STATUS   CAPABILITIES   READY
fake-store   Xs    Valid    ReadWrite      True
```

### ExternalSecretで動作確認

```bash
cat > /tmp/demo-external-secret.yaml << 'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: demo-secret
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: fake-store
    kind: ClusterSecretStore
  target:
    name: demo-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: /demo/password
        version: v1
EOF

kubectl apply -f /tmp/demo-external-secret.yaml

# K8s Secretの自動生成を確認
kubectl get externalsecret -n default
kubectl get secret demo-secret -n default -o jsonpath='{.data.password}' | base64 -d
# → supersecret

# 動作確認後に削除
kubectl delete -f /tmp/demo-external-secret.yaml
```

### コミット

```bash
cd ~/platform-gitops
git add platform/secrets/
git commit -m "feat: add External Secrets Operator with fake provider"
git push origin main
```

---

## Step 2-5: README更新とコミット

```bash
cd ~/platform-gitops
git add README.md
git commit -m "docs: add Phase 2 section to README"
git push origin main
```

---
### ArgoCDの初期パスワード変更

```bash
argocd account update-password \
  --current-password $(argocd admin initial-password -n argocd | head -1) \
  --new-password <新しいパスワード>
```

```bash
# 変更後、初期パスワードのSecretを削除（任意）
kubectl delete secret argocd-initial-admin-secret -n argocd
```

---
## 発見した知見

### 1. miseのリポジトリ単位での有効化

miseはリポジトリの `.mise.toml` を見てツールを有効化する設計のため、
新しいリポジトリを作成するたびに以下の対応が必要：

1. `.mise.toml` のコピー（または新規作成）
2. `.envrc` のコピー
3. `direnv allow` で有効化
4. `mise trust` で信頼済みに設定

### 2. External Secrets OperatorのAPIバージョン

ESO v0.10以降、APIバージョンが `v1beta1` から `v1` に変更された。
マニフェスト作成時は `kubectl api-resources | grep externalsecret` で
現在のAPIバージョンを確認すること。

### 3. PrivateリポジトリとArgoCDのSSH鍵登録

ArgoCDがPrivateリポジトリにアクセスするにはSSH鍵の登録が必要。
クラスタを再作成した場合は再登録が必要になる点に注意。

```bash
argocd repo add git@github.com:<org>/<repo>.git \
  --ssh-private-key-path ~/.ssh/id_ed25519
```

---

## Phase 2完了時点のリポジトリ構成

```
platform-gitops/
├── bootstrap/
│   └── root.yaml                      # ArgoCD Root App (App of Apps)
├── platform/
│   ├── argocd/
│   │   └── values.yaml                # ArgoCDのHelm values
│   ├── secrets/
│   │   ├── values.yaml                # ESOのHelm values
│   │   └── cluster-secret-store.yaml  # Fakeプロバイダーの設定
│   ├── ingress/
│   ├── monitoring/
│   ├── logging/
│   └── policy/
├── apps/
├── .mise.toml
├── .envrc
└── README.md
```
