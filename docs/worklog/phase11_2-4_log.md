# Phase 11 作業ログ（11-1〜11-4）


## Phase 11-2：Trivy Operator 導入

### Namespace 追加
```bash
# platform-namespaces.yaml に trivy-system を追記
vi ~/platform-gitops/platform/policy/platform-namespaces.yaml
```

### ArgoCD App 作成
```bash
mkdir -p ~/platform-gitops/platform/security

cat > ~/platform-gitops/platform/security/trivy-operator-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trivy-operator
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  project: default
  source:
    repoURL: https://aquasecurity.github.io/helm-charts/
    chart: trivy-operator
    targetRevision: 0.32.1
    helm:
      values: |
        trivy:
          ignoreUnfixed: true
        operator:
          excludeNamespaces: "kube-system,trivy-system"
        serviceMonitor:
          enabled: true
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
  destination:
    server: https://kubernetes.default.svc
    namespace: trivy-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
EOF
```

### Git push
```bash
cd ~/platform-gitops
git add platform/policy/platform-namespaces.yaml platform/security/
git commit -m "feat(security): add Trivy Operator v0.32.1"
git push origin main
```

### 動作確認
```bash
kubectl get application -n argocd trivy-operator
kubectl get pods -n trivy-system
kubectl get vulnerabilityreports --all-namespaces | head -30
kubectl get vulnerabilityreports --all-namespaces -o json \
  | jq -r '.items[] | [.metadata.namespace, .metadata.name, .report.summary.criticalCount, .report.summary.highCount] | @tsv' \
  | sort | head -20
kubectl get configauditreports --all-namespaces | head -20
kubectl get vulnerabilityreports -n sample-app
```

---

## Phase 11-3：Argo Rollouts 導入

### Namespace 追加
```bash
cat >> ~/platform-gitops/platform/policy/platform-namespaces.yaml << 'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: argo-rollouts
  labels:
    platform-managed: "true"
EOF
```

### ArgoCD App 作成
```bash
cat > ~/platform-gitops/platform/security/argo-rollouts-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-rollouts
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  project: default
  source:
    repoURL: https://argoproj.github.io/argo-helm
    chart: argo-rollouts
    targetRevision: 2.40.9
    helm:
      values: |
        dashboard:
          enabled: true
        serviceMonitor:
          enabled: true
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
  destination:
    server: https://kubernetes.default.svc
    namespace: argo-rollouts
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
EOF

cd ~/platform-gitops
git add platform/policy/platform-namespaces.yaml platform/security/argo-rollouts-app.yaml
git commit -m "feat(rollouts): add Argo Rollouts v2.40.9"
git push origin main
```

### common-app Library Chart 更新（v0.1.0 → v0.2.0）
```bash
# _helpers.tpl に Rollout テンプレートを追加
cat >> ~/platform-charts/charts/common-app/templates/_helpers.tpl << 'EOF'
（Rollout テンプレート本文 省略）
EOF

# values.yaml に rollout デフォルト値を追加
cat >> ~/platform-charts/charts/common-app/values.yaml << 'EOF'
rollout:
  enabled: false
  canary:
    steps:
      - setWeight: 20
      - pause: {}
      - setWeight: 100
EOF

# バージョンアップ
sed -i 's/^version: 0.1.0/version: 0.2.0/' ~/platform-charts/charts/common-app/Chart.yaml

# 再パッケージ
cd ~/platform-charts/charts/common-app
helm package .
mv common-app-0.2.0.tgz ~/platform-charts/charts/sample-backend/charts/
rm ~/platform-charts/charts/sample-backend/charts/common-app-0.1.0.tgz

# sample-backend Chart.yaml の依存バージョン更新
sed -i '/common-app/{n;s/version: "0.1.0"/version: "0.2.0"/}' \
  ~/platform-charts/charts/sample-backend/Chart.yaml

# dependency update
cd ~/platform-charts/charts/sample-backend
helm dependency update .

cd ~/platform-charts
git add .
git commit -m "feat(common-app): add Rollout template v0.2.0"
git push origin main
```

