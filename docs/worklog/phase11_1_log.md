# Phase 11-1 作業ログ（Gateway API / Envoy Gateway 移行）


## 作業概要
ingress-nginx（2026年3月メンテナンス停止）をEnvoy Gateway v1.7.2に置き換え。
Gateway API（GatewayClass / Gateway / HTTPRoute）によるルーティングに移行した。

---

## 1. 事前調査・バージョン確認

```bash
# Envoy Gateway最新安定版確認（web検索）
# → v1.7.2 が最新安定版であることを確認
```

---

## 2. platform-gitopsディレクトリ構成作成

```bash
mkdir -p ~/platform-gitops/platform/gateway/config
mkdir -p ~/platform-gitops/platform/gateway/envoy-gateway-crds
mkdir -p ~/platform-gitops/platform/gateway/envoy-gateway
```

---

## 3. Rendered Manifests Patternでチャートをレンダリング

※ 当初はArgoCDのOCI直接参照（選択肢A）を試みたが失敗。Rendered Manifests Pattern（選択肢B）に切り替えた。
（詳細はトラブルシューティングセクション参照）

```bash
# DockerHubにログイン
helm registry login registry-1.docker.io \
  --username <DockerHubユーザー名> \
  --password <AccessToken>

# CRDをレンダリング
helm template envoy-gateway-crds \
  oci://docker.io/envoyproxy/gateway-crds-helm \
  --version v1.7.2 \
  --set crds.gatewayAPI.enabled=true \
  --set crds.gatewayAPI.channel=standard \
  --set crds.envoyGateway.enabled=true \
  > ~/platform-gitops/platform/gateway/envoy-gateway-crds/crds.yaml

# 本体をレンダリング
helm template envoy-gateway \
  oci://docker.io/envoyproxy/gateway-helm \
  --version v1.7.2 \
  --namespace envoy-gateway-system \
  --set deployment.envoyGateway.resources.requests.cpu=100m \
  --set deployment.envoyGateway.resources.requests.memory=256Mi \
  --set deployment.envoyGateway.resources.limits.cpu=500m \
  --set deployment.envoyGateway.resources.limits.memory=512Mi \
  > ~/platform-gitops/platform/gateway/envoy-gateway/manifests.yaml
```

---

## 4. ArgoCD Application作成

### envoy-gateway-crds-app.yaml（wave 1）
```yaml
# platform/gateway/envoy-gateway-crds-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: envoy-gateway-crds
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
    path: platform/gateway/envoy-gateway-crds
  destination:
    server: https://kubernetes.default.svc
    namespace: envoy-gateway-system
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
```

### envoy-gateway-app.yaml（wave 2）
```yaml
# platform/gateway/envoy-gateway-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: envoy-gateway
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:okccl/platform-gitops.git
    targetRevision: HEAD
    path: platform/gateway/envoy-gateway
  destination:
    server: https://kubernetes.default.svc
    namespace: envoy-gateway-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
```

### gateway-config-app.yaml（wave 4）
```yaml
# platform/gateway/gateway-config-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gateway-config
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "4"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:okccl/platform-gitops.git
    targetRevision: HEAD
    path: platform/gateway/config
  destination:
    server: https://kubernetes.default.svc
    namespace: envoy-gateway-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
  ignoreDifferences:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      jqPathExpressions:
        - .spec.parentRefs[]?.group
        - .spec.parentRefs[]?.kind
        - .spec.rules[]?.backendRefs[]?.group
        - .spec.rules[]?.backendRefs[]?.kind
        - .spec.rules[]?.backendRefs[]?.weight
```

---

## 5. root.yaml excludeパターン更新

```bash
# クラスタ上のroot Appにパッチ（Gitと同期するまでの暫定対処）
kubectl patch application root -n argocd --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/source/directory/exclude",
    "value": "{policy/policies/*,ingress/config/*,secrets/config/*,gateway/config/*,gateway/envoy-gateway-crds/*,gateway/envoy-gateway/*}"
  }
]'
```

bootstrap/root.yaml も同様に更新：
```yaml
exclude: "{policy/policies/*,ingress/config/*,secrets/config/*,gateway/config/*,gateway/envoy-gateway-crds/*,gateway/envoy-gateway/*}"
```

---

## 6. Kyvernoポリシー除外設定

```bash
# platform/policy/platform-namespaces.yamlに追記
cat >> ~/platform-gitops/platform/policy/platform-namespaces.yaml << 'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: envoy-gateway-system
  labels:
    platform-managed: "true"
EOF
```

---

## 7. Gateway設定リソース作成（platform/gateway/config/）

```yaml
# gateway-class.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
  namespace: envoy-gateway-system
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

```yaml
# httproute-backend.yaml / httproute-frontend.yaml / httproute-argocd.yaml
# httproute-grafana.yaml / httproute-goldilocks.yaml
# （各サービスのHTTPRoute。backendRefsでクロスNamespace参照）
```

```yaml
# reference-grant.yaml / reference-grant-argocd.yaml / reference-grant-monitoring.yaml
# （クロスNamespaceのService参照を許可するReferenceGrant）
```

---

## 8. ingress-nginx廃止・各サービスのIngress無効化

```bash
# ingress-nginx Application削除
git rm ~/platform-gitops/platform/ingress/application.yaml

# 各values.yamlのingress.enabled: false に変更
# - apps/sample-backend/values.yaml
# - apps/sample-frontend/values.yaml
# - platform/monitoring/values.yaml（grafana.ingress.enabled: false）
# - platform/goldilocks/values.yaml（dashboard.ingress.enabled: false）
# - platform/argocd/values.yaml（ingress.enabled: false）
```

---

## 9. 動作確認

```bash
# Gateway状態確認
kubectl get gateway -n envoy-gateway-system
# NAME   CLASS           ADDRESS      PROGRAMMED   AGE
# eg     envoy-gateway   172.19.0.2   True         ...

