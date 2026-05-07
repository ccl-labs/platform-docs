# Phase 9 作業ログ


## 概要
Phase 9の準備として、k3dクラスタの3ノード化とMakefileのbootstrap自動化を実施した。

---

## Step 1: cluster.yaml の更新（agents: 2 → 3）

```yaml
# ~/platform-infra/k3d/cluster.yaml
agents: 3  # 変更箇所のみ記載
```

---

## Step 2: Makefile の更新

以下の改善を実施。

- `cluster-create` に冪等性を追加（既存クラスタを削除してから作成）
- `helm install` → `helm upgrade --install` に変更
- `ghcr-pat` Secret の投入を自動化（`--dry-run=client -o yaml | kubectl apply -f -`）
- `platform-charts` リポジトリ登録を追加
- `external-secrets-webhook` の起動待機を追加
- `minio-auth` / `minio-backup-secret` の投入を自動化
- `GITHUB_PAT` を環境変数で渡す形式に変更（未設定時はエラーで停止）

最終的な Makefile は引継ぎ資料を参照。

---

## Step 3: クラスタ再作成

```bash
# 既存クラスタ削除（Makefile経由）
cd ~/platform-infra/k3d
k3d cluster delete dev

# 4ノード構成で再作成
k3d cluster create --config cluster.yaml

# ノード確認
kubectl get nodes
# k3d-dev-agent-0, k3d-dev-agent-1, k3d-dev-agent-2, k3d-dev-server-0 が Ready であることを確認
```

---

## Step 4: bootstrap 実行

```bash
cd ~/platform-infra/k3d

# bootstrap-argocd のみ実行（クラスタ作成済みのため）
make bootstrap-argocd GITHUB_PAT=<PAT>

# sync 実行
make bootstrap-sync
```

### bootstrap-sync 中の手動対応

`make bootstrap-sync` がRoot App syncでハングしたため Ctrl+C で中断し、以下を手動実行。

```bash
# argocd.localhost 経由でログイン
argocd login argocd.localhost \
  --username admin \
  --password $(kubectl get secret argocd-initial-admin-secret -n argocd \
    -o jsonpath="{.data.password}" | base64 -d) \
  --insecure --grpc-web

# CRD確立待機
kubectl wait --for=condition=established \
  crd/clusterissuers.cert-manager.io --timeout=180s
kubectl wait --for=condition=available deployment/ingress-nginx-controller \
  -n ingress-nginx --timeout=300s
kubectl wait --for=condition=established \
  crd/clustersecretstores.external-secrets.io --timeout=180s
kubectl wait --for=condition=established \
  crd/clusterpolicies.kyverno.io --timeout=180s

# 残りのコンポーネントをsync
argocd app sync kyverno --async
argocd app sync kyverno-policies --async
argocd app sync kube-prometheus-stack --async
argocd app sync minio --async
argocd app sync apps-root --server-side --async

# platform-charts リポジトリ登録（抜けていたため追加）
argocd repo add git@github.com:okccl/platform-charts.git \
  --ssh-private-key-path ~/.ssh/id_ed25519

# sample-backend/frontend の sync
argocd app sync sample-backend --async
argocd app sync sample-frontend --async
```

---

## Step 5: ghcr-pull-secret のトラブルシュート

`external-secrets-webhook` が未起動のため `ClusterExternalSecret` が `Ready: False` になっていた。

```bash
# webhook 起動待機
kubectl wait --for=condition=available deployment/external-secrets-webhook \
  -n external-secrets --timeout=120s

# ClusterExternalSecret を手動リフレッシュ
kubectl annotate clusterexternalsecret ghcr-pull-secret \
  force-sync=$(date +%s) --overwrite

# 確認
kubectl get clusterexternalsecret ghcr-pull-secret
kubectl get secret -n sample-app
```

`ghcr-pull-secret` 作成確認後、ImagePullBackOff になっていた Pod を再起動。

```bash
kubectl rollout restart deployment/sample-backend -n sample-app
kubectl rollout restart deployment/sample-frontend -n sample-app
kubectl get pods -n sample-app
```

---

## Step 6: platform-infra の push

```bash
cd ~/platform-infra
git add -A
git commit -m "feat: Makefileをbootstrap自動化に対応（Phase9）"
git push origin main
```

---

## 最終確認

```bash
argocd app list
kubectl get pods -n sample-app
```

全コンポーネント Synced/Healthy、sample-backend/frontend が Running であることを確認。
