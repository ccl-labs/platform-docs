# Phase12-A14 作業ログ

| # | 作業内容 |
|---|---|
| 1 | GitHub ccl-labs → okccl 移行（リポジトリ transfer・コード置換・GitHub App 移行・argocd-repo-creds 更新） |
| 2 | Makefile cluster-start に Cilium 疎通確認を追加 |
| 3 | トラブルシュート：Cilium 起動タイミング問題による各コンポーネント障害 |
| 4 | トラブルシュート：make cluster-restart 後の CiliumNode IP ズレによるクロスノード通信断 |
| 5 | Makefile cluster-start に CiliumNode 削除を追加（IP ズレ恒久対策） |
| 6 | ghcr-pull-secret 仕組みの削除（packages public 化により不要と判明） |
| 7 | トラブルシュート：GitHub Actions CI/CD 疎通（GITOPS_APP_PRIVATE_KEY 欠落・token スコープ不足・imagePullSecrets 残留）|
| 8 | トラブルシュート：全 sync 後に sample-backend/frontend が Unknown（apps-gitops の application.yaml に ccl-labs 残留）|

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

## 5. Makefile cluster-start に CiliumNode 削除を追加

### 背景

セクション 4 で「次セッションで検討」とした恒久対策を実施した。`k3d cluster stop` → `start` で Docker がノードコンテナに異なる IP を再割り当てすると、`CiliumNode` リソースが古い IP のまま残留してクロスノード通信が断絶する。`kubectl delete ciliumnodes --all` を `cluster-start` の Cilium 再起動前に挿入することで、毎回正しい IP で再登録させる。

副作用の有無を確認するため、クラスターの IPAM モードを事前に確認した。

```bash
kubectl get configmap cilium-config -n kube-system -o yaml | grep ipam
# ipam: kubernetes
```

`kubernetes` モードでは Pod CIDR は Node オブジェクトの `spec.podCIDR` が source of truth であり、`CiliumNode` はそのミラーにすぎない。削除しても Cilium エージェントが同一 CIDR で再登録するため副作用なし。

```makefile
@echo ">>> CiliumNode キャッシュをクリア中..."
kubectl delete ciliumnodes --all
```

---

## 6. ghcr-pull-secret 仕組みの削除

### 背景

`ghcr-pat.yaml`（SOPS 暗号化）の調査中に、`lastmodified: 2026-05-05` と古く username が `ccl`（誤記）であることに気づいた。調査の結果、以下の経緯が判明した。

- もともと ghcr.io パッケージは private だったため imagePullSecret が必要だった
- ポートフォリオ公開のために packages を public に変更した際に仕組みを削除し忘れた
- ccl-labs 移行時もそのまま放置されたため、ccl-labs には PAT が存在せず、5/5 から更新されないファイルとして残っていた

public パッケージは認証不要で pull できるため、仕組み全体を削除した。

### 実施手順

以下のファイルを削除し、関連する参照をすべて除去した。

| 削除ファイル | 内容 |
|---|---|
| `platform/secrets/sources/ghcr-pat.yaml` | SOPS 暗号化 PAT |
| `platform/secrets/sources/ghcr-pull-secret-source.yaml` | ExternalSecret |
| `platform/secrets/config/ghcr-pull-secret.yaml` | ClusterExternalSecret |
| `platform/policy/policies/generate-ghcr-pull-secret.yaml` | Kyverno 配布 Policy |

参照削除対象：`kustomization.yaml` / `secret-generator.yaml` / `backstage values.yaml` / gitops-skeleton 2ファイル / namespace.yaml 2ファイル / `backstage.yaml`（managedNamespaceMetadata ラベル）

---

## 7. トラブルシュート：GitHub Actions CI/CD 疎通

### 問題

`ccl-labs` → `okccl` 移行後に `sample-backend` / `sample-frontend` の Build and Push workflow が `trigger-update` ジョブで失敗。`platform-gitops` の `update-gitops` workflow も連鎖して失敗。

```
Error: The 'client-id' (or deprecated 'app-id') input must be set to a non-empty string.
```

### 原因

複数の問題が重なっていた。

