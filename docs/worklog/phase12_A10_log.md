# Phase 12-A10 作業ログ（CI/CD GitHub App移行・クラスター障害対応・ArgoCD OutOfSync修正）

## 概要

| # | 作業内容 |
|---|---|
| 1 | CI/CD: GITOPS_TOKEN を GitHub App トークン方式に移行（3リポジトリ） |
| 2 | sample-frontend: `packages: write` 権限不足の修正 |
| 3 | 【トラブル】クラスター再起動後の Cilium eBPF スタルルーティング障害対応 |
| 4 | sample-backend: Rollout のイメージ更新・カナリアプロモーション |
| 5 | ArgoCD: HTTPRoute・ExternalSecret の OutOfSync 修正 |
| 6 | Makefile: クラスター安全再起動手順の整備 |

---

## 1. CI/CD: GitHub App トークン方式への移行

前セッションで GitHub org を `okccl` → `ccl-labs` に移行した際、CI/CD で使用していた `GITOPS_TOKEN`（fine-grained PAT）が旧 org 向けであり `ccl-labs/platform-gitops` へのアクセス権がなかった。PAT 再発行ではなく GitHub App トークン方式（`actions/create-github-app-token@v3.1.1`）に直接移行した。

**GUI操作（ユーザーが実施）**: GitHub App を新規作成（Repository permissions: Contents/PRs Read & write）し、`ccl-labs` org にインストール。org レベルで `GITOPS_APP_CLIENT_ID`（Variable）と `GITOPS_APP_PRIVATE_KEY`（Secret）を登録。

`sample-backend`・`sample-frontend` の `build.yaml` では `trigger-update` ジョブにトークン生成ステップを追加し、`repository_dispatch` の Authorization ヘッダーを差し替えた。`platform-gitops` の `update-gitops.yaml` では、`actions/checkout` の `token` と `gh pr merge` の `GH_TOKEN` を差し替えた。

```bash
cd ~/sample-backend
git add .github/workflows/build.yaml
git commit -m "ci: GitHub App トークン方式に移行"
git push

cd ~/sample-frontend
git add .github/workflows/build.yaml
git commit -m "ci: GitHub App トークン方式に移行"
git push

cd ~/platform-gitops
git add .github/workflows/update-gitops.yaml
git commit -m "ci: GitHub App トークン方式に移行"
git push
```

---

## 2. sample-frontend: `packages: write` 権限不足の修正

CI/CD を動かしたところ sample-frontend の Actions で `denied: installation not allowed to Create organization package` が発生した。`sample-backend` には `permissions: packages: write` があったが `sample-frontend` には `permissions` ブロック自体が欠落していた。`build.yaml` に追記してコミット。

```bash
cd ~/sample-frontend
git add .github/workflows/build.yaml
git commit -m "ci: packages: write 権限を追加"
git push
```

---

## 3. 【トラブル】クラスター再起動後の Cilium eBPF 障害対応

前夜に `k3d cluster stop dev` でクラスターを停止してから WSL をシャットダウンし、翌朝起動したところ Kyverno Webhook タイムアウト（`context deadline exceeded`）と ArgoCD Redis 接続タイムアウト（`dial tcp 10.43.13.19:6379: i/o timeout`）が発生した。

**根本原因**: `k3d cluster stop` は Docker コンテナを停止するだけで Kubernetes の graceful shutdown を経由しない。Cilium の eBPF ルーティングマップはLinuxカーネルに保持される（コンテナ外）ため、再起動後に pod の veth インターフェース（ifindex）が変わっても古いマップが残存する。ClusterIP 経由の通信が古いエンドポイントにルーティングされてタイムアウトしていた。port-forward は eBPF マップを経由しないため正常動作していた。

`kube-system` Namespace は Kyverno Webhook 対象外のため、Cilium DaemonSet を直接再起動することで eBPF マップが現在のインターフェース情報で再構築された。Cilium 復旧後、ArgoCD Redis・ESO・Kyverno Webhook はすべて自動回復した。