### sample-backend/templates/all.yaml を Rollout 対応に更新
```bash
cat > ~/platform-charts/charts/sample-backend/templates/all.yaml << 'EOF'
{{- if .Values.rollout.enabled }}
{{ include "common-app.rollout" . }}
{{- else }}
{{ include "common-app.deployment" . }}
{{- end }}
---
{{ include "common-app.service" . }}
---
{{ include "common-app.ingress" . }}
---
{{ include "common-app.hpa" . }}
---
{{ include "common-app.serviceMonitor" . }}
---
{{ include "common-db.cluster" . }}
---
{{ include "common-db.scheduledBackup" . }}
EOF
```

### platform-gitops：sample-backend values 更新
```bash
cat >> ~/platform-gitops/apps/sample-backend/values.yaml << 'EOF'

rollout:
  enabled: true
  canary:
    steps:
      - setWeight: 20
      - pause: {}
      - setWeight: 100
EOF
```

### Rollouts Dashboard HTTPRoute 追加
```bash
cat > ~/platform-gitops/platform/gateway/config/httproute-rollouts.yaml << 'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rollouts-dashboard
  namespace: envoy-gateway-system
spec:
  parentRefs:
    - name: eg
      namespace: envoy-gateway-system
  hostnames:
    - "rollouts.localhost"
  rules:
    - backendRefs:
        - kind: Service
          name: argo-rollouts-dashboard
          namespace: argo-rollouts
          port: 3100
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: reference-grant-argo-rollouts
  namespace: argo-rollouts
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: envoy-gateway-system
  to:
    - group: ""
      kind: Service
EOF

cd ~/platform-gitops
git add apps/sample-backend/values.yaml \
        platform/gateway/config/httproute-rollouts.yaml
git commit -m "feat(rollouts): enable Rollout for sample-backend, add dashboard HTTPRoute"
git push origin main
```

### mise に kubectl-argo-rollouts を追加
```bash
cd ~/platform-infra
mise use "ubi:argoproj/argo-rollouts[exe=kubectl-argo-rollouts]@1.8.3"

# platform-gitops にも追加
echo '"ubi:argoproj/argo-rollouts" = { version = "1.8.3", exe = "kubectl-argo-rollouts" }' \
  >> ~/platform-gitops/.mise.toml
cd ~/platform-gitops
mise install

# mise.toml を commit
cd ~/platform-infra
git add .mise.toml
git commit -m "chore(mise): add kubectl-argo-rollouts v1.8.3"
git push origin main

cd ~/platform-gitops
git add .mise.toml
git commit -m "chore(mise): add kubectl-argo-rollouts v1.8.3"
git push origin main
```

### カナリア動作確認
```bash
# イメージタグ変更でカナリアをトリガー
sed -i 's/tag: b5a1299/tag: canary-test/' \
  ~/platform-gitops/apps/sample-backend/values.yaml
cd ~/platform-gitops
git add apps/sample-backend/values.yaml
git commit -m "test(rollouts): trigger canary deployment for verification"
git push origin main

# Rollout の状態を watch（Paused / 20% weight を確認）
kubectl argo rollouts get rollout sample-backend -n sample-app --watch

# タグを元に戻してロールバック
sed -i 's/tag: canary-test/tag: b5a1299/' \
  ~/platform-gitops/apps/sample-backend/values.yaml
cd ~/platform-gitops
git add apps/sample-backend/values.yaml
git commit -m "revert(rollouts): restore stable image tag after canary verification"
git push origin main
```

---

## Phase 11-4：KEDA 導入

### Namespace 追加
```bash
cat >> ~/platform-gitops/platform/policy/platform-namespaces.yaml << 'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: keda
  labels:
    platform-managed: "true"
EOF
```

### ArgoCD App 作成
```bash
mkdir -p ~/platform-gitops/platform/scaling

cat > ~/platform-gitops/platform/scaling/keda-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: keda
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  project: default
  source:
    repoURL: https://kedacore.github.io/charts
    chart: keda
    targetRevision: 2.19.0
    helm:
      values: |
        resources:
          operator:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          metricServer:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
        prometheus:
          operator:
            enabled: true
          metricServer:
            enabled: true
          webhooks:
            enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: keda
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
EOF

cd ~/platform-gitops
git add platform/policy/platform-namespaces.yaml platform/scaling/
git commit -m "feat(scaling): add KEDA v2.19.0"
git push origin main
```

