# Phase 4 作業ログ

## Step 1: kube-prometheus-stack の導入

### ファイル作成

```
platform-gitops/platform/monitoring/application.yaml
platform-gitops/platform/monitoring/values.yaml
```

### Git操作

```bash
cd ~/platform-gitops
git add platform/monitoring/
git commit -m "feat: add kube-prometheus-stack (Phase 4)"
git push
```

### ArgoCD操作

```bash
# root appを強制syncしてApplicationを認識させる
argocd app sync root --force

# syncと起動完了を待機
argocd app sync kube-prometheus-stack --async
argocd app wait kube-prometheus-stack --timeout 300

# ステータス確認
argocd app get kube-prometheus-stack | grep -E "Sync Status|Health Status"
```

### トラブルシューティング

**問題:** application.yamlの `valuesFile` 指定ではvalues.yamlが読み込まれなかった

**原因:** HelmリポジトリチャートとGitリポジトリのvalues.yamlを分離する場合、ArgoCDのmulti-source機能を使う必要がある

**対処:** application.yamlを `sources:` (複数形) のmulti-source構成に変更し、`$values` 参照でvalues.yamlを指定した

```yaml
# 修正後の構成
sources:
  - repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 83.6.0
    helm:
      valueFiles:
        - $values/platform/monitoring/values.yaml
  - repoURL: git@github.com:okccl/platform-gitops.git
    targetRevision: main
    ref: values
```

### 動作確認

```bash
# Pod起動確認
kubectl get pods -n monitoring

# Ingress確認
kubectl get ingress -n monitoring
```

ブラウザで `https://grafana.localhost` にアクセスしてGrafanaのログイン画面が表示されることを確認。
Dashboards → Kubernetes / Compute Resources / Cluster でメトリクスが表示されることを確認。

---

## Step 2: Loki + Grafana Alloy の導入

### ファイル作成

```
platform-gitops/platform/logging/loki-application.yaml
platform-gitops/platform/logging/loki-values.yaml
platform-gitops/platform/logging/alloy-application.yaml
platform-gitops/platform/logging/alloy-values.yaml
```

### Helmチャートバージョン確認

```bash
# grafana repoが未追加の場合は追加
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Alloyのバージョン確認
helm search repo grafana/alloy --versions | head -5
```

### Git操作

```bash
cd ~/platform-gitops
git add platform/logging/
git commit -m "feat: add Loki and Alloy (Phase 4 Step 2)"
git push
```

### ArgoCD操作

```bash
argocd app sync root --force
argocd app list
```

### トラブルシューティング

**問題:** AlloyのHelmチャートバージョン `1.10.1` が見つからないエラー

**原因:** AlloyのHelmチャートバージョン体系はApp Versionと異なる。実際の最新チャートバージョンは `1.7.0`

**対処:** `helm search repo grafana/alloy --versions` で確認後、`alloy-application.yaml` のtargetRevisionを `1.7.0` に修正

```bash
cd ~/platform-gitops
git add platform/logging/alloy-application.yaml
git commit -m "fix: correct alloy helm chart version to 1.7.0"
git push
```

**問題:** GrafanaにLokiデータソースが自動登録されなかった

**原因:** Lokiのvalues.yamlに書いた `grafana.datasources` はLokiチャート自身には効果がない。Grafana側（kube-prometheus-stack）のvalues.yamlに設定する必要がある

**対処:** `platform/monitoring/values.yaml` に `additionalDataSources` としてLokiを追加

```bash
cd ~/platform-gitops
git add platform/monitoring/values.yaml platform/logging/loki-values.yaml
git commit -m "fix: add Loki datasource to Grafana"
git push
```

### 動作確認

```bash
kubectl get pods -n logging
```

GrafanaのExplore → Loki → `{namespace="argocd"}` クエリでログが表示されることを確認。

---

## Step 3: Tempo の導入

### ファイル作成・編集

```
platform-gitops/platform/tracing/tempo-application.yaml  （新規）
platform-gitops/platform/tracing/tempo-values.yaml        （新規）
platform-gitops/platform/monitoring/values.yaml           （編集: Tempoデータソース追加）
platform-gitops/platform/logging/alloy-values.yaml        （編集: OTLPレシーバー追加）
```

### Helmチャートバージョン確認

```bash
helm search repo grafana/tempo --versions | head -5
```

### Git操作

```bash
cd ~/platform-gitops
git add platform/tracing/ platform/monitoring/values.yaml platform/logging/alloy-values.yaml
git commit -m "feat: add Tempo and OTLP tracing pipeline (Phase 4 Step 3)"
git push
```

### ArgoCD操作

```bash
argocd app sync root --force
argocd app list
```

### トラブルシューティング

**問題:** GrafanaのTempoデータソースのTestが `dial tcp: i/o timeout` で失敗

**原因:** TempoのクエリポートはLokiと同じ `3100` ではなく `3200`

**確認コマンド:**
```bash
kubectl get svc -n tracing
```

**対処:** `monitoring/values.yaml` のTempoデータソースURLのポートを `3200` に修正

```bash
cd ~/platform-gitops
git add platform/monitoring/values.yaml
git commit -m "fix: correct Tempo service port to 3200"
git push
```

### 動作確認

```bash
kubectl get pods -n tracing
```

GrafanaのExplore → Tempo → Run query でエラーなし（0件）を確認。

---

## Step 4: データソース統合・動作確認

### ファイル編集

```
platform-gitops/platform/monitoring/values.yaml  （編集: Loki Derived Fields追加）
```

### GrafanaのTempoデータソースUID確認

GrafanaのUI（Connections → Data sources → Tempo）のURLから確認:
```
https://grafana.localhost/connections/datasources/edit/P214B5B846CF3925F
                                                        ^^^^^^^^^^^^^^^^
                                                        このIDをDerived Fieldsに使用
```

### Git操作

```bash
cd ~/platform-gitops
git add platform/monitoring/values.yaml
git commit -m "feat: add Loki to Tempo derived fields for trace correlation"
git push
```

### ArgoCD操作

```bash
argocd app sync root --force
argocd app list
```

### 動作確認

GrafanaのConnections → Data sources → Loki → Derived fieldsセクションに以下が設定されていることを確認:

| 項目 | 値 |
|---|---|
| Name | TraceID |
| Type | Regex in log line |
| Regex | `traceID=(\w+)` |
| Internal link | Tempo |
