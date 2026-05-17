# Phase12-A16 作業ログ

| # | 作業内容 |
|---|---|
| 1 | CoreDNS rewrite の事前確定（Envoy Gateway SVC 名固定） |
| 2 | DOCS_RULE.md 修正（設計書の保存場所ルール更新） |
| 3 | bootstrap 設計書の新規作成 |
| 4 | bootstrap 単一ルート集約（wave 番号付け替え・root.yaml 作成・旧3ルート削除・Makefile 更新） |

---

## 1. CoreDNS rewrite の事前確定

### 背景

bootstrap 単一ルート集約の前提として、`*.platform.local` の CoreDNS rewrite ルールを bootstrap 前の時点で確定できるようにする必要があった。

従来の `bootstrap-sync` では、Envoy Gateway が Gateway リソースの apply 後に動的生成する Service 名（`envoy-envoy-gateway-system-eg-<ハッシュ>`）を `kubectl get svc -l owning-gateway-name=eg` で取得してから CoreDNS を更新していた。この動的取得処理は ArgoCD の wave 自動進行に割り込めないため、単一ルート化の障害になっていた。

Envoy Gateway v1.1 以降で利用可能な `EnvoyProxy` リソースの `spec.provider.kubernetes.envoyService.name` を使い、Service 名を `envoy-eg` に固定することで解決した。

### 実施内容

`platform/gateway/config/envoy-proxy-config.yaml` を新規作成し、`gateway.yaml` に `infrastructure.parametersRef` を追加した。

```yaml
# 新規: envoy-proxy-config.yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: eg-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        name: envoy-eg
```

```yaml
# gateway.yaml に追加
spec:
  infrastructure:
    parametersRef:
      group: gateway.envoyproxy.io
      kind: EnvoyProxy
      name: eg-proxy-config
```

`k3d/Makefile` の `fix-coredns` ターゲットに `*.platform.local` rewrite ルールを追加し、`bootstrap-sync` の動的 CoreDNS パッチ処理を削除した。

```makefile
cf=d['data']['Corefile']; \
rule='    rewrite name keycloak.platform.local envoy-eg.envoy-gateway-system.svc.cluster.local\n...'; \
d['data']['Corefile']=cf.replace('ready\n', 'ready\n'+rule) if 'rewrite name keycloak.platform.local' not in cf else cf; \
```

push 後に ArgoCD が gateway-config を自動 sync し、旧 SVC（ハッシュ付き）が削除されて `envoy-eg` が作成された。`make fix-coredns` を手動実行して現クラスタの CoreDNS を更新し、`argocd.platform.local` / `keycloak.platform.local` の疎通を確認した。

---

## 2. DOCS_RULE.md 修正

設計書の保存場所ルールが実態（`platform-docs/docs/design/`）と乖離していたため修正した。

- 旧: 設計ドキュメントは `~/internal/claude/tmp_<テーマ>.md`
- 新: 正式設計書は `platform-docs/docs/design/<テーマ>-design.md`。設計メモ（tmp ファイル）のセクションは削除。

章立て書式（`## 1.` / `### 1.1`）と、ステータス行は作成中のときのみ記載するルールを追記した。

---

## 3. bootstrap 設計書の新規作成

改修後の bootstrap 構成を `platform-docs/docs/design/bootstrap-design.md` として新規作成した。

記載内容：

- **ルート App 構成**：単一 Application の YAML 定義・ディレクトリ構造
- **Wave 設計**：全 37 Application の wave 番号一覧（0–4 / 10–15 / 20–22）
- **カスタムヘルスチェック**：定義一覧と各チェックが保証する内容
- **Bootstrap フロー**：`make bootstrap` のステップ表と `bootstrap-sync` の詳細
- **CoreDNS 設計**：`*.platform.local` 事前解決の仕組み

---

## 4. bootstrap 単一ルート集約

### wave 番号付け替え

グループ間を 10 刻みで分離し、root-2-auth（旧 1–6 → 新 10–15）・root-3-others（旧 1–3 → 新 20–22）を更新した。`user-apps.yaml` / `user-apps-project.yaml` には wave アノテーション自体がなかったため wave 20 を追加した。

### 単一ルート root.yaml 作成

`platform/group-roots/root.yaml` を新規作成。`platform/applications` を `recurse: true` で参照し、`vcluster-dev.yaml` のみ exclude する構成。

```yaml
directory:
  recurse: true
  exclude: 'root-3-others/vcluster-dev.yaml'
```

### 移行操作

```bash
kubectl apply -f ~/platform-gitops/platform/group-roots/root.yaml
argocd app delete root-1-gateway --cascade=false
argocd app delete root-2-auth    --cascade=false
argocd app delete root-3-others  --cascade=false
```

`--cascade=false` により子 App（cert-manager・keycloak 等）を残したまま旧3ルートを削除した。root App が自動 sync して Synced + Healthy になったことを確認後、旧3ルートの YAML ファイルを削除した。

### Makefile 更新

`bootstrap-sync` を3ルートの順次 apply から単一ルートの apply + wait に置き換えた。

```makefile
kubectl apply -f ~/platform-gitops/platform/group-roots/root.yaml
argocd app sync root --server-side || true
# ... Envoy Gateway 待機・ログイン切替 ...
argocd app wait root --sync --health --timeout 1800
```

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-gitops/platform/gateway/config/envoy-proxy-config.yaml` | 新規作成：EnvoyProxy リソース（SVC 名 `envoy-eg` に固定） |
| `platform-gitops/platform/gateway/config/gateway.yaml` | `infrastructure.parametersRef` 追加 |
| `platform-gitops/platform/applications/root-2-auth/*.yaml` | sync-wave: 1–6 → 10–15 |
| `platform-gitops/platform/applications/root-3-others/*.yaml` | sync-wave: 1–3 → 20–22、user-apps 2ファイルに wave 20 追加 |
| `platform-gitops/platform/group-roots/root.yaml` | 新規作成：単一ルート App-of-Apps |
| `platform-gitops/platform/group-roots/root-{1,2,3}-*.yaml` | 削除（単一ルート移行完了） |
| `platform-infra/k3d/Makefile` | fix-coredns に rewrite 追加、bootstrap-sync を単一ルート対応に更新 |