### ScaledObject 作成
```bash
mkdir -p ~/platform-gitops/apps/sample-backend/manifests

cat > ~/platform-gitops/apps/sample-backend/manifests/scaledobject.yaml << 'EOF'
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sample-backend
  namespace: sample-app
spec:
  scaleTargetRef:
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    name: sample-backend
  minReplicaCount: 1
  maxReplicaCount: 5
  cooldownPeriod: 30
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
        metricName: http_requests_total
        query: |
          sum(rate(http_requests_total{job="sample-backend"}[1m]))
        threshold: "10"
EOF
```

### sample-backend.yaml に manifests source を追加
```bash
cat > ~/platform-gitops/apps/sample-backend.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-backend
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "4"
spec:
  project: default
  sources:
    - repoURL: git@github.com:okccl/platform-charts.git
      targetRevision: main
      path: charts/sample-backend
      helm:
        valueFiles:
          - $values/apps/sample-backend/values.yaml
    - repoURL: git@github.com:okccl/platform-gitops.git
      targetRevision: main
      ref: values
    - repoURL: git@github.com:okccl/platform-gitops.git
      targetRevision: main
      path: apps/sample-backend/manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: sample-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
    managedNamespaceMetadata:
      labels:
        goldilocks.fairwinds.com/enabled: "true"
EOF

cd ~/platform-gitops
git add apps/sample-backend/
git commit -m "feat(scaling): add KEDA ScaledObject for sample-backend"
git push origin main
```

### common-app Service ポート名修正（v0.2.0 → v0.3.0）

ServiceMonitor が Prometheus scrape ターゲットを解決できない問題を修正。  
Service の port に `name: http` が欠落していたため追加。

```bash
sed -i 's/    - protocol: TCP/    - name: http\n      protocol: TCP/' \
  ~/platform-charts/charts/common-app/templates/_helpers.tpl

sed -i 's/^version: 0.2.0/version: 0.3.0/' \
  ~/platform-charts/charts/common-app/Chart.yaml

cd ~/platform-charts/charts/common-app
helm package .
mv common-app-0.3.0.tgz ~/platform-charts/charts/sample-backend/charts/
rm ~/platform-charts/charts/sample-backend/charts/common-app-0.2.0.tgz

sed -i '/common-app/{n;s/version: "0.2.0"/version: "0.3.0"/}' \
  ~/platform-charts/charts/sample-backend/Chart.yaml

cd ~/platform-charts/charts/sample-backend
helm dependency update .

cd ~/platform-charts
git add .
git commit -m "fix(common-app): add port name 'http' to Service template v0.3.0"
git push origin main
```

### mise に oha を追加
```bash
cd ~/platform-infra
mise use "github:hatoo/oha@1.14.0"
sed -i '/"ubi:hatoo\/oha"/d' ~/platform-infra/.mise.toml
git add .mise.toml
git commit -m "chore(mise): add oha v1.14.0"
git push origin main

echo '"github:hatoo/oha" = "1.14.0"' >> ~/platform-gitops/.mise.toml
cd ~/platform-gitops
mise install
git add .mise.toml
git commit -m "chore(mise): add oha v1.14.0"
git push origin main
```

### 負荷テスト（KEDA スケールアウト確認）
```bash
# ターミナル1：Rollout を watch
kubectl argo rollouts get rollout sample-backend -n sample-app --watch

# ターミナル2：負荷をかける（50並列・120秒）
oha -z 120s -c 50 --host sample-backend.localhost http://172.19.0.2/health

# HPA の状態確認
kubectl get hpa -n sample-app keda-hpa-sample-backend
```

**結果**：1 → 5 レプリカへのスケールアウト、および負荷終了後のスケールイン（5 → 1）を確認。