```bash
kubectl rollout restart daemonset cilium -n kube-system
kubectl rollout status daemonset cilium -n kube-system --timeout=120s
```

---

## 4. sample-backend: Rollout イメージ更新・カナリアプロモーション

CI/CD 復旧後、`platform-gitops` の values.yaml は `ghcr.io/ccl-labs/sample-backend:d094371` に更新済みだったが、ArgoCD の server-side apply が Rollout リソースに対してno-op となっており、クラスター上のイメージが旧イメージ `ghcr.io/okccl/sample-backend:ca2e588` のままだった（Cilium 障害中に sync が詰まっていたことが原因）。`kubectl argo rollouts set image` で直接更新し、canary pod が Running になったことを確認してプロモーションした。

```bash
kubectl argo rollouts set image sample-backend -n sample-app \
  sample-backend=ghcr.io/ccl-labs/sample-backend:d094371

kubectl argo rollouts promote sample-backend -n sample-app
```

---

## 5. ArgoCD: HTTPRoute・ExternalSecret の OutOfSync 修正

`sample-backend` / `sample-frontend` の ArgoCD Application が HTTPRoute と ExternalSecret で OutOfSync が続いていた。Gateway API controller と ESO controller がデフォルト値をフィールドに補完するが、git 上のマニフェストにそれらが書かれていなかったことが原因。

まず `ignoreDifferences` で ExternalSecret のデフォルト補完フィールドをカバーし、あわせて HTTPRoute 用のエントリも追加した。`sample-backend.yaml` と `sample-frontend.yaml` を修正してプッシュ。ExternalSecret は Synced になったが HTTPRoute は OutOfSync のままだった。

```bash
cd ~/platform-gitops
git add apps/sample-backend.yaml apps/sample-frontend.yaml
git commit -m "fix: HTTPRoute・ExternalSecret のデフォルト補完フィールドを ignoreDifferences に追加"
git push
```

HTTPRoute は `jqPathExpressions` だけでは `backendRefs` 側の差分がカバーしきれていなかったため、マニフェストにデフォルト値（`parentRefs[].group/kind`・`backendRefs[].group/kind/weight`）を明示して根本解消した。

```bash
git add apps/sample-backend/manifests/httproute.yaml apps/sample-frontend/manifests/httproute.yaml
git commit -m "fix: HTTPRoute に Gateway API のデフォルト補完フィールドを明示"
git push

kubectl annotate application sample-backend sample-frontend \
  -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

ArgoCD GUI で両 Application が Synced / Healthy になったことを確認。

---

## 6. Makefile: クラスター安全再起動手順の整備

Cilium の eBPF 問題を踏まえ、`k3d cluster start` 後に必ず Cilium を再起動する手順を `platform-infra/Makefile` に追加した。以降は `make cluster-start` / `make cluster-restart` を使用する。

```bash
cd ~/platform-infra
git add Makefile
git commit -m "chore: クラスター安全再起動ターゲットを Makefile に追加"
git push
```

---

## 関連ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `sample-backend/.github/workflows/build.yaml` | GitHub App トークン方式に移行 |
| `sample-frontend/.github/workflows/build.yaml` | GitHub App トークン方式に移行・`permissions` 追加 |
| `platform-gitops/.github/workflows/update-gitops.yaml` | GitHub App トークン方式に移行 |
| `platform-gitops/apps/sample-backend.yaml` | HTTPRoute・ExternalSecret の `ignoreDifferences` 追加 |
| `platform-gitops/apps/sample-frontend.yaml` | HTTPRoute の `ignoreDifferences`・`RespectIgnoreDifferences=true` 追加 |
| `platform-gitops/apps/sample-backend/manifests/httproute.yaml` | Gateway API デフォルト値を明示 |
| `platform-gitops/apps/sample-frontend/manifests/httproute.yaml` | Gateway API デフォルト値を明示 |
| `platform-infra/Makefile` | `cluster-start` / `cluster-stop` / `cluster-restart` ターゲット追加 |
