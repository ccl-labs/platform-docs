# Phase12-A14 作業ログ

## テーマ
GitHub 組織アカウント（ccl-labs）から個人アカウント（okccl）への移行

## 背景
Findy 等の転職サイトの GitHub 連携では Organization リポジトリが参照されないため、ポートフォリオの可視性向上を目的に全リポジトリを個人アカウントへ transfer した。

---

## 実施内容

### 1. リポジトリ transfer（GitHub UI）
以下 8 リポジトリを `ccl-labs` → `okccl` へ transfer：
- platform-infra / platform-gitops / platform-charts / platform-docs
- sample-backend / sample-frontend / apps-gitops / backstage

### 2. コード一括置換
全リポジトリで `ccl-labs` → `okccl` を sed で一括置換・コミット・push。  
対象：ArgoCD Application の repoURL / Backstage catalog target URL / ghcr.io イメージ参照 / catalog-info.yaml / GitHub Actions workflows / Scaffolder テンプレート等。

### 3. git remote URL 更新
各リポジトリの remote origin を `git@github.com:okccl/<repo>.git` に変更。

### 4. GitHub App 移行
- ArgoCD 用 GitHub App：Transfer → okccl に再インストール
- Backstage 用 GitHub App / OAuth App：OAuth App は元から okccl 個人アカウントに紐づいていたため変更不要。GitHub App は Transfer 済み。

### 5. argocd-repo-creds 更新
GitHub App の Installation ID が変更（130426273 → 131473848）。  
SOPS 暗号化ファイルの name・url・installationID を更新し、クラスタに apply。

### 6. Makefile 修正（cluster-start）
Cilium eBPF マップの初期化完了を保証するため、`cilium status --wait` を追加。  
`cilium` CLI が未インストールのため `kubectl exec -n kube-system ds/cilium -- cilium status --wait` で実装。

---

## 発生した問題と対応

### Kyverno webhook タイムアウト（複数回）
- **原因**：Cilium 起動タイミングで Kyverno admission controller のネットワークが正常に設定されなかった
- **対応**：`kubectl rollout restart deployment/kyverno-admission-controller -n kyverno`

### ArgoCD repo-server CrashLoopBackOff
- **原因**：同上。Cilium CNI ソケット未準備状態で Pod が起動し、liveness probe が恒常的にタイムアウト
- **対応**：`kubectl rollout restart deployment/argocd-repo-server -n argocd`

### ArgoCD app-controller Redis DNS タイムアウト
- **原因**：同上の Cilium タイミング問題
- **対応**：`kubectl rollout restart statefulset/argocd-application-controller -n argocd`

### クロスノード Pod 通信断（Keycloak DB 接続失敗）
- **原因**：`make cluster-restart` 後に Docker がコンテナへ異なる IP を再割り当て。`CiliumNode` リソースが古い IP のまま残留し、VXLAN トンネルが誤ったホスト IP に向いた
- **対応**：`kubectl delete ciliumnodes --all` → Cilium DaemonSet rollout restart で IP 再登録

---

## 残タスク
- ghcr.io イメージを okccl 名義で push（sample-backend / sample-frontend / backstage）
- `ghcr-pat` シークレットを okccl の PAT に更新
- `make cluster-restart` 後の CiliumNode IP ズレ問題の恒久対策（次セッションで検討）
