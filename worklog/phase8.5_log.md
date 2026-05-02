# Phase 8.5 作業ログ

## Step 1: GitHubリポジトリ名リネーム（sample-service → sample-backend）

### GitHub Web UI での操作
- `https://github.com/okccl/sample-service` → Settings → Repository name を `sample-backend` に変更

### ローカルのリモートURL更新
```bash
cd ~/sample-service
git remote set-url origin git@github.com:okccl/sample-backend.git
git remote -v
```

---

## Step 2: namespace・各種リソース名を sample-backend → sample-app にリネーム

### 変更対象ファイルの確認
```bash
grep -r "sample-backend" ~/platform-gitops --include="*.yaml" -l
grep -r "sample-backend" ~/platform-charts --include="*.yaml" -l
grep -r "sample-backend" ~/platform-charts --include="*.json" -l
```

### platform-charts の変更
```bash
cd ~/platform-charts

cp -r charts/sample-backend charts/sample-app

sed -i 's/name: sample-backend/name: sample-app/' charts/sample-app/Chart.yaml
sed -i 's/description: サンプルバックエンドアプリ/description: サンプルアプリ（backend）/' charts/sample-app/Chart.yaml
sed -i 's/name: sample-backend$/name: sample-app/' charts/sample-app/values.yaml
sed -i 's/name: sample-backend-db/name: sample-app-db/' charts/sample-app/values.yaml

grep -n "sample-backend\|sample-app" charts/sample-app/Chart.yaml
grep -n "sample-backend\|sample-app" charts/sample-app/values.yaml

rm -rf charts/sample-backend
ls charts/

git add -A
git commit -m "feat: rename sample-backend chart to sample-app"
git push origin main
```

### platform-gitops の変更
```bash
cd ~/platform-gitops

cp -r apps/sample-backend apps/sample-app
sed -i 's/name: sample-backend$/name: sample-app/' apps/sample-app/values.yaml
sed -i 's/name: sample-backend-db/name: sample-app-db/' apps/sample-app/values.yaml
grep -n "sample-backend\|sample-app" apps/sample-app/values.yaml

cp apps/sample-backend.yaml apps/sample-app.yaml
sed -i 's/name: sample-backend/name: sample-app/' apps/sample-app.yaml
sed -i 's/path: charts\/sample-backend/path: charts\/sample-app/' apps/sample-app.yaml
sed -i 's/apps\/sample-backend\/values.yaml/apps\/sample-app\/values.yaml/' apps/sample-app.yaml
sed -i 's/namespace: sample-backend/namespace: sample-app/' apps/sample-app.yaml
grep -n "sample-backend\|sample-app" apps/sample-app.yaml

sed -i 's/name: sample-backend/name: sample-app/' platform/secrets/external-secrets/namespace-labels.yaml
grep -n "sample-backend\|sample-app" platform/secrets/external-secrets/namespace-labels.yaml

rm apps/sample-backend.yaml
rm -rf apps/sample-backend

git add -A
git commit -m "feat: rename sample-backend to sample-app"
git push origin main
```

### クラスタ上のリソース切り替え
```bash
argocd login argocd.localhost --username admin --password $(kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d) --insecure --grpc-web

argocd app delete sample-backend --cascade=false

argocd app sync sample-app --grpc-web
argocd app get sample-app --grpc-web

kubectl delete namespace sample-backend
```

---

## Step 3: sample-backend に OpenTelemetry を組み込む

### requirements.txt の更新
```bash
cat > ~/sample-service/src/requirements.txt << 'EOF'
fastapi==0.115.12
uvicorn==0.34.0
asyncpg==0.30.0
pydantic==2.11.3
prometheus-client==0.21.1
opentelemetry-sdk==1.33.0
opentelemetry-exporter-otlp-proto-http==1.33.0
opentelemetry-instrumentation-fastapi==0.54b0
EOF
```

### main.py の更新（OTel初期化・CORS追加）
```bash
# OTel初期化・FastAPIInstrumentor・CORSMiddleware・手動スパンを追加
# 詳細は ~/sample-service/src/main.py を参照
```

### common-app テンプレートに任意 env サポートを追加
```bash
cd ~/platform-charts

python3 - << 'EOF'
with open("charts/common-app/templates/_helpers.tpl", "r") as f:
    content = f.read()

old = "          {{- end }}\n          livenessProbe:"
new = """          {{- end }}
          {{- if .Values.env }}
          {{- toYaml .Values.env | nindent 12 }}
          {{- end }}
          livenessProbe:"""

content = content.replace(old, new)

with open("charts/common-app/templates/_helpers.tpl", "w") as f:
    f.write(content)
EOF

sed -n '57,68p' charts/common-app/templates/_helpers.tpl

git add -A
git commit -m "feat: add optional env vars support to common-app"
git push origin main
```

### platform-gitops に OTLP エンドポイントを追加
```bash
cd ~/platform-gitops

cat >> apps/sample-app/values.yaml << 'EOF'
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://tempo.tracing.svc.cluster.local:4318"
  - name: OTEL_SERVICE_NAME
    value: "sample-backend"
EOF

git add -A
git commit -m "chore: update sample-backend image tag to 7c14aad"
git push origin main
```

