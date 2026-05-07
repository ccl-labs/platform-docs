# Phase 12 作業ログ

## 12-5: Backstage × Keycloak 連携

### Keycloak で backstage Client を作成（手動）

- `http://keycloak.platform.local` → `platform` Realm
- Client ID: `backstage`、Client authentication: ON
- Valid redirect URIs: `http://backstage.platform.local/api/auth/oidc/handler/frame`
- Web origins: `http://backstage.platform.local`
- Credentials タブから Client Secret をコピー

### SOPS Secret の作成・暗号化・適用

```bash
cat > ~/platform-gitops/secrets/templates/backstage-keycloak-secret-source.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: backstage-keycloak-secret-source
  namespace: argocd
stringData:
  clientSecret: "<Keycloak からコピーした値>"
EOF

sops --encrypt \
  --age $(cat ~/.config/sops/age/keys.txt | grep "public key" | awk '{print $4}') \
  ~/platform-gitops/secrets/templates/backstage-keycloak-secret-source.yaml \
  > ~/platform-gitops/secrets/encrypted/backstage-keycloak-secret-source.yaml

SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops decrypt \
  ~/platform-gitops/secrets/encrypted/backstage-keycloak-secret-source.yaml \
  | kubectl apply -f -

kubectl get secret backstage-keycloak-secret-source -n argocd
```

### ExternalSecret に KEYCLOAK_CLIENT_SECRET を追加

`platform/backstage/app-config/external-secret.yaml` に以下のエントリを追加：

```yaml
- secretKey: KEYCLOAK_CLIENT_SECRET
  remoteRef:
    key: backstage-keycloak-secret-source
    property: clientSecret
```

### sync-wave を -1 に変更（Secret 先行作成のため）

ExternalSecret の `argocd.argoproj.io/sync-wave` を `"4"` から `"-1"` に変更。
Deployment より先に Secret が作成されるようにすることで、Secret 未作成による
Deployment のハングを防ぐ。

```bash
# ExternalSecret だけ先に sync
argocd app sync backstage --resource external-secrets.io:ExternalSecret:backstage-secret --server-side --grpc-web
kubectl wait externalsecret backstage-secret -n backstage --for=condition=Ready --timeout=60s
kubectl get secret backstage-secret -n backstage -o jsonpath='{.data}' | jq 'keys'
```

### values.yaml に OIDC 設定を追加

`platform/backstage/values.yaml` に以下を追加：

```yaml
auth:
  environment: development
  providers:
    oidc:
      development:
        metadataUrl: http://keycloak.platform.local/realms/platform/.well-known/openid-configuration
        clientId: backstage
        clientSecret: ${KEYCLOAK_CLIENT_SECRET}
        prompt: auto
        signIn:
          resolvers:
            - resolver: emailMatchingUserEntityProfileEmail
```

※ 引き継ぎ資料の metadataUrl に `/auth/` が含まれていたが、Keycloak 26.x では
`/auth` パスが廃止されているため `/realms/platform/...` に修正した。

### Backstage イメージタグの固定

`latest` タグで `NotImplementedError: No implementation available for apiRef{plugin.notifications.service}`
が発生。フロントエンドバンドルに焼き込まれたプラグインと実行時 API バインディングの不一致が原因。

GHCR のタグ形式は GitHub Releases の `v1.x.x` とは異なり `1.x.x`（v プレフィックスなし）。
利用可能なタグを GitHub API で確認：

```bash
curl -s "https://api.github.com/repos/backstage/backstage/releases?per_page=5" \
  | python3 -c "import sys,json; [print(r['tag_name']) for r in json.load(sys.stdin)]"
# → v1.50.4 等が返るが GHCR には存在しない
# GHCR に存在するタグは別途確認が必要（例: 1.48.1）
```

```bash
sed -i 's/tag: latest/tag: 1.48.1/' ~/platform-gitops/platform/backstage/values.yaml
```

### Backstage DB の再作成（マイグレーション不一致の解消）

`latest` イメージが適用した新しいマイグレーションを `1.48.1` が認識できず起動失敗。
DB を作り直して解消：

```bash
kubectl scale deployment backstage -n backstage --replicas=0
kubectl delete cluster backstage-db -n backstage
argocd app sync backstage-db --server-side --grpc-web
kubectl wait pod -n backstage -l cnpg.io/cluster=backstage-db --for=condition=Ready --timeout=120s
kubectl scale deployment backstage -n backstage --replicas=1
kubectl rollout status deployment backstage -n backstage --timeout=180s
```

### dangerouslyDisableDefaultAuthPolicy の設定

Guest ユーザーが catalog API に 401 を返す問題を解消：

```yaml
backend:
  auth:
    dangerouslyDisableDefaultAuthPolicy: true
```

### notifications の無効化

```yaml
notifications:
  enabled: false
```

### Git push・sync

```bash
cd ~/platform-gitops
git add platform/backstage/ secrets/encrypted/backstage-keycloak-secret-source.yaml
git commit -m "feat(backstage): add Keycloak OIDC integration (12-5)"
git push

argocd app sync backstage --server-side --grpc-web
```

---

## 12-6: vCluster

### ArgoCD Application と values.yaml の作成

