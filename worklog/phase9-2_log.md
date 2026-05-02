# Phase 9-2 作業ログ

## 実施内容

### タスク1: podAntiAffinity の設定（common-app Library Chart）

```bash
cat > ~/platform-charts/charts/common-app/templates/_helpers.tpl << 'EOF'
# affinity.podAntiAffinity を spec.template.spec に追加
# preferredDuringSchedulingIgnoredDuringExecution で同名Podを異なるNodeに分散
EOF

cd ~/platform-charts
git add charts/common-app/templates/_helpers.tpl
git commit -m "feat(common-app): add podAntiAffinity to distribute pods across nodes"
git push origin main
```

---

### タスク2: replicas の変更（platform-gitops）

```bash
# platform-gitops の apps/sample-backend/values.yaml を編集
# replicaCount: 1 → 2
# db.instances: 1 → 2
cat > ~/platform-gitops/apps/sample-backend/values.yaml << 'EOF'
# replicaCount: 2, db.instances: 2 に変更
EOF

# platform-gitops の apps/sample-frontend/values.yaml を編集
# replicaCount: 1 → 2
cat > ~/platform-gitops/apps/sample-frontend/values.yaml << 'EOF'
# replicaCount: 2 に変更
EOF

cd ~/platform-gitops
git add apps/sample-backend/values.yaml apps/sample-frontend/values.yaml
git commit -m "feat(sample-backend,sample-frontend): set replicaCount 2 and cnpg instances 2 for HA"
git push origin main
```

---

### タスク3: CI/CD パイプラインの修復

#### 問題1: sample-backend の build.yaml が同リポジトリ内の update-gitops.yaml を dispatch していた
```bash
# sample-backend/.github/workflows/build.yaml を修正
# dispatch先を platform-gitops リポジトリに変更
# workflow_dispatch → repository_dispatch に変更
cat > ~/sample-backend/.github/workflows/build.yaml << 'EOF'
# trigger-update ジョブで curl による repository_dispatch を使用
# GITOPS_TOKEN を使って platform-gitops の dispatches エンドポイントを叩く
EOF

cd ~/sample-backend
git add .github/workflows/build.yaml
git commit -m "fix(ci): use repository_dispatch instead of workflow_dispatch"
git push origin main
```

#### 問題2: sample-frontend の build.yaml も同様の問題
```bash
cat > ~/sample-frontend/.github/workflows/build.yaml << 'EOF'
# trigger-update ジョブで curl による repository_dispatch を使用
EOF

cd ~/sample-frontend
git add .github/workflows/build.yaml
git commit -m "fix(ci): use repository_dispatch instead of workflow_dispatch"
git push origin main
```

#### 問題3: platform-gitops に update-gitops.yaml が存在しなかった
```bash
mkdir -p ~/platform-gitops/.github/workflows

cat > ~/platform-gitops/.github/workflows/update-gitops.yaml << 'EOF'
# repository_dispatch トリガー（event_type: update-image）
# client_payload.image_repository からアプリ名を自動判定
# apps/<APP_NAME>/values.yaml の repository と tag を sed で更新
EOF

cd ~/platform-gitops
git add .github/workflows/update-gitops.yaml
git commit -m "fix(ci): generalize update-gitops to support multiple apps"
git push origin main
```

#### 問題4: GITOPS_TOKEN のスコープ不足（repo + workflow が必要）
- GitHub の Personal Access Tokens (classic) で `repo` と `workflow` スコープを付与した新規トークンを作成
- sample-backend / sample-frontend / platform-gitops の Secrets に `GITOPS_TOKEN` として登録

#### 問題5: ghcr-pull の PAT がチャットに露出
- PAT を即時 Regenerate
- クラスタ内の ghcr-pat Secret を更新

```bash
kubectl create secret generic ghcr-pat \
  --from-literal=username=ccl \
  --from-literal=password=<新しいトークン> \
  --from-literal=email=cclima12@gmail.com \
  -n argocd \
  --dry-run=client -o yaml | kubectl apply -f -
```

- ESOによる ghcr-pull-secret の強制再同期

```bash
kubectl annotate externalsecret -n sample-app --all \
  force-sync=$(date +%s) --overwrite
```

#### 問題6: mise trust エラー（ディレクトリ名変更の影響）
```bash
mise trust ~/sample-backend
direnv allow ~/sample-backend
```

---

### 動作確認

```bash
# Pod分散配置の確認
kubectl get pods -n sample-app -o wide

# DB Primary の確認
kubectl exec -n sample-app sample-backend-db-1 -- \
  psql -U postgres -c "SELECT pg_is_in_recovery();"

kubectl get cluster -n sample-app sample-backend-db \
  -o jsonpath='{.status.currentPrimary}'
```

#### 確認結果

| Pod | Node | 状態 |
|---|---|---|
| sample-backend-5867fd9d5c-2j22b | k3d-dev-server-0 | Running |
| sample-backend-5867fd9d5c-grgl4 | k3d-dev-agent-1 | Running |
| sample-backend-db-1 | k3d-dev-agent-0 | Running（Primary） |
| sample-backend-db-2 | k3d-dev-server-0 | Running（Replica） |
| sample-frontend-64cb8fd8d4-kbt2m | k3d-dev-agent-0 | Running |
| sample-frontend-64cb8fd8d4-t77fg | k3d-dev-agent-2 | Running |

- podAntiAffinity により全アプリPodが異なるNodeに分散されていることを確認 ✅
- CNPG Primary: sample-backend-db-1（k3d-dev-agent-0） ✅
