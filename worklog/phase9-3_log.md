# Phase 9 作業ログ

## セッション情報
- スコープ: Phase 9（タスク4〜6）

---

## タスク4: kubectl drain によるノード停止試験

### drain前の状態確認
```bash
kubectl get pods -n sample-app -o wide
kubectl get cluster -n sample-app sample-backend-db -o jsonpath='{.status.currentPrimary}'
```

**drain前のPod配置**

| Pod | Node | 役割 |
|---|---|---|
| sample-backend-5867fd9d5c-2j22b | k3d-dev-server-0 | アプリ |
| sample-backend-5867fd9d5c-grgl4 | k3d-dev-agent-1 | アプリ |
| sample-backend-db-1 | k3d-dev-agent-0 | DB Primary |
| sample-backend-db-2 | k3d-dev-server-0 | DB Replica |
| sample-frontend-64cb8fd8d4-kbt2m | k3d-dev-agent-0 | アプリ |
| sample-frontend-64cb8fd8d4-t77fg | k3d-dev-agent-2 | アプリ |

### リクエスト監視起動（ターミナル1）
```bash
watch -n 1 'curl -sk https://sample-backend.localhost/items | head -c 100'
```

### drain実行（ターミナル2）
```bash
kubectl drain k3d-dev-agent-0 --ignore-daemonsets --delete-emptydir-data
```

### drain後の状態確認
```bash
kubectl get pods -n sample-app -o wide
kubectl get cluster -n sample-app sample-backend-db -o jsonpath='{.status.currentPrimary}'
kubectl get events -n sample-app --sort-by='.lastTimestamp' | grep -i "db"
```

**drain後のPod配置**

| Pod | Node | 役割 |
|---|---|---|
| sample-backend-5867fd9d5c-2j22b | k3d-dev-server-0 | アプリ |
| sample-backend-5867fd9d5c-grgl4 | k3d-dev-agent-1 | アプリ |
| sample-backend-db-1 | Pending（cordon中） | - |
| sample-backend-db-2 | k3d-dev-server-0 | DB Primary（昇格） |
| sample-frontend-64cb8fd8d4-t77fg | k3d-dev-agent-2 | アプリ |
| sample-frontend-64cb8fd8d4-tpc56 | k3d-dev-server-0 | アプリ（再スケジュール） |

**Switchoverイベント確認**
- `SwitchingOver`: agent-0がunschedulableになったことを検知、db-1→db-2へ自動Switchover
- drain中はDB接続エラーが一時発生（Switchover完了まで数十秒の瞬断）

### uncordon（Node復旧）
```bash
kubectl uncordon k3d-dev-agent-0
```

### トラブルシュート: db-1がPending/Unhealthyのまま回復しない

**原因1: minio-backup-secret が存在しない**
```bash
# エラーログ確認
kubectl logs sample-backend-db-1 -n sample-app --tail=30
# → "while getting secret minio-backup-secret: secrets not found"

# 対処: Secretを再作成
kubectl create secret generic minio-backup-secret \
  -n sample-app \
  --from-literal=ACCESS_KEY_ID=minioadmin \
  --from-literal=ACCESS_SECRET_KEY=minioadmin123
```

**原因2: minio-auth が存在せずMinIO Pod が起動できない**
```bash
# MinIO Pod状態確認
kubectl get pods -n minio
kubectl describe pod -n minio minio-89bd995d4-fnw7x | tail -20
# → "MountVolume.SetUp failed for volume minio-user: secret minio-auth not found"

# 対処: Secretを再作成
kubectl create secret generic minio-auth \
  --from-literal=rootUser=minioadmin \
  --from-literal=rootPassword=minioadmin123 \
  -n minio
```

**db-1の手動再起動**
```bash
kubectl delete pod sample-backend-db-1 -n sample-app
kubectl get pods -n sample-app -o wide
```

---

## タスク5: VPA（Vertical Pod Autoscaler）の導入

### ディレクトリ作成
```bash
mkdir -p ~/platform-gitops/platform/vpa
mkdir -p ~/platform-gitops/platform/goldilocks
```

### vpa/application.yaml 作成
```bash
cat > ~/platform-gitops/platform/vpa/application.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vpa
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://charts.fairwinds.com/stable
      chart: vpa
      targetRevision: 4.11.0
      helm:
        valueFiles:
          - $values/platform/vpa/values.yaml
    - repoURL: git@github.com:okccl/platform-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: vpa
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
EOF
```

### vpa/values.yaml 作成
```bash
cat > ~/platform-gitops/platform/vpa/values.yaml << 'EOF'
recommender:
  enabled: true

updater:
  enabled: false

admissionController:
  enabled: false
EOF
```

### goldilocks/application.yaml 作成
```bash
cat > ~/platform-gitops/platform/goldilocks/application.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: goldilocks
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://charts.fairwinds.com/stable
      chart: goldilocks
      targetRevision: 10.3.0
      helm:
        valueFiles:
          - $values/platform/goldilocks/values.yaml
    - repoURL: git@github.com:okccl/platform-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: goldilocks
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
EOF
```

### goldilocks/values.yaml 作成
```bash
cat > ~/platform-gitops/platform/goldilocks/values.yaml << 'EOF'
vpa:
  enabled: false

dashboard:
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: selfsigned-issuer
    hosts:
      - host: goldilocks.localhost
        paths:
          - path: /
            type: Prefix
    tls:
      - secretName: goldilocks-tls
        hosts:
          - goldilocks.localhost
EOF
```

### トラブルシュート: Kyvernoポリシーに弾かれる

**原因**: `vpa` / `goldilocks` namespaceに `platform-managed: "true"` ラベルが未設定

**対処**: platform-namespaces.yaml に追記してpush後、Kyvernoを再起動
```bash
# platform-namespaces.yaml に vpa / goldilocks を追記（全文上書き）
cat > ~/platform-gitops/platform/policy/platform-namespaces.yaml << 'EOF'
# （略: vpa と goldilocks の Namespace を追記）
EOF

cd ~/platform-gitops
git add platform/policy/platform-namespaces.yaml
git commit -m "feat: add vpa and goldilocks namespaces to platform-managed"
git push origin main

# Kyvernoキャッシュリフレッシュ
kubectl rollout restart deployment -n kyverno
kubectl rollout status deployment -n kyverno
```

### push & 確認
```bash
cd ~/platform-gitops
git add platform/vpa/ platform/goldilocks/
git commit -m "feat: add VPA and Goldilocks for resource recommendation"
git push origin main

kubectl get applications -n argocd | grep -E "vpa|goldilocks"
kubectl get pods -n vpa
kubectl get pods -n goldilocks
```

---

## タスク6: Goldilocksダッシュボードでのリソース使用量可視化

### sample-app namespace にラベル付与
```bash
kubectl label namespace sample-app goldilocks.fairwinds.com/enabled=true
```

### VPAオブジェクト自動生成確認
```bash
kubectl get vpa -n sample-app
```

### ダッシュボード確認
```
https://goldilocks.localhost
```

**結果**: sample-backend / sample-frontend の推奨値が表示されることを確認
- sample-backend: CPU過剰（100m→23m推奨）、Memory適正（128Mi→100Mi推奨）
- sample-frontend: CPU過剰（50m→15m推奨）、Memory不足（64Mi→100Mi推奨）