```bash
mkdir -p ~/platform-gitops/platform/vcluster

# vcluster-app.yaml 作成
cat > ~/platform-gitops/platform/vcluster/vcluster-app.yaml << 'EOF'
# （省略 - platform/vcluster/vcluster-app.yaml 参照）
EOF

# values.yaml 作成
cat > ~/platform-gitops/platform/vcluster/values.yaml << 'EOF'
# （省略 - platform/vcluster/values.yaml 参照）
EOF

cd ~/platform-gitops
git add platform/vcluster/
git commit -m "feat(vcluster): add vcluster-dev managed by ArgoCD (12-6)"
git push
```

### ignoreDifferences の追加（StatefulSet volumeClaimTemplates）

StatefulSet の `volumeClaimTemplates` に `apiVersion`・`kind` が含まれるかどうかの
差分で OutOfSync になる ArgoCD の既知問題を ignoreDifferences で解消：

```bash
git add platform/vcluster/vcluster-app.yaml
git commit -m "fix(vcluster): ignore StatefulSet volumeClaimTemplates diff"
git push
argocd app sync vcluster-dev --server-side --grpc-web
```

### Kyverno ポリシー対応（LimitRange の有効化）

vCluster からホストクラスタへの Pod 同期時に `require-resource-limits` ポリシーが
ブロック。`policies.limitRange` を有効化して解消：

```yaml
policies:
  limitRange:
    enabled: true
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```

### 動作確認

```bash
# port-forward（別ターミナル）
kubectl port-forward svc/vcluster-dev 8443:443 -n vcluster-dev

# kubeconfig 取得・接続確認
kubectl get secret vcluster-dev-kubeconfig -n vcluster-dev \
  -o jsonpath='{.data.config}' | base64 -d \
  | sed "s|https://localhost:8443|https://127.0.0.1:8443|" \
  > /tmp/vcluster-dev.kubeconfig

KUBECONFIG=/tmp/vcluster-dev.kubeconfig kubectl get node --insecure-skip-tls-verify
# → k3d-dev-server-0 が表示される

# テスト Pod のデプロイ・ホストクラスタへの同期確認
KUBECONFIG=/tmp/vcluster-dev.kubeconfig kubectl run nginx-test --image=nginx:alpine --insecure-skip-tls-verify
kubectl get pod -n vcluster-dev
# → nginx-test-x-default-x-vcluster-dev が Running

# テスト Pod の削除
KUBECONFIG=/tmp/vcluster-dev.kubeconfig kubectl delete pod nginx-test --insecure-skip-tls-verify
```

---

## 12-7: Backstage × vCluster 連携

### GitHub integration の設定

既存の `ghcr-pat` は `read:packages` スコープのみで PR 作成不可。
`repo` スコープを持つ専用 PAT を新規登録：

```bash
# テンプレート作成・暗号化・適用
cat > ~/platform-gitops/secrets/templates/backstage-github-token-source.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: backstage-github-token-source
  namespace: argocd
stringData:
  token: "<repo スコープの PAT>"
EOF

sops --encrypt \
  --age $(cat ~/.config/sops/age/keys.txt | grep "public key" | awk '{print $4}') \
  ~/platform-gitops/secrets/templates/backstage-github-token-source.yaml \
  > ~/platform-gitops/secrets/encrypted/backstage-github-token-source.yaml

SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops decrypt \
  ~/platform-gitops/secrets/encrypted/backstage-github-token-source.yaml \
  | kubectl apply -f -
```

ExternalSecret に `GITHUB_TOKEN` エントリを追加（`backstage-github-token-source` を参照）。

`values.yaml` に GitHub integration と catalog location を追加：

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

catalog:
  locations:
    - type: url
      target: https://github.com/okccl/platform-gitops/blob/main/backstage/templates/vcluster/template.yaml
      rules:
        - allow: [Template]
```

### Software Template の作成

```bash
mkdir -p ~/platform-gitops/backstage/templates/vcluster/skeleton

# template.yaml・skeleton/vcluster-app.yaml・skeleton/values.yaml を作成
# （backstage/templates/vcluster/ 参照）

cd ~/platform-gitops
git add backstage/
git commit -m "feat(backstage): add vCluster Software Template (12-7)"
git push
```

### ignoreDifferences の更新（ExternalSecret index 2 追加）

GITHUB_TOKEN エントリ追加により ExternalSecret の data が3件になったため
`backstage-app.yaml` の ignoreDifferences に index 2 のエントリを追加：

```bash
git add platform/backstage/backstage-app.yaml
git commit -m "fix(backstage): add ignoreDifferences for GITHUB_TOKEN ExternalSecret entry"
git push
argocd app sync backstage --server-side --grpc-web
```

### 動作確認

1. `http://backstage.platform.local` → Guest → **Create...**
2. **vCluster 払い出し** テンプレートを選択
3. クラスタ名: `dev-test` を入力して実行
4. GitHub に PR `feat(vcluster): add vcluster-dev-test` が作成されることを確認
5. PR をマージ
6. ArgoCD が `vcluster-dev-test` Application を自動作成することを確認

```bash
kubectl get application -n argocd | grep vcluster
# vcluster-dev       Synced  Healthy
# vcluster-dev-test  Synced  Healthy

kubectl get pod -n vcluster-dev-test
# vcluster-dev-test-0  Running
```
