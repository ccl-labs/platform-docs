# Bootstrap 設計書

---

## 1. 概要

プラットフォームの bootstrap は、コンポーネント間の依存関係により単純な一斉 apply では正しく起動しない。単一の ArgoCD App-of-Apps と `sync-wave` アノテーションによる起動順序制御、およびカスタムヘルスチェックによる wave 進行の保証を組み合わせて、再現性のある bootstrap を実現する。Makefile は ArgoCD が管理できない操作（クラスタ作成・CNI インストール・CoreDNS 設定・ArgoCD 自身のインストール）のみを担う。

---

## 2. ルート App 構成

### 2.1 Application 定義

`platform-gitops/platform/group-roots/root.yaml` に単一の App-of-Apps を定義する。

```yaml
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
    repoURL: https://github.com/okccl/platform-gitops.git
    targetRevision: HEAD
    path: platform/applications
    directory:
      recurse: true
      exclude: 'vcluster-dev.yaml'
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 2.2 ディレクトリ構造

```
platform/applications/   # 全 Application を直下にフラット配置
```

---

## 3. Wave 設計

グループ間の wave 番号は 10 刻みで分離し、グループをまたぐ依存関係が wave 制御で確実に解消されるようにしている。

### 3.1 Wave 0–4（ネットワーク基盤）

| Wave | Application | 依存根拠 |
|---|---|---|
| 0 | storage | 依存なし（local-path StorageClass） |
| 1 | cert-manager | 依存なし |
| 1 | cilium | 依存なし |
| 1 | envoy-gateway-crds | 依存なし（CRD 先行インストール） |
| 2 | envoy-gateway | envoy-gateway-crds（CRD 依存） |
| 3 | cert-manager-config | cert-manager webhook（CA Issuer・Certificate） |
| 4 | gateway-config | envoy-gateway（Gateway/EnvoyProxy）+ cert-manager-config（TLS 証明書） |

### 3.2 Wave 10–15（認証・シークレット基盤）

| Wave | Application | 依存根拠 |
|---|---|---|
| 10 | external-secrets | 依存なし |
| 10 | kube-prometheus-stack | 依存なし |
| 10 | kyverno | 依存なし |
| 11 | cnpg | 依存なし（webhook は Wave 10 完了後に有効化） |
| 11 | platform-secrets-sources | external-secrets（ESO CRD + ClusterSecretStore） |
| 12 | external-secrets-config | platform-secrets-sources（source secrets 展開済み） |
| 12 | kyverno-policies | kyverno webhook |
| 13 | keycloak-db | cnpg webhook + platform-secrets-sources（DB パスワード Secret） |
| 14 | keycloak | keycloak-db（DB 接続先） |
| 15 | keycloak-config-cli | keycloak（設定投入先） |
| 15 | keycloak-routes | keycloak（HTTPRoute のバックエンド） |

### 3.3 Wave 20–22（その他プラットフォームコンポーネント）

| Wave | Application | 依存根拠 |
|---|---|---|
| 20 | alloy | 依存なし（loki/tempo は起動後に接続） |
| 20 | argo-rollouts | 依存なし |
| 20 | argocd | 依存なし（self-manage） |
| 20 | crossplane | 依存なし |
| 20 | gateway-routes | 依存なし（バックエンドは前グループで起動済み） |
| 20 | goldilocks | 依存なし |
| 20 | keda | 依存なし |
| 20 | loki | 依存なし |
| 20 | tempo | 依存なし |
| 20 | trivy-operator | 依存なし |
| 20 | user-apps | 依存なし（apps-gitops 別リポジトリ） |
| 20 | user-apps-project | 依存なし |
| 20 | vpa | 依存なし |
| 21 | backstage-db | cnpg webhook（Wave 11 完了で保証済み） |
| 21 | crossplane-config | crossplane |
| 21 | grafana-dashboards | kube-prometheus-stack（ConfigMap 自動検出） |
| 21 | platform-alerts | kube-prometheus-stack（PrometheusRule） |
| 21 | user-apps-infra | user-apps-project |
| 22 | backstage | backstage-db（DB 接続先） |

---

## 4. カスタムヘルスチェック

`platform/argocd/values.yaml` に定義。wave の進行条件として ArgoCD が使用する。

| リソース種別 | 保証する内容 |
|---|---|
| `admissionregistration.k8s.io/ValidatingWebhookConfiguration` | webhook CA bundle の注入完了。cert-manager・CNPG・ESO・Kyverno の webhook が呼び出し可能な状態になるまで次の wave に進まない |
| `external-secrets.io/ClusterSecretStore` | ESO が外部プロバイダー（kubernetes-store）へ接続できているか |
| `external-secrets.io/ExternalSecret` | ESO が Secret を正常に生成できているか |

---

## 5. Bootstrap フロー

`make bootstrap` が実行するステップ。

| ステップ | Make ターゲット | 内容 |
|---|---|---|
| 1 | `cluster-create` | k3d クラスタ作成 |
| 2 | `install-cilium` | Cilium インストール（ArgoCD 起動前に CNI が必要） |
| 3 | `fix-coredns` | CoreDNS 設定（後述） |
| 4 | `bootstrap-argocd` | cert-manager/argocd namespace 作成・CA 投入・ArgoCD インストール・認証情報投入 |
| 5 | `bootstrap-sync` | ルート App apply → 完全起動まで待機 |

`bootstrap-sync` の詳細：

```
1. ArgoCD port-forward 起動 + localhost:8080 でログイン
2. root App を apply（kubectl apply -f root.yaml）
3. Envoy Gateway Pod が Ready になるまで待機
   （argocd.platform.local アクセスに必要）
4. ArgoCD ログインを argocd.platform.local 経由に切り替え
5. argocd app wait root（全 wave 完了まで）
```

### 5.1 CoreDNS 設定

`make fix-coredns`（ステップ 3）で以下の 2 つを CoreDNS ConfigMap に書き込む。bootstrap 前の時点で実行するため、Envoy Gateway SVC が存在しない状態でも事前に書き込み可能。

**NodeHosts — `host.k3d.internal` の登録**

MinIO コンテナ（`minio-external`）への疎通のため、k3d ネットワークのゲートウェイ IP を `host.k3d.internal` として登録する。

**Corefile — `*.platform.local` のリライトルール**

クラスタ内 Pod が `keycloak.platform.local` などを解決できるよう、Envoy Gateway の Service 名へリライトする。

```
rewrite name keycloak.platform.local  envoy-eg.envoy-gateway-system.svc.cluster.local
rewrite name argocd.platform.local    envoy-eg.envoy-gateway-system.svc.cluster.local
rewrite name backstage.platform.local envoy-eg.envoy-gateway-system.svc.cluster.local
```

Service 名は `platform/gateway/config/envoy-proxy-config.yaml`（EnvoyProxy リソース）で `envoy-eg` に固定しているため、bootstrap 前の時点で確定した名前を記述できる。
