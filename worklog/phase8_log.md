# Phase 8 作業ログ

## 環境情報

| 項目 | 値 |
|---|---|
| 作業日 | 2026-04-19 |
| OS (WSL) | Ubuntu 24.04 LTS |
| 作業ユーザー | ccl |

---

## 事前作業: sample-service → sample-backend リネーム

### ArgoCDからsample-helloを削除
```bash
argocd app delete sample-hello --cascade
cd ~/platform-gitops
git rm apps/sample-hello.yaml
git rm -r apps/sample-hello/
git commit -m "chore: remove sample-hello application"
git push origin main
```

### ArgoCDからsample-serviceを削除（リソースは残す）
```bash
argocd app delete sample-service --cascade=false
```

### platform-gitopsのファイルをリネーム
```bash
cd ~/platform-gitops
git mv apps/sample-service.yaml apps/sample-backend.yaml
git mv apps/sample-service apps/sample-backend
# apps/sample-backend.yaml を sample-backend 向けに書き換え
# apps/sample-backend/values.yaml を sample-backend 向けに書き換え
git add -A
git commit -m "chore: rename sample-service to sample-backend"
git push origin main
```

### platform-chartsのディレクトリをリネーム
```bash
cd ~/platform-charts
git mv charts/sample-service charts/sample-backend
# charts/sample-backend/Chart.yaml の name を sample-backend に変更
# charts/sample-backend/values.yaml を sample-backend 向けに書き換え
git add -A
git commit -m "chore: rename sample-service to sample-backend in platform-charts"
git push origin main
```

### namespace-labels.yamlを更新
```bash
# platform/secrets/external-secrets/namespace-labels.yaml の name を sample-backend に変更
cd ~/platform-gitops
git add -A
git commit -m "chore: rename sample-service namespace to sample-backend"
git push origin main
```

### 古いnamespaceを削除
```bash
kubectl delete namespace sample-service
```

### apps-rootを手動sync
```bash
argocd app sync apps-root
```

### Kyvernoのlatest-tagブロック対応（一時的なplaceholderタグ設定）
```bash
# apps/sample-backend/values.yaml の tag を placeholder に変更
git add -A
git commit -m "fix: use placeholder tag to pass kyverno policy"
git push origin main
```

---

## Step 1: CNPG Operator 導入

### ArgoCD Applicationの作成
```bash
mkdir -p ~/platform-gitops/platform/cnpg
cat > ~/platform-gitops/platform/cnpg/application.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cnpg
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://cloudnative-pg.github.io/charts
    chart: cloudnative-pg
    targetRevision: 0.28.0
    helm:
      valuesObject:
        monitoring:
          podMonitorEnabled: true
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 256Mi
  destination:
    server: https://kubernetes.default.svc
    namespace: cnpg-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
EOF

cd ~/platform-gitops
git add platform/cnpg/application.yaml
git commit -m "feat: add cloudnative-pg operator (v0.28.0)"
git push origin main
```

### 導入確認
```bash
kubectl get applications -n argocd | grep cnpg
kubectl get pods -n cnpg-system
```

---

## Step 2: MinIO 導入

### namespace作成とSecret作成
```bash
kubectl create namespace minio
kubectl create secret generic minio-auth \
  --from-literal=rootUser=minioadmin \
  --from-literal=rootPassword=minioadmin123 \
  -n minio
```

### ArgoCD Applicationの作成
```bash
mkdir -p ~/platform-gitops/platform/minio
cat > ~/platform-gitops/platform/minio/application.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: minio
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.min.io
    chart: minio
    targetRevision: 5.4.0
    helm:
      valuesObject:
        existingSecret: minio-auth
        mode: standalone
        persistence:
          enabled: true
          size: 10Gi
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        makeBucketJob:
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: 100m
              memory: 128Mi
        buckets:
          - name: cnpg-backup
            policy: none
            purge: false
        ingress:
          enabled: true
          ingressClassName: nginx
          hosts:
            - minio.localhost
        consoleIngress:
          enabled: true
          ingressClassName: nginx
          hosts:
            - minio-console.localhost
        metrics:
          serviceMonitor:
            enabled: true
            includeNode: true
  destination:
    server: https://kubernetes.default.svc
    namespace: minio
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

cd ~/platform-gitops
git add -A
git commit -m "feat: add minio for cnpg backup storage"
git push origin main
```

### 導入確認
```bash
kubectl get pods -n minio
kubectl exec -n minio deploy/minio -- mc alias set local http://localhost:9000 minioadmin minioadmin123
kubectl exec -n minio deploy/minio -- mc ls local
```

---

## 副産物: Kyvernoポリシーの改善（namespace selectorベースへ）

### 背景
MinIO・CNPGなどプラットフォーム側のJobにもresource limits制約が適用されてしまい、
Jobが作成できない問題が発生。根本的な解決策として、namespace selectorを使った除外方式に変更。

### ポリシーの変更（3ファイル共通）
`require-resource-limits.yaml` / `disallow-latest-tag.yaml` / `add-default-labels.yaml` の
`exclude`セクションを以下のように変更。

