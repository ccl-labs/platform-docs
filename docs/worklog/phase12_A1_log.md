# Phase 12 クリーンアップ 作業ログ

## セッション概要

Phase 12 完了後の持ち越し課題のうち、ローカル環境で対応可能なものを整理・解消したセッション。
作業中に bootstrap の構造的な問題が発覚し、App-of-Apps の整理・sync-wave の修正・Makefile の改善を合わせて実施した。

---

## B. vCluster ResourceQuota の明示的設定

```bash
# values.yaml に resourceQuota セクションを追記
cat >> ~/platform-gitops/platform/vcluster/values.yaml << 'EOF'
  resourceQuota:
    enabled: true
    quota:
      limits.cpu: "4"
      limits.memory: 8Gi
      requests.cpu: "2"
      requests.memory: 4Gi
      requests.storage: 20Gi
      count/pods: "20"
      count/services: "20"
      count/secrets: "100"
      count/configmaps: "100"
      count/persistentvolumeclaims: "20"
      services.nodeports: "0"
      services.loadbalancers: "1"
EOF

git add platform/vcluster/values.yaml
git commit -m "feat(vcluster): ResourceQuota を明示的に設定"
git push origin main

argocd app wait vcluster-dev --grpc-web
kubectl describe ns vcluster-dev
```

---

## C. Kyverno opt-in モデルへの変更

```bash
# 3つのポリシーファイルを書き換え（exclude 削除・match に namespaceSelector 追加）
cat > ~/platform-gitops/platform/policy/policies/add-default-labels.yaml << 'EOF'
# （ファイル内容省略 - match に policy-enforced: "true" の namespaceSelector を追加）
EOF

cat > ~/platform-gitops/platform/policy/policies/disallow-latest-tag.yaml << 'EOF'
# （ファイル内容省略 - 同上）
EOF

cat > ~/platform-gitops/platform/policy/policies/require-resource-limits.yaml << 'EOF'
# （ファイル内容省略 - 同上）
EOF

# sample-backend に policy-enforced: "true" ラベルを追加
# managedNamespaceMetadata.labels に追記

# platform-namespaces.yaml を削除（→ argocd namespace ごと消えてクラスタ崩壊）
git rm platform/policy/platform-namespaces.yaml  # ← 失敗の原因

git add apps/sample-backend.yaml
git add platform/policy/policies/
git commit -m "feat(kyverno): opt-out から opt-in モデルへ変更"
git push origin main
```

### ⚠️ 失敗と教訓

`platform-namespaces.yaml` を削除したことで ArgoCD が prune により `argocd` namespace ごと削除し、クラスタが自己崩壊した。
**教訓：App-of-Apps で管理している namespace を定義するファイルは、ポリシー変更のためであっても削除してはならない。**

クラスタを `make bootstrap` で再作成。

---

## bootstrap 構造改善（C 作業中の脱線対応）

### App-of-Apps の整理（platform/applications/ への集約）

```bash
# applications/ ディレクトリを作成し Application yaml を集約
mkdir -p ~/platform-gitops/platform/applications

cd ~/platform-gitops/platform
mv argocd/application.yaml          applications/argocd.yaml
mv backstage/backstage-app.yaml     applications/backstage.yaml
mv backstage/backstage-db-app.yaml  applications/backstage-db.yaml
mv cilium/application.yaml          applications/cilium.yaml
mv cnpg/application.yaml            applications/cnpg.yaml
mv crossplane/application.yaml      applications/crossplane.yaml
mv crossplane/config-app.yaml       applications/crossplane-config.yaml
mv gateway/envoy-gateway-app.yaml        applications/envoy-gateway.yaml
mv gateway/envoy-gateway-crds-app.yaml   applications/envoy-gateway-crds.yaml
mv gateway/gateway-config-app.yaml       applications/gateway-config.yaml
mv goldilocks/application.yaml      applications/goldilocks.yaml
mv ingress/cert-manager-app.yaml         applications/cert-manager.yaml
mv ingress/cert-manager-config-app.yaml  applications/cert-manager-config.yaml
mv keycloak/keycloak-app.yaml       applications/keycloak.yaml
mv keycloak/keycloak-db-app.yaml    applications/keycloak-db.yaml
mv logging/alloy-application.yaml   applications/alloy.yaml
mv logging/loki-application.yaml    applications/loki.yaml
mv monitoring/application.yaml      applications/kube-prometheus-stack.yaml
mv policy/kyverno-policies.yaml     applications/kyverno-policies.yaml
mv policy/kyverno.yaml              applications/kyverno.yaml
mv scaling/keda-app.yaml            applications/keda.yaml
mv secrets/application.yaml              applications/external-secrets.yaml
mv secrets/external-secrets-config-app.yaml  applications/external-secrets-config.yaml
mv security/argo-rollouts-app.yaml  applications/argo-rollouts.yaml
mv security/trivy-operator-app.yaml applications/trivy-operator.yaml
mv tracing/tempo-application.yaml   applications/tempo.yaml
mv vcluster/vcluster-app.yaml       applications/vcluster-dev.yaml
mv vpa/application.yaml             applications/vpa.yaml
```