### イメージビルド・デプロイ
```bash
cd ~/sample-service
git add -A
git commit -m "feat: add OpenTelemetry instrumentation"
git push origin main

# GitHub Actions ビルド完了後
cd ~/platform-gitops
sed -i 's/tag: .*/tag: 7c14aad/' apps/sample-app/values.yaml
git add -A
git commit -m "chore: update sample-backend image tag to 7c14aad"
git push origin main

argocd app sync sample-app --grpc-web
```

### Tempo probe 設定の緩和（再起動ループ対応）
```bash
cat >> ~/platform-gitops/platform/tracing/tempo-values.yaml << 'EOF'
  livenessProbe:
    initialDelaySeconds: 60
    periodSeconds: 30
    timeoutSeconds: 10
    failureThreshold: 5
  readinessProbe:
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 10
    failureThreshold: 5
EOF

git add -A
git commit -m "fix: relax tempo liveness/readiness probe settings"
git push origin main
```

---

## Step 4: sample-frontend リポジトリ新規作成（Vite + React）

### GitHub Web UI での操作
- `https://github.com/new` → Repository name: `sample-frontend` で新規作成

### ローカルへの clone とアプリ初期化
```bash
cd ~
git clone git@github.com:okccl/sample-frontend.git
cd sample-frontend

cp ~/platform-infra/.mise.toml .
cp ~/platform-infra/.envrc .
direnv allow .
mise trust .

# node のインストール
mise use node@lts
eval "$(mise activate bash)"

# Vite + React アプリ初期化
npm create vite@latest . -- --template react
npm install
npm run build
```

### アプリ実装・Dockerfile 作成
```bash
# src/App.jsx を backend API 呼び出しUI に書き換え
# Dockerfile（node:24-alpine でビルド → nginx:alpine で配信）を作成
# nginx.conf（SPA ルーティング対応）を作成

cat > .gitignore << 'EOF'
node_modules/
dist/
.env
.env.local
EOF

git add -A
git commit -m "feat: initial Vite+React app with nginx Dockerfile"
git push origin main
```

---

## Step 5: sample-frontend Helm Chart 作成

```bash
cd ~/platform-charts
mkdir -p charts/sample-frontend/templates

# Chart.yaml・values.yaml・templates/all.yaml を作成

cd charts/sample-frontend
helm dependency update

cd ~/platform-charts
git add -A
git commit -m "feat: add sample-frontend Helm chart"
git push origin main
```

---

## Step 6: sample-frontend ArgoCD Application 登録

```bash
cd ~/platform-gitops

# apps/sample-frontend.yaml を作成（destination namespace: sample-app）
# apps/sample-frontend/values.yaml を作成

git add -A
git commit -m "feat: add sample-frontend ArgoCD Application"
git push origin main
```

### GitHub Actions ワークフロー追加
```bash
cd ~/sample-frontend
mkdir -p .github/workflows
# build.yaml・update-gitops.yaml を作成

git add -A
git commit -m "ci: add GitHub Actions workflows"
git push origin main
```

### イメージタグ更新・pullSecretName 追加
```bash
# GitHub Actions ビルド完了後
cd ~/platform-gitops

# pullSecretName が未設定だったため追加
sed -i 's/  pullPolicy: IfNotPresent/  pullPolicy: IfNotPresent\n  pullSecretName: ghcr-pull-secret/' apps/sample-frontend/values.yaml

sed -i 's/tag: latest/tag: cfeb206/' apps/sample-frontend/values.yaml

git add -A
git commit -m "fix: add pullSecretName to sample-frontend values"
git push origin main

argocd app sync sample-frontend --grpc-web
```

---

## Step 7: 動作確認

```bash
# Pod 確認
kubectl get pods -n sample-app

# backend 疎通確認
curl -Lk https://sample-backend.localhost/health
curl -Lk https://sample-backend.localhost/items
curl -Lk -X POST https://sample-backend.localhost/items \
  -H "Content-Type: application/json" \
  -d '{"name": "trace-test"}'

# Tempo 疎通確認（Pod 内から）
kubectl exec -n sample-app deployment/sample-app -- python3 -c "
import urllib.request
req = urllib.request.Request(
    'http://tempo.tracing.svc.cluster.local:4318/v1/traces',
    data=b'{}',
    headers={'Content-Type': 'application/json'},
    method='POST'
)
res = urllib.request.urlopen(req, timeout=5)
print('OK:', res.status)
"
```

### CORS 対応（sample-backend に CORSMiddleware 追加）
```bash
# main.py に CORSMiddleware を追加
# allow_origins: ["https://sample-frontend.localhost"]

cd ~/sample-service
git add -A
git commit -m "fix: add CORS middleware for sample-frontend"
git push origin main

# GitHub Actions ビルド完了後
cd ~/platform-gitops
sed -i 's/tag: 7c14aad/tag: cd19f62/' apps/sample-app/values.yaml
git add -A
git commit -m "chore: update sample-backend image tag to cd19f62"
git push origin main

argocd app sync sample-app --grpc-web
kubectl rollout status deployment/sample-app -n sample-app
```
