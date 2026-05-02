# Phase 7 作業ログ

## 環境情報

| 項目                  | 値            |
| ------------------- | ------------ |
| Kubernetes Server   | v1.31.5+k3s1 |
| Kyverno Helmチャート    | 3.7.1        |
| Kyverno App version | v1.17.1      |

---

## 実施手順

### 1. ディレクトリ・ファイル構成の作成

```bash
mkdir -p ~/platform-gitops/platform/policy/policies
```

### 2. ArgoCD Application マニフェストの作成

#### Kyverno本体（最終版）

```bash
cat > ~/platform-gitops/platform/policy/kyverno.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kyverno
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://kyverno.github.io/kyverno
    chart: kyverno
    targetRevision: 3.7.1
    helm:
      values: |
        admissionController:
          replicas: 1
        backgroundController:
          replicas: 1
        cleanupController:
          replicas: 1
        reportsController:
          replicas: 1
  destination:
    server: https://kubernetes.default.svc
    namespace: kyverno
  ignoreDifferences:
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
      name: kyverno:migrate-resources
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding
      name: kyverno:migrate-resources
    - group: apiextensions.k8s.io
      kind: CustomResourceDefinition
      jsonPointers:
        - /metadata/labels
        - /metadata/annotations
    - group: batch
      kind: Job
      name: kyverno-migrate-resources
      jsonPointers:
        - /status
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
      - RespectIgnoreDifferences=true
EOF
```

#### ポリシー集

```bash
cat > ~/platform-gitops/platform/policy/kyverno-policies.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kyverno-policies
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/okccl/platform-gitops
    targetRevision: main
    path: platform/policy/policies
  destination:
    server: https://kubernetes.default.svc
    namespace: kyverno
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### 3. ポリシーマニフェストの作成

#### require-resource-limits.yaml

```bash
cat > ~/platform-gitops/platform/policy/policies/require-resource-limits.yaml << 'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
  annotations:
    policies.kyverno.io/title: リソース制限の必須化
    policies.kyverno.io/description: >-
      すべてのコンテナにCPU・Memoryのlimitsが設定されていることを強制します。
      設定がない場合、ノードのリソースを使い果たすリスクがあります。
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: require-limits
      match:
        any:
          - resources:
              kinds:
                - Pod
      exclude:
        any:
          - resources:
              namespaces:
                - kyverno
                - kube-system
                - argocd
                - monitoring
                - logging
                - tracing
                - ingress-nginx
                - cert-manager
                - external-secrets
      validate:
        message: "すべてのコンテナにresources.limitsのcpuとmemoryを設定してください。"
        pattern:
          spec:
            containers:
              - name: "*"
                resources:
                  limits:
                    cpu: "?*"
                    memory: "?*"
EOF
```

#### disallow-latest-tag.yaml

```bash
cat > ~/platform-gitops/platform/policy/policies/disallow-latest-tag.yaml << 'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
  annotations:
    policies.kyverno.io/title: latestタグの禁止
    policies.kyverno.io/description: >-
      イメージタグにlatestを使用すること、またはタグを省略することを禁止します。
      明示的なバージョンタグを使用することでデプロイの再現性を確保します。
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: disallow-latest-tag
      match:
        any:
          - resources:
              kinds:
                - Pod
      exclude:
        any:
          - resources:
              namespaces:
                - kyverno
                - kube-system
                - argocd
                - monitoring
                - logging
                - tracing
                - ingress-nginx
                - cert-manager
                - external-secrets
      validate:
        message: "latestタグ・タグ未指定のイメージは使用できません。明示的なバージョンタグを指定してください。"
        deny:
          conditions:
            any:
              - key: "{{ images.containers.*.tag }}"
                operator: AnyIn
                value:
                  - "latest"
                  - ""
EOF
```

#### add-default-labels.yaml

```bash
cat > ~/platform-gitops/platform/policy/policies/add-default-labels.yaml << 'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
  annotations:
    policies.kyverno.io/title: デフォルトラベルの自動付与
    policies.kyverno.io/description: >-
      すべてのPodに app.kubernetes.io/managed-by: platform ラベルを自動付与します。
      プラットフォームが管理するリソースの識別・フィルタリングに使用します。
spec:
  rules:
    - name: add-managed-by-label
      match:
        any:
          - resources:
              kinds:
                - Pod
      exclude:
        any:
          - resources:
              namespaces:
                - kyverno
                - kube-system
                - argocd
                - monitoring
                - logging
                - tracing
                - ingress-nginx
                - cert-manager
                - external-secrets
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              app.kubernetes.io/managed-by: platform
EOF
```

### 4. Git push

```bash
cd ~/platform-gitops
git add platform/policy/
git commit -m "feat: Phase7 Kyvernoポリシーエンジンの追加"
git push origin main
```

### 5. ArgoCDへの登録

```bash
# ArgoCDログイン
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)
argocd login argocd.localhost \
  --username admin \
  --password $ARGOCD_PASSWORD \
  --insecure --grpc-web

# Kyverno本体の登録
argocd app create -f ~/platform-gitops/platform/policy/kyverno.yaml --grpc-web

# ポリシー集の登録
argocd app create -f ~/platform-gitops/platform/policy/kyverno-policies.yaml --grpc-web
```

### 6. デプロイ確認

```bash
kubectl get pods -n kyverno
kubectl get validatingwebhookconfiguration | grep kyverno
kubectl get clusterpolicy
```

### 7. 動作検証

#### Validateポリシー検証（拒否確認）

```bash
# limits未設定 → 拒否されることを確認
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-no-limits
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:1.25
EOF

# latestタグ → 拒否されることを確認
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-latest-tag
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:latest
      resources:
        limits:
          cpu: "100m"
          memory: "128Mi"
EOF
```

#### Mutateポリシー検証（ラベル自動付与確認）

```bash
# 正常なPodを作成
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-mutate
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      resources:
        limits:
          cpu: "100m"
          memory: "128Mi"
EOF

# ラベルの確認
kubectl get pod test-mutate -o jsonpath='{.metadata.labels}' | python3 -m json.tool

# テストPodの削除
kubectl delete pod test-mutate
```

---

## トラブルシューティング記録

### 問題1: kyverno:migrate-resources の競合

**症状:** `clusterroles.rbac.authorization.k8s.io "kyverno:migrate-resources" already exists`

**原因:** KyvernoのHelmチャートが持つpost-upgradeフックのRBACリソースが、ArgoCDのCreate処理と競合。

**対処:** `ignoreDifferences` にClusterRole/ClusterRoleBindingを追加し、`ServerSideApply=true` + `RespectIgnoreDifferences=true` を設定。

### 問題2: CRDのOutOfSync（annotations: {}）

**症状:** 複数のCRDで `annotations: {}` の差分が発生し続ける。

**原因:** HelmチャートがCRDに空のannotationsを期待しているが、クラスタ上のCRDにはフィールド自体が存在しない。

**対処:** `ignoreDifferences` に `/metadata/annotations` と `/metadata/labels` のjsonPointerを追加。Applicationを再作成して反映。

### 問題3: disallow-latest-tagポリシーのオペレーターエラー

**症状:** `operator NotContains is invalid`

**原因:** Kyverno 3.7.1では `NotContains` オペレーターが無効。

**対処:** `images.containers.*.tag` のJMESPathと `AnyIn` オペレーターを使用する方式に変更。
