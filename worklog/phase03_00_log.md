# Phase 3-1 作業ログ

## 完了したStep
- Step 3-1: ingress-nginx インストール
- Step 3-2: cert-manager インストール
- Step 3-3: ClusterIssuer 作成

---

## 事前対応: root.yaml の修正

`platform/` 配下のサブディレクトリを再帰処理するため `recurse: true` を追加。
この変更はGit pushだけでは反映されないため、`kubectl apply` で直接適用する。

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
    directory:
      recurse: true
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
cd ~/platform-gitops
git add bootstrap/root.yaml
git commit -m "fix: enable recursive directory sync in root app"
git push origin main
kubectl apply -f ~/platform-gitops/bootstrap/root.yaml
```

---

## Step 3-1: ingress-nginx インストール

### ファイル作成

```bash
cat > ~/platform-gitops/platform/ingress/application.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: 4.15.1
    helm:
      valuesObject:
        controller:
          # k3dのServiceLBがNodePortに対応しているためLoadBalancerで動作する
          service:
            type: LoadBalancer
          # k3dのLoadBalancerがポート80/443をリッスンするため必要
          hostPort:
            enabled: false
          # Prometheusメトリクスを有効化（Phase 4で使用）
          metrics:
            enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### Git push & 反映

```bash
cd ~/platform-gitops
git add platform/ingress/application.yaml
git commit -m "feat: add ingress-nginx application"
git push origin main
kubectl apply -f ~/platform-gitops/bootstrap/root.yaml
argocd app sync root --force
```

### 確認

```bash
kubectl get pods -n ingress-nginx
kubectl get application ingress-nginx -n argocd
```

---

## Step 3-2: cert-manager インストール

### ファイル作成

```bash
cat > ~/platform-gitops/platform/ingress/cert-manager-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://charts.jetstack.io
    chart: cert-manager
    targetRevision: v1.20.2
    helm:
      valuesObject:
        crds:
          enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      # CRDを含むリソースの更新に必要
      - ServerSideApply=true
EOF
```

### Git push & 反映

```bash
cd ~/platform-gitops
git add platform/ingress/cert-manager-app.yaml
git commit -m "feat: add cert-manager application"
git push origin main
kubectl apply -f ~/platform-gitops/bootstrap/root.yaml
argocd app sync root --force
```

### 確認

```bash
kubectl get pods -n cert-manager
kubectl get application cert-manager -n argocd
```

---

## Step 3-3: ClusterIssuer 作成

### ファイル作成

```bash
cat > ~/platform-gitops/platform/ingress/cluster-issuer.yaml << 'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: local-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: local-ca
  secretName: local-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: local-ca-issuer
spec:
  ca:
    secretName: local-ca-secret
EOF
```

### Git push & 反映

```bash
cd ~/platform-gitops
git add platform/ingress/cluster-issuer.yaml
git commit -m "feat: add selfsigned ClusterIssuer for local development"
git push origin main
argocd app sync root --force
```

### 確認

```bash
kubectl get clusterissuer
```

期待する出力:
```
NAME                READY   AGE
local-ca-issuer     True    Xs
selfsigned-issuer   True    Xs
```

---

## 最終状態確認

```bash
kubectl get application -n argocd
kubectl get pods -n ingress-nginx
kubectl get pods -n cert-manager
kubectl get clusterissuer
```

期待する出力:
```
# application
NAME            SYNC STATUS   HEALTH STATUS
cert-manager    Synced        Healthy
ingress-nginx   Synced        Progressing  ← EXTERNAL-IP pendingのため。実害なし
root            Synced        Healthy

# ingress-nginx pods
NAME                                       READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-xxxxxxxxx-xxxxx   1/1     Running   0          Xm

# cert-manager pods
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-xxxxxxxxx-xxxxx              1/1     Running   0          Xm
cert-manager-cainjector-xxxxxxxxx-xxxxx   1/1     Running   0          Xm
cert-manager-webhook-xxxxxxxxx-xxxxx      1/1     Running   0          Xm

# clusterissuer
NAME                READY   AGE
local-ca-issuer     True    Xm
selfsigned-issuer   True    Xm
```