```yaml
# 変更前: namespace名をリストアップ
exclude:
  any:
    - resources:
        namespaces:
          - kyverno
          - kube-system
          # ... 列挙

# 変更後: namespace selectorで除外
exclude:
  any:
    - resources:
        namespaceSelector:
          matchLabels:
            platform-managed: "true"
```

### プラットフォームnamespaceのマニフェスト作成
```bash
cat > ~/platform-gitops/platform/policy/platform-namespaces.yaml << 'EOF'
# platform-managed: "true" ラベルを持つnamespaceはKyvernoポリシー適用外
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    platform-managed: "true"
---
# ... （全プラットフォームnamespace）
EOF

git add -A
git commit -m "refactor: use namespace selector for kyverno policy exclusion"
git push origin main
```

---

## Step 3: common-db Library Chart 作成

```bash
mkdir -p ~/platform-charts/charts/common-db/templates

# Chart.yaml, values.yaml, templates/_helpers.tpl を作成
# _helpers.tpl に common-db.cluster / common-db.scheduledBackup テンプレートを定義

cd ~/platform-charts
git add -A
git commit -m "feat: add common-db library chart for CNPG PostgreSQL"
git push origin main
```

---

## Step 4: sample-backendをFastAPIアプリに作り替え

### アプリコードの作成
```bash
cd ~/sample-service
rm -rf src/
mkdir src
# src/main.py を FastAPI + asyncpg で作成
# src/requirements.txt を作成
# Dockerfile を python:3.12-slim ベースで作り替え
```

### GitHub Actionsワークフローの更新
```bash
# .github/workflows/build.yaml の IMAGE_NAME を okccl/sample-backend に変更
# .github/workflows/update-gitops.yaml の参照パスを sample-backend に変更

git add -A
git commit -m "feat: replace nginx with FastAPI backend app"
git push origin main
```

---

## Step 5: sample-backendにCNPG DBを追加

### platform-chartsのChart.yamlにcommon-db依存を追加
```bash
cd ~/platform-charts/charts/sample-backend
# Chart.yaml に common-db の依存を追加
helm dependency update

# templates/all.yaml に common-db.cluster / common-db.scheduledBackup を追加
# values.yaml に db セクションを追加

cd ~/platform-charts
git add -A
git commit -m "feat: add common-db dependency to sample-backend chart"
git push origin main
```

### common-appのDeploymentテンプレートにDB環境変数を追加
CNPGが自動生成するSecret（`<cluster名>-app`）から環境変数を注入。

```bash
# charts/common-app/templates/_helpers.tpl の common-app.deployment に
# db.name が設定されている場合にSecretから環境変数を注入する処理を追加
# charts/common-app/values.yaml に db.name: "" のデフォルト値を追加

git add -A
git commit -m "feat: inject cnpg secret as env vars in common-app deployment"
git push origin main
```

### 注入される環境変数（CNPGが生成するSecret `<cluster名>-app` のキー）
| 環境変数 | Secretのキー |
|---|---|
| DB_HOST | host |
| DB_PORT | port |
| DB_NAME | dbname |
| DB_USER | username |
| DB_PASSWORD | password |

### 動作確認
```bash
kubectl exec -n sample-backend deploy/sample-backend -- python3 -c "
import urllib.request, json
res = urllib.request.urlopen('http://localhost:8000/health')
print('health:', json.loads(res.read()))
req = urllib.request.Request('http://localhost:8000/items',
    data=json.dumps({'name': 'test-item-1'}).encode(),
    headers={'Content-Type': 'application/json'}, method='POST')
res = urllib.request.urlopen(req)
print('created:', json.loads(res.read()))
res = urllib.request.urlopen('http://localhost:8000/items')
print('items:', json.loads(res.read()))
"
```

---

## Step 6: MinIOバックアップ疎通確認

### MinIO認証情報SecretをCNPG用に作成
```bash
kubectl create secret generic minio-backup-secret \
  -n sample-backend \
  --from-literal=ACCESS_KEY_ID=minioadmin \
  --from-literal=ACCESS_SECRET_KEY=minioadmin123
```

### platform-gitopsのvalues.yamlでバックアップを有効化
```yaml
db:
  backup:
    enabled: true
    endpointURL: "http://minio.minio.svc.cluster.local:9000"
    bucketName: "cnpg-backup"
    secretName: "minio-backup-secret"
```

### 手動ベースバックアップの取得
```bash
kubectl apply -f - << 'EOF'
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: sample-backend-db-manual-01
  namespace: sample-backend
spec:
  method: barmanObjectStore
  cluster:
    name: sample-backend-db
EOF

kubectl get backup -n sample-backend
```

### MinIOへの転送確認
```bash
kubectl exec -n minio deploy/minio -- mc alias set local http://localhost:9000 minioadmin minioadmin123
kubectl exec -n minio deploy/minio -- mc ls --recursive local/cnpg-backup/
```