| 問題 | 原因 |
|---|---|
| `GITOPS_APP_PRIVATE_KEY` 欠落 | ccl-labs org レベルの Secret として登録されていたため、repo transfer 時に引き継がれなかった |
| `GITOPS_APP_CLIENT_ID` 欠落（sample-frontend のみ） | 同上。sample-backend は repo レベルにも設定されていたが sample-frontend はなかった |
| `platform-gitops` の同 Secret 欠落 | platform-gitops も org レベルのみで管理されていた |
| update-gitops の token スコープ不足 | `actions/create-github-app-token` で `repositories` 未指定のため token が `platform-gitops` のみにスコープされ、`apps-gitops` への push が 403 になった |
| ImagePullBackOff（sample-frontend） | `apps-gitops` の values.yaml に `pullSecretName: ghcr-pull-secret` が残留しており、存在しない Secret を参照して pull 認証が失敗した |

### 対処

**GITOPS_APP_PRIVATE_KEY の復旧**

ローカルに残っていた private key を SOPS で暗号化して保存し、`gh secret set` で各リポジトリに設定した。

```bash
# SOPS ファイル作成
sops platform/secrets/sources/gitops-github-app-source.yaml

# GitHub Actions secret に設定
sops -d .../gitops-github-app-source.yaml \
  | python3 -c "import sys,yaml; print(yaml.safe_load(sys.stdin)['stringData']['privateKey'])" \
  | gh secret set GITOPS_APP_PRIVATE_KEY -R okccl/sample-backend

# sample-frontend / platform-gitops も同様
```

**GITOPS_APP_CLIENT_ID の設定（sample-frontend / platform-gitops）**

```bash
gh variable set GITOPS_APP_CLIENT_ID -R okccl/sample-frontend --body "Iv23liMGIykpOssjwg6a"
gh variable set GITOPS_APP_CLIENT_ID -R okccl/platform-gitops --body "Iv23liMGIykpOssjwg6a"
```

**update-gitops の token スコープ修正**

```yaml
uses: actions/create-github-app-token@v3.1.1
with:
  client-id: ${{ vars.GITOPS_APP_CLIENT_ID }}
  private-key: ${{ secrets.GITOPS_APP_PRIVATE_KEY }}
  owner: okccl
  repositories: apps-gitops   # 追加
```

**apps-gitops の pullSecretName 削除**

既存の `apps-gitops/apps/sample-backend/values.yaml` と `apps/sample-frontend/values.yaml` に `pullSecretName: ghcr-pull-secret` が残留していたため、GitHub API 経由で削除した。

---

## 8. トラブルシュート：全 sync 後に sample-backend/frontend が Unknown

### 問題

ArgoCD で全 Application を sync したところ、sample-backend / sample-frontend が `Unknown` になった。

```
application repo https://github.com/ccl-labs/apps-gitops.git is not permitted in project 'sample-apps'
application repo https://github.com/ccl-labs/platform-charts.git is not permitted in project 'sample-apps'
```

### 原因

`apps-gitops/apps/sample-backend/application.yaml` と `apps/sample-frontend/application.yaml` の `sources[].repoURL` が `ccl-labs` のまま残っていた。前回の一括置換では apps-gitops がローカルにクローンされていなかったため対象外になっていた。sync で ArgoCD がこの定義を適用し、AppProject の許可リストと不一致になった。

### 対処

GitHub API で両ファイルの `ccl-labs` → `okccl` を修正・push。ArgoCD が自動 sync で反映。

あわせて apps-gitops を `/home/ccl/apps-gitops` にクローンし、今後の検索漏れを防止した。

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
| `platform-infra/Makefile` | cluster-start に `kubectl delete ciliumnodes --all` 追加 |
| `platform-gitops/platform/secrets/sources/` 4ファイル削除 | ghcr-pat・ghcr-pull-secret 関連ファイル |
| `platform-gitops/` 8ファイル | ghcr-pull-secret 参照削除（kustomization / values / namespace 等） |
| `platform-gitops/platform/secrets/sources/gitops-github-app-source.yaml` | ccl-labs-gitops App private key を SOPS 保存（新規） |
| `platform-gitops/.github/workflows/update-gitops.yaml` | App token スコープに `repositories: apps-gitops` 追加 |
| `apps-gitops/apps/sample-backend/values.yaml` | `pullSecretName` 削除 |
| `apps-gitops/apps/sample-frontend/values.yaml` | `pullSecretName` 削除 |
| `apps-gitops/apps/sample-backend/application.yaml` | sources の repoURL を ccl-labs → okccl に修正 |
| `apps-gitops/apps/sample-frontend/application.yaml` | sources の repoURL を ccl-labs → okccl に修正 |
