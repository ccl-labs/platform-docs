# Phase 12 作業ログ（12-1〜12-3）

## 12-1: Keycloak デプロイ

### SOPS 暗号化 Secret の作成

```bash
# DB 認証情報テンプレート作成
cat > ~/platform-gitops/secrets/templates/keycloak-db-credentials.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: keycloak-db-credentials-source
  namespace: argocd
stringData:
  username: keycloak
  password: changeme-keycloak-db
EOF

# 暗号化
sops --encrypt \
  --age $(cat ~/.config/sops/age/keys.txt | grep "public key" | awk '{print $4}') \
  ~/platform-gitops/secrets/templates/keycloak-db-credentials.yaml \
  > ~/platform-gitops/secrets/encrypted/keycloak-db-credentials.yaml

# admin 認証情報テンプレート作成
cat > ~/platform-gitops/secrets/templates/keycloak-admin-credentials.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: keycloak-admin-credentials-source
  namespace: argocd
stringData:
  password: changeme-keycloak-admin
EOF

# 暗号化
sops --encrypt \
  --age $(cat ~/.config/sops/age/keys.txt | grep "public key" | awk '{print $4}') \
  ~/platform-gitops/secrets/templates/keycloak-admin-credentials.yaml \
  > ~/platform-gitops/secrets/encrypted/keycloak-admin-credentials.yaml
```

### Makefile への Secret 投入処理追加（platform-infra）

`~/platform-infra/k3d/Makefile` の `bootstrap-sync` ターゲットに以下を追加：

```makefile
SOPS_AGE_KEY_FILE=$(SOPS_AGE_KEY) sops decrypt $(SECRETS_DIR)/keycloak-db-credentials.yaml \
        | kubectl apply -f -
SOPS_AGE_KEY_FILE=$(SOPS_AGE_KEY) sops decrypt $(SECRETS_DIR)/keycloak-admin-credentials.yaml \
        | kubectl apply -f -
```

### platform-gitops ファイル構成

```
platform/keycloak/
  keycloak-db-app.yaml       # ArgoCD Application（wave 3）
  keycloak-app.yaml          # ArgoCD Application（wave 5）
  db-config/
    namespace.yaml
    external-secret.yaml
    cnpg-cluster.yaml
  app-config/
    reference-grant.yaml
  values.yaml
platform/gateway/config/
  httproute-keycloak.yaml
```

### bootstrap/root.yaml の exclude 更新

```yaml
exclude: "{...,keycloak/db-config/*,keycloak/app-config/*}"
```

### 手動 Secret 投入（初回のみ）

```bash
kubectl create namespace keycloak --dry-run=client -o yaml | kubectl apply -f -

SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops decrypt \
  ~/platform-gitops/secrets/encrypted/keycloak-db-credentials.yaml \
  | kubectl apply -f -

SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops decrypt \
  ~/platform-gitops/secrets/encrypted/keycloak-admin-credentials.yaml \
  | kubectl apply -f -
```

### ArgoCD sync

```bash
argocd app sync root --server-side --grpc-web
argocd app sync keycloak-db --server-side --grpc-web
argocd app sync keycloak --server-side --grpc-web
argocd app sync gateway-config --server-side --grpc-web
```

### 動作確認

```bash
kubectl get pods -n keycloak
# NAME                   READY   STATUS    RESTARTS   AGE
# keycloak-db-1          1/1     Running   0          xx
# keycloak-keycloakx-0   1/1     Running   0          xx

curl -s -o /dev/null -w "%{http_code}" http://keycloak.platform.local/
# 302
```

---

## 12-2: root App の自己管理化

```bash
# bootstrap/root.yaml を platform/argocd/ にコピー
cp ~/platform-gitops/bootstrap/root.yaml \
   ~/platform-gitops/platform/argocd/root-app.yaml

git add platform/argocd/root-app.yaml
git commit -m "feat(argocd): manage root App by ArgoCD itself"
git push origin main

argocd app sync root --server-side --grpc-web
```

---

## 12-3: ArgoCD × Keycloak SSO

### ドメイン変更（*.localhost → *.platform.local）

```bash
# platform-gitops 内の全置換
find ~/platform-gitops -name "*.yaml" | xargs sed -i 's/\.localhost/.platform.local/g'

# platform-infra/k3d/Makefile も更新
sed -i 's/keycloak\.localhost/keycloak.platform.local/g' ~/platform-infra/k3d/Makefile
sed -i 's/argocd\.localhost/argocd.platform.local/g' ~/platform-infra/k3d/Makefile
```

### Windows hosts ファイルへの追記（管理者 PowerShell）

```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "172.17.53.188 keycloak.platform.local"
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "172.17.53.188 argocd.platform.local"
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "172.17.53.188 grafana.platform.local"
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "172.17.53.188 backstage.platform.local"
```

