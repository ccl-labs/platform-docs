# Phase12-A14 作業ログ

| # | 作業内容 |
|---|---|
| 1 | GitHub ccl-labs → okccl 移行（リポジトリ transfer・コード置換・GitHub App 移行・argocd-repo-creds 更新） |
| 2 | Makefile cluster-start に Cilium 疎通確認を追加 |
| 3 | トラブルシュート：Cilium 起動タイミング問題による各コンポーネント障害 |
| 4 | トラブルシュート：make cluster-restart 後の CiliumNode IP ズレによるクロスノード通信断 |

---

## 1. GitHub ccl-labs → okccl 移行

### 背景

Findy 等の転職サイトの GitHub 連携では Organization リポジトリが参照されないため、ポートフォリオの可視性向上を目的に全リポジトリを個人アカウントへ transfer した。ccl-labs Org を作った当初の目的は「開発者申請→PE承認」の権限分離検証だったが、GitHub の検証にすぎずポートフォリオ的に意味が薄いため廃止した。

### 実施手順

**GitHub UI 作業**

以下 8 リポジトリを `ccl-labs` → `okccl` へ transfer：
platform-infra / platform-gitops / platform-charts / platform-docs / sample-backend / sample-frontend / apps-gitops / backstage

GitHub App（ArgoCD 用・Backstage 用）は "Transfer ownership" で `okccl` へ移管し、対象リポジトリに再インストール。Backstage OAuth App は元から `okccl` 個人アカウントに紐づいていたため変更不要。

**コード一括置換**

全リポジトリで `ccl-labs` → `okccl` を sed で一括置換・コミット・push。

```bash
find /home/ccl/<repo> -type f \( -name "*.yaml" -o -name "*.md" \) \
  | xargs grep -l "ccl-labs" | xargs sed -i 's/ccl-labs/okccl/g'
```

対象箇所：ArgoCD Application の repoURL / Backstage catalog target URL / ghcr.io イメージ参照 / catalog-info.yaml の `github.com/project-slug` / GitHub Actions workflows の owner・image 参照 / Scaffolder テンプレートの owner

ディレクトリ名に `${{ values.appName }}` を含む gitops-skeleton は `find` がスキップするため、直接 `sed -i` で対応した。

**git remote URL 更新**

```bash
git remote set-url origin git@github.com:okccl/<repo>.git
```

**argocd-repo-creds 更新**

GitHub App transfer 後に Installation ID が変更（130426273 → 131473848）。さらに Secret の `name` と `url` も `ccl-labs` のままだったため、SOPS ファイルを 3 箇所修正した。

```yaml
name: argocd-repo-creds-okccl        # ccl-labs → okccl
url: https://github.com/okccl/        # ccl-labs → okccl
githubAppInstallationID: "131473848"  # 130426273 → 131473848
```

---

## 2. Makefile cluster-start に Cilium 疎通確認を追加

### 背景

`make cluster-start` で Cilium DaemonSet を再起動しても、eBPF マップの構築完了前に他の Pod が再起動すると ClusterIP 通信が断絶する問題が繰り返し発生した。`cilium status --wait` で Cilium が完全に疎通確認できるまでブロックする処理を追加した。

`cilium` CLI は未インストールのため、Cilium Pod 内のバイナリを `kubectl exec` 経由で呼び出す形で実装した。

```makefile
@echo ">>> Cilium の全ノード疎通を確認中..."
kubectl exec -n kube-system ds/cilium -- cilium status --wait
```

---

## 3. トラブルシュート：Cilium 起動タイミング問題による各コンポーネント障害

### 問題

`make cluster-start` 後、以下のコンポーネントが障害状態になった。

| コンポーネント | 症状 |
|---|---|
| Kyverno admission controller | webhook タイムアウト（`context deadline exceeded`） |
| ArgoCD repo-server | CrashLoopBackOff（liveness probe タイムアウト） |
| ArgoCD app-controller | Redis DNS 解決失敗（`lookup argocd-redis: i/o timeout`） |

### 原因

Cilium DaemonSet の再起動中（eBPF マップ構築中）に各 Pod が再起動し、Cilium CNI が正常にネットワークインターフェースを設定できなかった状態で固定された。その後 Cilium が正常化しても、既存 Pod のネットワーク設定は修正されない。

### 対処

各 Pod を再起動して、正常な Cilium eBPF マップ上で新しいネットワークインターフェースを取得させた。

```bash
kubectl rollout restart deployment/kyverno-admission-controller -n kyverno
kubectl rollout restart deployment/argocd-repo-server -n argocd
kubectl rollout restart statefulset/argocd-application-controller -n argocd
```

**本対応（将来方針）**: `make cluster-start` に追加した `cilium status --wait` により、次回以降は Cilium が完全に準備できてから他の Pod が再起動するため、この問題は発生しなくなる想定。

---

## 4. トラブルシュート：make cluster-restart 後の CiliumNode IP ズレによるクロスノード通信断

### 問題

`make cluster-restart`（stop → start）後、Keycloak が PostgreSQL に接続できず CrashLoopBackOff が継続。調査の結果、クロスノードの Pod-to-Pod 通信が完全に断絶していることが判明した（同一ノード内の通信は正常）。

### 原因

`k3d cluster stop` → `start` で Docker がコンテナに異なる IP を再割り当てしたにもかかわらず、`CiliumNode` カスタムリソースが古い IP のまま残留した。Cilium の VXLAN トンネルが誤ったホスト IP に向くため、別ノードの Pod への通信がすべて失敗した。

```
# 再起動前後で agent-0 と agent-1 の IP が入れ替わった
CiliumNode の認識: agent-0=172.19.0.3 / agent-1=172.19.0.5
Docker の実際:     agent-0=172.19.0.5 / agent-1=172.19.0.3
```

`kubectl get ciliumnodes` と `kubectl get nodes -o wide` の InternalIP を比較して特定した。

### 対処

CiliumNode リソースを全削除し、Cilium DaemonSet を再起動して正しい IP で再登録させた。

```bash
kubectl delete ciliumnodes --all
kubectl rollout restart daemonset/cilium -n kube-system
kubectl rollout status daemonset/cilium -n kube-system --timeout=120s
```

**本対応（将来方針）**: `make cluster-start` に `kubectl delete ciliumnodes --all` を追加することで恒久対策となる可能性がある。ただし副作用の有無を確認してから採用する（次セッションで検討）。

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-infra/Makefile` | cluster-start に `cilium status --wait` 追加 |
| `platform-infra/catalog-info.yaml` | `github.com/project-slug` を okccl に変更 |
| `platform-gitops/` 全体（50ファイル） | repoURL・catalog target・テンプレート等の ccl-labs → okccl 置換 |
| `platform-gitops/secrets/encrypted/argocd-repo-creds.yaml` | name・url・installationID を okccl 向けに更新 |
| `platform-gitops/secrets/templates/argocd-repo-creds.yaml` | 同上（テンプレート側） |
| `platform-charts/` 4ファイル | ccl-labs → okccl 置換 |
| `sample-backend/` 4ファイル | ccl-labs → okccl 置換 |
| `sample-frontend/` 5ファイル | ccl-labs → okccl 置換 |
| `platform-docs/` 6ファイル | ccl-labs → okccl 置換 |