### root-app.yaml の更新

```bash
# path を platform/applications に変更・directory.exclude で vcluster-dev を除外
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
    path: platform/applications
    directory:
      exclude: vcluster-dev.yaml
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

git add platform/applications/
git add platform/argocd/root-app.yaml
git add -u
git commit -m "refactor: Application yaml を platform/applications/ に集約し root App を簡素化"
git push origin main
```

### bootstrap/root.yaml の廃止と Makefile 修正

```bash
# bootstrap/root.yaml を削除（platform/argocd/root-app.yaml に一本化）
cd ~/platform-gitops
git rm bootstrap/root.yaml

# bootstrap/apps-root.yaml を残しつつ Makefile のターゲットを分離
# platform-infra/k3d/Makefile:
#   bootstrap-argocd: root-app.yaml を直接参照、apps-root.yaml の apply を削除
#   bootstrap-apps: 新規追加（アプリ層の展開・sample-app Secret 投入）
#   bootstrap-sync: Keycloak・Backstage の source Secret 投入を追加

cd ~/platform-infra
git add k3d/Makefile
git commit -m "fix: bootstrap 構造改善 - root-app.yaml 一本化・bootstrap-apps ターゲット追加"
git push origin main
```

### sync-wave の修正

```bash
# wave 未設定だった Application に sync-wave を追加
# kyverno.yaml: wave 2
# kyverno-policies.yaml: wave 4
# loki/alloy/tempo/vpa: wave 3
# gateway/config/*.yaml: wave 4（GatewayClass・Gateway・HTTPRoute・ReferenceGrant）

# CRD 依存順序の修正
# cnpg.yaml: wave 2→4（PodMonitor CRD は kube-prometheus-stack wave 3 の後）
# trivy-operator.yaml: wave 3→4（ServiceMonitor CRD 依存）
# argo-rollouts.yaml: wave 3→4（ServiceMonitor CRD 依存）
# backstage-db.yaml: wave 3→5（CNPG operator wave 4 の後）
# keycloak-db.yaml: wave 3→5（同上）

# ExternalSecret の wave 修正
# backstage/app-config/external-secret.yaml: wave -1→5（ESO wave 1・ClusterSecretStore wave 4 の後）
# backstage/db-config/external-secret.yaml: wave 2→5（同上）
# backstage/db-config/cluster.yaml: wave 3→5（CNPG operator の後）
# keycloak/db-config/external-secret.yaml: wave 2→5（同上）
# keycloak/db-config/cnpg-cluster.yaml: wave 3→5（同上）

# CreateNamespace=true の追加漏れ修正
# argo-rollouts/cert-manager-config/cilium/crossplane-config/
# envoy-gateway-crds/external-secrets-config/keda/trivy-operator

git add platform/applications/
git add platform/backstage/
git add platform/keycloak/
git commit -m "fix: sync-wave 未設定・誤設定・CreateNamespace 漏れを一括修正"
git push origin main
```

### sample-app の HTTPRoute・ReferenceGrant をアプリ側に移動

```bash
mkdir -p ~/platform-gitops/apps/sample-frontend/manifests

mv ~/platform-gitops/platform/gateway/config/httproute-backend.yaml \
   ~/platform-gitops/apps/sample-backend/manifests/httproute.yaml
mv ~/platform-gitops/platform/gateway/config/httproute-frontend.yaml \
   ~/platform-gitops/apps/sample-frontend/manifests/httproute.yaml
mv ~/platform-gitops/platform/gateway/config/reference-grant.yaml \
   ~/platform-gitops/apps/sample-backend/manifests/reference-grant.yaml

# sample-frontend.yaml に manifests/ を source として追加

git add apps/sample-backend/manifests/
git add apps/sample-frontend/
git add platform/gateway/config/
git commit -m "refactor: sample-app の HTTPRoute・ReferenceGrant をアプリ側に移動"
git push origin main
```

### cnpg.yaml のアノテーション重複を解消

```bash
# metadata.annotations が2重に定義されていたため削除
# wave 2（古い値）を削除し wave 4 に統一

git add platform/applications/cnpg.yaml
git commit -m "fix: cnpg Application の sync-wave アノテーション重複を解消（wave 4 に統一）"
git push origin main
```
