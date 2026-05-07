# Phase 11 作業ログ（11-6〜11-8）

## 環境情報

| 項目 | 値 |
|---|---|
| OS (WSL) | Ubuntu 24.04 LTS |
| 作業ユーザー | ccl |
| 作業日 | 2026-05-01 |

---

## 11-6: Crossplane（provider-helm）

### ファイル作成

```bash
mkdir -p ~/platform-gitops/platform/crossplane/config

# Crossplane 本体の ArgoCD Application
cat > ~/platform-gitops/platform/crossplane/application.yaml << 'EOF'
# （内容省略：sync-wave "2"、Helm Chart v2.2.1）
EOF

# crossplane-config の ArgoCD Application
cat > ~/platform-gitops/platform/crossplane/config-app.yaml << 'EOF'
# （内容省略：sync-wave "5"、SkipDryRunOnMissingResource=true）
EOF

# provider + DeploymentRuntimeConfig（wave "1"）
cat > ~/platform-gitops/platform/crossplane/config/provider.yaml << 'EOF'
# （内容省略）
EOF

# RBAC（wave "1"）
cat > ~/platform-gitops/platform/crossplane/config/rbac.yaml << 'EOF'
# （内容省略）
EOF

# ProviderConfig InjectedIdentity（wave "2"）
cat > ~/platform-gitops/platform/crossplane/config/providerconfig.yaml << 'EOF'
# （内容省略）
EOF

# podinfo デモ用 Release CR（wave "2"）
cat > ~/platform-gitops/platform/crossplane/config/demo-release.yaml << 'EOF'
# （内容省略）
EOF
```

### root.yaml の exclude リスト更新

```bash
sed -i 's|exclude: "{|exclude: "{crossplane/config/*,|' \
  ~/platform-gitops/bootstrap/root.yaml

# exclude リストを手動で root Application に反映
kubectl apply -f ~/platform-gitops/bootstrap/root.yaml
```

### コミット

```bash
cd ~/platform-gitops
git add platform/crossplane/ bootstrap/root.yaml
git commit -m "feat(crossplane): add Crossplane v2.2.1 + provider-helm v1.0.4"
git push origin main
```

### トラブル対処：CRD 未存在による sync 失敗

```bash
# sync-wave と SkipDryRunOnMissingResource を追加
# （provider.yaml / rbac.yaml に wave "1"、providerconfig.yaml / demo-release.yaml に wave "2"）
# config-app.yaml に SkipDryRunOnMissingResource=true を追加

cd ~/platform-gitops
git add platform/crossplane/
git commit -m "fix(crossplane): add sync-waves and SkipDryRunOnMissingResource"
git push origin main

kubectl apply -f ~/platform-gitops/platform/crossplane/config-app.yaml
```

### 動作確認

```bash
kubectl get providers provider-helm
kubectl get providerconfig in-cluster
kubectl get release podinfo
kubectl get pods -n crossplane-demo
```

---

## 番外編：image tag 更新を PR 方式に変更

### リポジトリ設定

```bash
# マージ後ブランチ自動削除を有効化
gh api -X PATCH repos/okccl/platform-gitops \
  -f delete_branch_on_merge=true
```

### workflow 更新

```bash
# update-gitops.yaml を PR + squash merge 方式に変更
cat > ~/platform-gitops/.github/workflows/update-gitops.yaml << 'EOF'
# （内容省略：PR 作成 → squash merge → ブランチ自動削除）
EOF

cd ~/platform-gitops
git add .github/workflows/update-gitops.yaml
git commit -m "feat(gitops): change image tag update to PR + squash merge"
git push origin main
```

### 動作確認

```bash
# テスト用に repository_dispatch を手動発火
gh api repos/okccl/platform-gitops/dispatches \
  -f event_type=update-image \
  -f 'client_payload[image_repository]=ghcr.io/okccl/sample-backend' \
  -f 'client_payload[image_tag]=test-pr-flow'

gh run list --repo okccl/platform-gitops --workflow update-gitops.yaml --limit 3
gh pr list --repo okccl/platform-gitops --state merged --limit 3

# テスト用タグを元に戻す
git pull origin main
sed -i "s/^\(\s*tag:\s*\).*/\1  3e4f9ad/" apps/sample-backend/values.yaml
git add apps/sample-backend/values.yaml
git commit -m "chore: revert sample-backend image tag to 3e4f9ad (test cleanup)"
git push origin main
```

