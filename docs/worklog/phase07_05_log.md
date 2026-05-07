# Phase 7.5 作業ログ

## 環境情報

| 項目 | 値 |
|---|---|
| OS (WSL) | Ubuntu 24.04 LTS |
| 作業ユーザー | ccl |
| platform-infra | `~/platform-infra` |
| platform-gitops | `~/platform-gitops` |
| platform-charts | `~/platform-charts` |
| sample-service | `~/sample-service` |
| Gitブランチ | main |

---

## Step 1：ghcr-pull-secret の ESO 自動管理化

### 背景

GHCRからイメージをプルするための `ghcr-pull-secret` を各namespaceで手動作成していた運用を、
External Secrets Operator (ESO) による自動供給に移行した。

### 実施内容

#### 1. 事前確認

```bash
# ESO バージョン確認
helm list -n external-secrets

# 既存の ClusterSecretStore 確認
kubectl get clustersecretstore

# ghcr-pull-secret の有無確認
kubectl get secret ghcr-pull-secret -A 2>/dev/null || echo "該当Secretなし"

# Makefile に PAT 関連記述があるか確認
grep -n -i "ghcr\|pat\|token\|pull.secret" ~/platform-infra/k3d/Makefile
```

確認結果：
- `fake-store` という ClusterSecretStore が存在（デモ用）
- `ghcr-pull-secret` はどの namespace にも存在しない
- Makefile に PAT 関連の記述なし

#### 2. GitHub PAT を Kubernetes Secret として登録

GitHub の https://github.com/settings/tokens で PAT（スコープ: `read:packages`）を用意し、
`argocd` namespace に登録した。

```bash
kubectl create secret generic ghcr-pat \
  --from-literal=username=ccl \
  --from-literal=password=<PAT> \
  --from-literal=email=cclina12@gmail.com \
  -n argocd

# 作成確認
kubectl get secret ghcr-pat -n argocd
```

#### 3. マニフェストの作成

```bash
# kubernetes プロバイダーの ClusterSecretStore を既存ファイルに追記
cat >> ~/platform-gitops/platform/secrets/cluster-secret-store.yaml << 'EOF'
---
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: kubernetes-store
spec:
  provider:
    kubernetes:
      remoteNamespace: argocd
      server:
        caProvider:
          type: ConfigMap
          name: kube-root-ca.crt
          key: ca.crt
          namespace: argocd
      auth:
        serviceAccount:
          name: external-secrets
          namespace: external-secrets
EOF

# ExternalSecret 用ディレクトリを作成
mkdir -p ~/platform-gitops/platform/secrets/external-secrets

# ClusterExternalSecret マニフェストを作成
cat > ~/platform-gitops/platform/secrets/external-secrets/ghcr-pull-secret.yaml << 'EOF'
apiVersion: external-secrets.io/v1
kind: ClusterExternalSecret
metadata:
  name: ghcr-pull-secret
spec:
  namespaceSelectors:
    - matchLabels:
        ghcr-pull-secret: enabled
  refreshTime: 1h
  externalSecretSpec:
    refreshInterval: 1h
    secretStoreRef:
      name: kubernetes-store
      kind: ClusterSecretStore
    target:
      name: ghcr-pull-secret
      type: kubernetes.io/dockerconfigjson
      template:
        type: kubernetes.io/dockerconfigjson
        data:
          .dockerconfigjson: |
            {"auths":{"ghcr.io":{"username":"{{ .username }}","password":"{{ .password }}","email":"{{ .email }}","auth":"{{ printf "%s:%s" .username .password | b64enc }}"}}}
    data:
      - secretKey: username
        remoteRef:
          key: ghcr-pat
          property: username
      - secretKey: password
        remoteRef:
          key: ghcr-pat
          property: password
      - secretKey: email
        remoteRef:
          key: ghcr-pat
          property: email
EOF

# sample-service namespace にラベルを付与するマニフェストを作成
cat > ~/platform-gitops/platform/secrets/external-secrets/namespace-labels.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: sample-service
  labels:
    ghcr-pull-secret: enabled
EOF
```

#### 4. Git push と ArgoCD sync

```bash
cd ~/platform-gitops
git add platform/secrets/
git commit -m "feat: add kubernetes-store and ClusterExternalSecret for ghcr-pull-secret"
git push origin main

# root App を手動 sync
argocd app sync root --grpc-web

# 反映確認
kubectl get clustersecretstore
kubectl get clusterexternalsecret
kubectl get secret ghcr-pull-secret -n sample-service
kubectl get secret ghcr-pull-secret -n sample-service -o jsonpath='{.type}'
```

確認結果：
- `kubernetes-store`: READY=True
- `ghcr-pull-secret`（ClusterExternalSecret）: READY=True
- `sample-service` namespace に `kubernetes.io/dockerconfigjson` 型の Secret が自動生成

### 運用メモ

新しい namespace に `ghcr-pull-secret` を配布するには、対象 namespace のマニフェストに
以下のラベルを追加して push するだけでよい。

```yaml
metadata:
  labels:
    ghcr-pull-secret: enabled
```

---

## Step 2：common-app へのデフォルト resources 追加

### 確認結果

```bash
# Kyverno バックグラウンドスキャン結果確認
kubectl get policyreport -n sample-service -o yaml | grep -A 5 "result: fail" | head -40

# Deployment の resources 確認
kubectl get deployment -n sample-service \
  -o jsonpath='{.items[0].spec.template.spec.containers[0].resources}' | python3 -m json.tool

# common-app の values.yaml 確認
cat ~/platform-charts/charts/common-app/values.yaml

# sample-service の values.yaml 確認
cat ~/platform-charts/charts/sample-service/values.yaml
```

確認結果：
- `common-app/values.yaml` にはすでに resources のデフォルト値が設定済み
- `sample-service/values.yaml` にも resources が明示的に設定済み
- Kyverno の違反なし・Deployment にも正しく反映済み

**対応不要（すでに完了済み）**

---

## Step 3：Helm v4 移行の検討

### 調査結果

| 項目 | 内容 |
|---|---|
| 現在のバージョン | v3.20.1 |
| v4 最新安定版 | v4.1.3（2025年11月リリース） |
| v3 バグフィックス期限 | 2026年7月 |
| v3 セキュリティフィックス期限 | 2026年11月 |
| チャート互換性 | v2 チャート（Helm 3 のチャート）は v4 でも修正不要で動作 |

### 判断：v3 継続使用

以下の理由から、現時点では v3 を継続使用し、Phase 8 完了後に改めて移行フェーズを設ける。

- ArgoCD が `ServerSideApply=true` でリソースを管理しており、Helm v4 の SSA デフォルト化との
  フィールドオーナーシップ競合リスクがある
- v3 のセキュリティサポートは 2026年11月まで存在する
- mise でバージョン管理しているため、移行自体は容易に実施可能

---

## Step 4：PEポートフォリオ.md の更新

以下の内容を追記・更新した（Obsidian で管理しているドキュメントを直接更新）。

1. **ステップ表に Phase 7.5 を追加**
2. **Phase 8 向け注記を追加**：Gateway API 移行方針（Ingress NGINX リタイアへの対応）
3. **ツール表下に Helm v4 移行判断を追記**：判断根拠と今後の方針を記録
4. **セクション4「Phase横断的な留意事項」を新規追加**：
   - SSH 鍵の準備（パス・GitHub登録）
   - GitHub PAT の準備と登録手順
   - WSL リソース割り当て設定
