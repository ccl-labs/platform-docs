# Phase 12 作業ログ（セッション3）

## 作業概要

bootstrap の完全自動化に向けて以下を実施した。
- App-of-Apps のグループ分割による sync 順序制御の根本解決
- keycloak-platform-admin-source の SOPS 管理化
- rootCA の動的取得・Secret 注入
- Makefile の bootstrap-sync 再設計

---

## 1. App-of-Apps のディレクトリ再編

```bash
cd ~/platform-gitops

# サブディレクトリ作成
mkdir -p platform/applications/gateway-stack
mkdir -p platform/applications/auth-stack
mkdir -p platform/applications/platform
mkdir -p platform/group-roots

# gateway-stack に移動
mv platform/applications/cert-manager.yaml      platform/applications/gateway-stack/
mv platform/applications/envoy-gateway-crds.yaml platform/applications/gateway-stack/
mv platform/applications/cilium.yaml             platform/applications/gateway-stack/
mv platform/applications/envoy-gateway.yaml      platform/applications/gateway-stack/
mv platform/applications/cert-manager-config.yaml platform/applications/gateway-stack/
mv platform/applications/gateway-config.yaml     platform/applications/gateway-stack/

# auth-stack に移動
mv platform/applications/external-secrets.yaml        platform/applications/auth-stack/
mv platform/applications/cnpg.yaml                    platform/applications/auth-stack/
mv platform/applications/kyverno.yaml                 platform/applications/auth-stack/
mv platform/applications/external-secrets-config.yaml platform/applications/auth-stack/
mv platform/applications/kyverno-policies.yaml        platform/applications/auth-stack/
mv platform/applications/keycloak-db.yaml             platform/applications/auth-stack/
mv platform/applications/keycloak.yaml                platform/applications/auth-stack/
mv platform/applications/keycloak-config-cli.yaml     platform/applications/auth-stack/
mv platform/applications/kube-prometheus-stack.yaml   platform/applications/auth-stack/

# platform に移動
mv platform/applications/argocd.yaml              platform/applications/platform/
mv platform/applications/alloy.yaml               platform/applications/platform/
mv platform/applications/loki.yaml                platform/applications/platform/
mv platform/applications/tempo.yaml               platform/applications/platform/
mv platform/applications/goldilocks.yaml          platform/applications/platform/
mv platform/applications/keda.yaml                platform/applications/platform/
mv platform/applications/vpa.yaml                 platform/applications/platform/
mv platform/applications/argo-rollouts.yaml       platform/applications/platform/
mv platform/applications/trivy-operator.yaml      platform/applications/platform/
mv platform/applications/crossplane.yaml          platform/applications/platform/
mv platform/applications/vcluster-dev.yaml        platform/applications/platform/
mv platform/applications/crossplane-config.yaml   platform/applications/platform/
mv platform/applications/backstage-db.yaml        platform/applications/platform/
mv platform/applications/backstage.yaml           platform/applications/platform/
```

## 2. sync-wave の振り直し

```bash
# gateway-stack
# cert-manager-config: 4→3
sed -i 's/argocd.argoproj.io\/sync-wave: "4"/argocd.argoproj.io\/sync-wave: "3"/' \
  platform/applications/gateway-stack/cert-manager-config.yaml

# auth-stack
# cnpg: 4→2（kube-prometheus-stack wave1の後）
sed -i 's/argocd.argoproj.io\/sync-wave: "4"/argocd.argoproj.io\/sync-wave: "2"/' \
  platform/applications/auth-stack/cnpg.yaml
# kyverno: 2→1
sed -i 's/argocd.argoproj.io\/sync-wave: "2"/argocd.argoproj.io\/sync-wave: "1"/' \
  platform/applications/auth-stack/kyverno.yaml
# external-secrets-config: 4→3
sed -i 's/argocd.argoproj.io\/sync-wave: "4"/argocd.argoproj.io\/sync-wave: "3"/' \
  platform/applications/auth-stack/external-secrets-config.yaml
# kyverno-policies: 4→3
sed -i 's/argocd.argoproj.io\/sync-wave: "4"/argocd.argoproj.io\/sync-wave: "3"/' \
  platform/applications/auth-stack/kyverno-policies.yaml
# keycloak-db: 5→4
sed -i 's/argocd.argoproj.io\/sync-wave: "5"/argocd.argoproj.io\/sync-wave: "4"/' \
  platform/applications/auth-stack/keycloak-db.yaml
# keycloak: 5→5（変更なし）
# keycloak-config-cli: 6→6（変更なし）

# platform（全て wave 1 に統一、crossplane-config/backstage-dbは wave 2、backstage は wave 3）
for f in alloy loki tempo goldilocks keda vpa argo-rollouts trivy-operator crossplane vcluster-dev; do
  sed -i 's/argocd.argoproj.io\/sync-wave: "[0-9]*"/argocd.argoproj.io\/sync-wave: "1"/' \
    platform/applications/platform/${f}.yaml
done
sed -i 's/argocd.argoproj.io\/sync-wave: "[0-9]*"/argocd.argoproj.io\/sync-wave: "1"/' \
  platform/applications/platform/argocd.yaml
sed -i 's/argocd.argoproj.io\/sync-wave: "[0-9]*"/argocd.argoproj.io\/sync-wave: "2"/' \
  platform/applications/platform/crossplane-config.yaml
sed -i 's/argocd.argoproj.io\/sync-wave: "[0-9]*"/argocd.argoproj.io\/sync-wave: "2"/' \
  platform/applications/platform/backstage-db.yaml
sed -i 's/argocd.argoproj.io\/sync-wave: "[0-9]*"/argocd.argoproj.io\/sync-wave: "3"/' \
  platform/applications/platform/backstage.yaml
```