---

## OutOfSync 解消

### crossplane の managed-by ラベル差異

```bash
# application.yaml に ignoreDifferences を追加
# （Helm が自動付与する managed-by: Helm ラベルを無視）
```

### sample-backend の replicas / protocol 差異

```bash
# KEDA が管理する replicas を ArgoCD が上書きしない設計に変更

# values.yaml から replicaCount を削除
sed -i '/^replicaCount:/d' ~/platform-gitops/apps/sample-backend/values.yaml

# Application に ignoreDifferences + RespectIgnoreDifferences=true を追加
# （/spec/replicas と /spec/template/spec/containers/0/ports/0/protocol）

cd ~/platform-gitops
git add apps/sample-backend.yaml apps/sample-backend/values.yaml
git commit -m "fix(sample-backend): let KEDA own replicas exclusively"
git push origin main
```

### gateway-config の HTTPRoute 差異

```bash
# httproute-rollouts.yaml に matches.path を明示的に追加
cat > ~/platform-gitops/platform/gateway/config/httproute-rollouts.yaml << 'EOF'
# （matches.path: PathPrefix: / を明示）
EOF

cd ~/platform-gitops
git add platform/gateway/config/httproute-rollouts.yaml
git commit -m "fix(gateway): explicitly declare PathPrefix match in rollouts HTTPRoute"
git push origin main
```

---

## 11-7: Cilium（CNI置き換え）

### platform-infra の変更

```bash
# cluster.yaml に flannel 無効化オプションを追加
cat > ~/platform-infra/k3d/cluster.yaml << 'EOF'
# --flannel-backend=none
# --disable-network-policy
# （kube-proxy は k3s に任せるため --disable-kube-proxy は追加しない）
EOF

# Makefile に install-cilium ターゲットを追加
# bootstrap 順序: cluster-create → install-cilium → fix-coredns → bootstrap-argocd → bootstrap-sync

cd ~/platform-infra
git add k3d/cluster.yaml k3d/Makefile
git commit -m "feat(cilium): replace flannel with Cilium v1.19.3"
git push origin main
```

### platform-gitops の変更

```bash
mkdir -p ~/platform-gitops/platform/cilium

cat > ~/platform-gitops/platform/cilium/application.yaml << 'EOF'
# kubeProxyReplacement: false
# ipam.mode: kubernetes
EOF

cd ~/platform-gitops
git add platform/cilium/
git commit -m "feat(cilium): add ArgoCD Application for Cilium v1.19.3"
git push origin main
```

### トラブル対処①：fix-coredns が CNI なしでハング

```bash
# 原因: flannel 無効後は CNI がないため全 Pod が Pending → CoreDNS の rollout status がブロック
# 対処: bootstrap 順序を cluster-create → install-cilium → fix-coredns に変更

cd ~/platform-infra
git add k3d/Makefile
git commit -m "fix(cilium): install-cilium before fix-coredns in bootstrap order"
git push origin main
```

### トラブル対処②：kubeProxyReplacement=true で CoreDNS が API サーバーに接続できない

```bash
# 原因: k8sServiceHost=0.0.0.0 で API サーバーへの接続が失敗
# 対処: kubeProxyReplacement=false に変更、k8sServiceHost/Port を削除

sed -i 's/--set kubeProxyReplacement=true/--set kubeProxyReplacement=false/' \
  ~/platform-infra/k3d/Makefile
sed -i '/--set k8sServiceHost/d' ~/platform-infra/k3d/Makefile
sed -i '/--set k8sServicePort/d' ~/platform-infra/k3d/Makefile

cd ~/platform-infra
git add k3d/Makefile
git commit -m "fix(cilium): disable kubeProxyReplacement for k3d stability"
git push origin main

cd ~/platform-gitops
git add platform/cilium/application.yaml
git commit -m "fix(cilium): set kubeProxyReplacement=false for local env"
git push origin main

# 既存クラスタに即時適用
helm upgrade cilium cilium/cilium \
  --version 1.19.3 \
  --namespace kube-system \
  --set kubeProxyReplacement=false \
  --set hostServices.enabled=false \
  --set externalIPs.enabled=true \
  --set nodePort.enabled=true \
  --set hostPort.enabled=true \
  --set bpf.masquerade=false \
  --set image.pullPolicy=IfNotPresent \
  --set ipam.mode=kubernetes \
  --wait --timeout=300s
```