### CoreDNS への rewrite 追加（Makefile による自動化）

`bootstrap-sync` の Envoy Gateway 待機後に以下を追加（Makefile）：

```makefile
# CoreDNS に *.platform.local の rewrite を追加（Envoy Gateway 起動後）
$(eval EG_SVC := $(shell kubectl get svc -n envoy-gateway-system \
    -l gateway.envoyproxy.io/owning-gateway-name=eg \
    -o jsonpath='{.items[0].metadata.name}'))
kubectl get configmap coredns -n kube-system -o json \
    | python3 -c "import sys,json; \
      d=json.load(sys.stdin); \
      cf=d['data']['Corefile']; \
      rule='    rewrite name keycloak.platform.local $(EG_SVC).envoy-gateway-system.svc.cluster.local\n    rewrite name argocd.platform.local $(EG_SVC).envoy-gateway-system.svc.cluster.local\n'; \
      d['data']['Corefile']=cf.replace('ready\n', 'ready\n'+rule) if 'rewrite name keycloak.platform.local' not in cf else cf; \
      print(json.dumps(d))" \
    | kubectl apply -f -
kubectl rollout restart deployment coredns -n kube-system
kubectl rollout status deployment coredns -n kube-system
```

### argocd-keycloak-secret の作成

```bash
# テンプレート作成（Client Secret は Keycloak GUI から取得）
cat > ~/platform-gitops/secrets/templates/argocd-keycloak-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: argocd-keycloak-secret-source
  namespace: argocd
stringData:
  clientSecret: <KEYCLOAK_CLIENT_SECRET>
EOF

# 暗号化
sops --encrypt \
  --age $(cat ~/.config/sops/age/keys.txt | grep "public key" | awk '{print $4}') \
  ~/platform-gitops/secrets/templates/argocd-keycloak-secret.yaml \
  > ~/platform-gitops/secrets/encrypted/argocd-keycloak-secret.yaml

# クラスタに投入
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops decrypt \
  ~/platform-gitops/secrets/encrypted/argocd-keycloak-secret.yaml \
  | kubectl apply -f -
```

Makefile にも追加：
```makefile
SOPS_AGE_KEY_FILE=$(SOPS_AGE_KEY) sops decrypt $(SECRETS_DIR)/argocd-keycloak-secret.yaml \
        | kubectl apply -f -
```

### platform/secrets/config/argocd-keycloak-secret.yaml（ESO）

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: argocd-keycloak-secret
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: kubernetes-store
    kind: ClusterSecretStore
  target:
    name: argocd-keycloak-secret
    template:
      metadata:
        labels:
          app.kubernetes.io/part-of: argocd  # ArgoCD が Secret を参照するために必須
  data:
    - secretKey: clientSecret
      remoteRef:
        key: argocd-keycloak-secret-source
        property: clientSecret
```

### platform/argocd/values.yaml（OIDC 設定）

```yaml
configs:
  params:
    server.insecure: "true"
  cm:
    url: "http://argocd.platform.local"
    oidc.config: |
      name: Keycloak
      issuer: http://keycloak.platform.local/auth/realms/platform
      clientID: argocd
      clientSecret: $argocd-keycloak-secret:clientSecret
      requestedScopes:
        - openid
        - profile
        - email
        - groups
      insecureSkipVerify: true
  rbac:
    policy.csv: |
      g, argocd-admins, role:admin
    policy.default: role:readonly
    scopes: "[groups]"

ingress:
  enabled: false
```

### Keycloak GUI での設定（bootstrap 後に毎回必要）

1. `platform` Realm を作成
2. `argocd` Client を作成
   - Client authentication: ON
   - Valid redirect URIs: `http://argocd.platform.local/auth/callback`
   - Web origins: `http://argocd.platform.local`
3. `groups` Client Scope を作成（Group Membership mapper 付き、Full group path: OFF）
4. `argocd` Client に `groups` scope を追加
5. `argocd-admins` Group を作成
6. `platform-admin` ユーザーを作成し `argocd-admins` に Join

### argocd CLI の接続設定

```bash
argocd login argocd.platform.local:80 \
  --plaintext \
  --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath='{.data.password}' | base64 -d)
```

### external-secrets-config の ignoreDifferences 更新

`ExternalSecret` kind を追加：

```yaml
ignoreDifferences:
  - group: external-secrets.io
    kind: ExternalSecret
    jqPathExpressions:
      - .spec.data[].remoteRef.conversionStrategy
      - .spec.data[].remoteRef.decodingStrategy
      - .spec.data[].remoteRef.metadataPolicy
      - .spec.data[].remoteRef.nullBytePolicy
```