## 3. group-root Application の作成

```bash
# platform/group-roots/gateway-stack.yaml
cat > ~/platform-gitops/platform/group-roots/gateway-stack.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gateway-stack
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:okccl/platform-gitops.git
    targetRevision: HEAD
    path: platform/applications/gateway-stack
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

# platform/group-roots/auth-stack.yaml（wave: "2"）
# platform/group-roots/platform.yaml（wave: "3"、vcluster-dev.yaml を exclude）
```

## 4. root-app.yaml の変更

```bash
# path を platform/applications から platform/group-roots に変更
cat > ~/platform-gitops/platform/argocd/root-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:okccl/platform-gitops.git
    targetRevision: HEAD
    path: platform/group-roots
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

## 5. keycloak-platform-admin-source の SOPS 管理化

```bash
# template 作成
cat > ~/platform-gitops/secrets/templates/keycloak-platform-admin-source.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: keycloak-platform-admin-source
  namespace: argocd
stringData:
  password: changeme-platform-admin
EOF

# 実際のパスワードを設定して暗号化
cp ~/platform-gitops/secrets/templates/keycloak-platform-admin-source.yaml \
   ~/platform-gitops/secrets/encrypted/keycloak-platform-admin-source.yaml
# パスワードを編集後
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops encrypt --in-place ~/platform-gitops/secrets/encrypted/keycloak-platform-admin-source.yaml

# 確認
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops decrypt ~/platform-gitops/secrets/encrypted/keycloak-platform-admin-source.yaml
```

## 6. update-root-ca.sh の作成

```bash
cat > ~/platform-infra/scripts/update-root-ca.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail

# local-ca-secret が存在するまで待機
echo "local-ca-secret を待機中..."
until kubectl get secret local-ca-secret -n cert-manager &>/dev/null; do
  sleep 5
done

# CA 証明書を取得して argocd namespace に Secret として作成
echo "argocd-local-ca Secret を作成中..."
kubectl get secret local-ca-secret -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/local-ca.crt

kubectl create secret generic argocd-local-ca \
  -n argocd \
  --from-file=ca.crt=/tmp/local-ca.crt \
  --dry-run=client -o yaml | kubectl apply -f -

echo "argocd-local-ca Secret を作成しました"
EOF

chmod +x ~/platform-infra/scripts/update-root-ca.sh
```

## 7. Makefile bootstrap-sync の再設計

主な変更点：
- 全 source secrets を root sync 前に一括投入
- keycloak-platform-admin-source を追加
- gateway-stack の子 App 起動を envoy-gateway Pod で待機
- gateway-stack 完了後に CoreDNS rewrite・argocd-local-ca Secret 作成
- gateway-stack 完了後に port-forward を切り argocd.platform.local 経由に切り替え
- auth-stack の healthy を待機して完了

```bash
# 全 source secrets を sync 前に投入（追加分）
SOPS_AGE_KEY_FILE=$(SOPS_AGE_KEY) sops decrypt $(SECRETS_DIR)/keycloak-platform-admin-source.yaml \
  | kubectl apply -f -
```

## 8. values.yaml の rootCA 変更

```bash
# rootCA を $secret 参照に変更（後に機能しないことが判明）
# → argocd-cm の rootCA フィールドは $secret:key 参照に非対応
# → argocd-local-ca Secret を inject する方式に変更
# → 次セッションでローカル CA の SOPS 固定管理に切り替え予定
```

## 9. 各種 webhook 無効化（ローカル環境向け）

```bash
# kube-prometheus-stack（admission webhook と TLS を無効化）
# platform/monitoring/values.yaml に追記
prometheusOperator:
  admissionWebhooks:
    enabled: false
    patch:
      enabled: false
  tls:
    enabled: false

# ESO（webhook Pod の作成を無効化）
# platform/applications/auth-stack/external-secrets.yaml に追記
webhook:
  create: false
certController:
  create: false

# backstage app-config の sync-wave を -1 に変更
# platform/backstage/app-config/external-secret.yaml
# platform/backstage/app-config/reference-grant.yaml
```

## 10. コミット履歴

```
refactor: App-of-Apps をグループ分割（gateway-stack / auth-stack / platform）
feat: bootstrap の完全自動化
fix: bootstrap-sync の gateway 待機と argocd sync --wait を修正
fix: rootCA を Secret 参照（$argocd-local-ca:ca.crt）に変更
fix: update-root-ca.sh を git push から Secret 作成に変更、argocd sync 削除
fix: EG_SVC を $(shell) からシェル変数評価に変更
fix: kube-prometheus-stack admission webhook 無効化、backstage ExternalSecret/ReferenceGrant を wave -1 に変更
fix: auth-stack の wave を再設計（kube-prometheus-stack追加、cnpg を wave 2 に移動）
fix: ESO・CNPG の webhook 無効化キーを enabled から create に修正
fix: prometheus operator の TLS を無効化
fix: ESO・CNPG の webhook 無効化キーを enabled から create に修正
revert: CNPG webhook 無効化を取り消し（Lua health check 互換性問題のため）
fix: gateway-stack 完了後に port-forward から argocd.platform.local 経由に切り替え
```