### トラブル対処③：DaemonSet に古い k8sServiceHost 設定が残存

```bash
# 原因: Helm upgrade が DaemonSet を完全更新できていなかった
# 対処: クラスタ再作成
cd ~/platform-infra/k3d
make bootstrap
```

### Makefile の追加修正

```bash
# envoy proxy の wait タイムアウトを 120s → 300s に延長
sed -i 's/--timeout=120s/--timeout=300s/' ~/platform-infra/k3d/Makefile

# 誤ったコメントを削除
sed -i '/envoy-gatewayの起動を待機（wave1,2が完了した証拠）/d' ~/platform-infra/k3d/Makefile

cd ~/platform-infra
git add k3d/Makefile
git commit -m "fix(bootstrap): increase envoy proxy wait timeout and remove incorrect comment"
git push origin main
```

---

## sync-wave 整理

```bash
# ESO を wave 1 に追加
sed -i '/^metadata:/a\  annotations:\n    argocd.argoproj.io/sync-wave: "1"' \
  ~/platform-gitops/platform/secrets/application.yaml

# CNPG を wave 2 に追加
sed -i '/^metadata:/a\  annotations:\n    argocd.argoproj.io/sync-wave: "2"' \
  ~/platform-gitops/platform/cnpg/application.yaml

# sample-backend/frontend を wave 4 → 5 に変更
sed -i 's/sync-wave: "4"/sync-wave: "5"/' ~/platform-gitops/apps/sample-backend.yaml
sed -i 's/sync-wave: "4"/sync-wave: "5"/' ~/platform-gitops/apps/sample-frontend.yaml

# crossplane-config を wave 5 → 6 に変更
sed -i 's/sync-wave: "5"/sync-wave: "6"/' \
  ~/platform-gitops/platform/crossplane/config-app.yaml

cd ~/platform-gitops
git add platform/secrets/application.yaml platform/cnpg/application.yaml \
        apps/sample-backend.yaml apps/sample-frontend.yaml \
        platform/crossplane/config-app.yaml
git commit -m "fix(argocd): fix sync-wave ordering for proper startup sequence"
git push origin main
```

---

## platform-charts の修正

### helm dependency の Git 管理化

```bash
cd ~/platform-charts

# .gitignore から tgz 除外ルールを削除
cat > .gitignore << 'EOF'
# Helm の依存関係パッケージは Git 管理する（ArgoCD の file:// 参照に必要）
EOF

# frontend の Chart.yaml バージョンを修正（0.1.0 → 0.3.0）
sed -i 's/version: "0.1.0"/version: "0.3.0"/' \
  charts/sample-frontend/Chart.yaml

# Chart.lock を更新
cd charts/sample-frontend
helm dependency update

cd ~/platform-charts
git add charts/sample-frontend/ charts/sample-backend/charts/ .gitignore
git commit -m "fix(charts): commit helm dependencies and fix frontend common-app version"
git push origin main
```

### CI ワークフロー追加

```bash
mkdir -p ~/platform-charts/.github/workflows
cat > ~/platform-charts/.github/workflows/update-dependencies.yaml << 'EOF'
# common-app/common-db 変更時に全 application chart の依存を自動更新
EOF

cd ~/platform-charts
git add .github/workflows/update-dependencies.yaml
git commit -m "feat(ci): auto-update helm dependencies on library chart changes"
git push origin main
```

---

## Trivy の同時スキャン数制限

```bash
# concurrentScanJobsLimit: 1 / concurrentNodeScanners: 1 を追加
cat > ~/platform-gitops/platform/security/trivy-operator-app.yaml << 'EOF'
# （内容省略）
EOF

cd ~/platform-gitops
git add platform/security/trivy-operator-app.yaml
git commit -m "fix(trivy): limit concurrent scan jobs to prevent DB cache lock conflict"
git push origin main
```

---

## DR 検証（クラスタ再作成）

```bash
cd ~/platform-infra/k3d
make bootstrap
# 結果: bootstrap 約11分 / 全App Synced/Healthy 約20分 / 手動介入なし
```

---

## 11-8: ADR・Runbook 作成

```bash
# platform-adr リポジトリに以下を追加
# adr/ADR-005-crossplane.md
# adr/ADR-006-cilium.md
# adr/ADR-007-local-to-cloud.md
# runbook/Runbook-001-cnpg-wal-error.md
# runbook/Runbook-002-coredns.md
```