# 全HTTPRoute確認
kubectl get httproute -n envoy-gateway-system

# 各サービスへの疎通確認
curl -H "Host: sample-backend.localhost" http://172.19.0.2/health
# {"status":"ok"}

curl -H "Host: sample-frontend.localhost" http://172.19.0.2/ -I
# HTTP/1.1 200 OK

curl -H "Host: grafana.localhost" http://172.19.0.2/ -I
# HTTP/1.1 302 Found

curl -H "Host: goldilocks.localhost" http://172.19.0.2/ -I
# HTTP/1.1 301 Moved Permanently

curl -H "Host: argocd.localhost" http://172.19.0.2/ -I
# HTTP/1.1 200 OK

# Ingress完全削除確認
kubectl get ingress -A
# No resources found
```

---

## 10. bootstrap Makefile更新

ingress-nginxの廃止に伴い、`bootstrap-sync` の待機対象を変更した。

変更前（ingress-nginx待機）：
```makefile
@until kubectl get deployment ingress-nginx-controller -n ingress-nginx 2>/dev/null; do echo "ingress-nginx待機..."; sleep 5; done
kubectl wait --for=condition=available deployment/ingress-nginx-controller \
    -n ingress-nginx --timeout=600s
```

変更後（envoy-gateway待機）：
```makefile
@until kubectl get deployment envoy-gateway -n envoy-gateway-system 2>/dev/null; do echo "envoy-gateway待機..."; sleep 5; done
kubectl wait --for=condition=available deployment/envoy-gateway \
    -n envoy-gateway-system --timeout=600s
```

あわせて完了メッセージのURLを `https://argocd.localhost` → `http://argocd.localhost` に変更（TLS未設定のため）。

```bash
cd ~/platform-infra
git add k3d/Makefile
git commit -m "fix(bootstrap): replace ingress-nginx wait with envoy-gateway wait"
git push origin main
```

### DR実施後の追加修正

Phase 11-1後に初めてDRを実施したところ、`make bootstrap`完了時点でEnvoy ProxyのデータプレーンPod（`envoy-envoy-gateway-system-eg-*`）がまだ`ContainerCreating`の状態でArgoCD GUIにアクセスできないことが判明。コントロールプレーン待ちだけでは不十分なため、データプレーンPodのReady待ちを追加した。

（変更後のmakefileスニペット）

### DR結果（Phase 11-1後）

| 指標 | Phase 10.5 | Phase 11-1後 |
|---|---|---|
| RTO① ArgoCD GUIアクセス可能 | 7分37秒 | 6分28秒 |
| RTO② 全App Synced | 15分24秒 | 約14分 |

ingress-nginx廃止によりコンポーネントが減り、RTOが改善された。

---

## 11. platform-infra README更新

Phase 11-1の内容を反映してREADMEを更新した。

- Phase 3にingress-nginx廃止の注記を追加
- Phase 1の設計メモをEnvoy Gatewayに更新（Traefik無効化の理由）
- Phase 11を「Cloud Expansion」から「Hardening & Exploration」に修正
- Phase 11-1の内容（Rendered Manifests Pattern採用理由・移行時のポート競合・更新手順）を追記

```bash
cd ~/platform-infra
git add README.md
git commit -m "docs: update README for Phase 11-1 Gateway API migration"
git push origin main
```

---

## トラブルシューティング

### 問題1：ArgoCDのOCI直接参照が401 Unauthorizedで失敗

**原因：** ArgoCDはApplicationの`repoURL`（`docker.io`）を内部で`registry-1.docker.io`に解決するが、credential照合はURLの完全一致で行われる。DockerHubの二重ドメイン構造に起因する既知の挙動。両URLにcredentialを登録しても解決しなかった。

**対処：** Rendered Manifests Pattern（`helm template`でローカルレンダリングしたYAMLをGitに格納）に切り替えた。

### 問題2：root appがgateway/config/*をwave 0で直接拾う

**原因：** `bootstrap/root.yaml`のexcludeパターンにgateway配下のパスが含まれていなかった。またクラスタ上のroot Appオブジェクトが古いexcludeパターンのまま残っていた。

**対処：** `kubectl patch`でクラスタ上のroot Appを直接更新し、`bootstrap/root.yaml`も修正してコミット。レンダリング済みマニフェストのパス（`gateway/envoy-gateway-crds/*`、`gateway/envoy-gateway/*`）もexcludeに追加した。

### 問題3：Kyvernoポリシーに引っかかりJob失敗

**原因：** `CreateNamespace=true`でArgoCDがKyvernoラベルなしでnamespaceを先に作成してしまうため、Jobがポリシー違反としてブロックされた。

**対処：** `CreateNamespace=true`をApp定義から削除し、`platform-namespaces.yaml`（root appのwave 0管理）に`envoy-gateway-system`を追加。DR再構築時も自動的にラベル付きで作成される構成に統一した。

### 問題4：ingress-nginxとのポート80競合

**原因：** k3dのServiceLBはポートを占有する仕組みのため、ingress-nginxが80番を使用している間はEnvoy GatewayのLoadBalancerがpendingのままとなった。

**対処：** 移行期間中はEnvoy Gatewayを8080番で起動して動作確認を行い、ingress-nginx削除後に80番に変更した。

### 問題5：HTTPRouteのArgoCD Diff

**原因：** KubernetesがHTTPRouteの`parentRefs`/`backendRefs`にデフォルト値（`group`/`kind`/`weight`）を自動付与するため、Gitのマニフェストと差分が出続ける。

**対処：** gateway-config AppのignoredDifferencesにjqPathExpressionsで該当フィールドを追加した。
