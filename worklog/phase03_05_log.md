# セッション作業ログ：Phase 3 持ち越し課題 1・2


## スコープ
- 持ち越し課題1: ArgoCDの自己管理設定
- 持ち越し課題2: クラスタ再作成手順のbootstrapスクリプト化

---

## 課題1: ArgoCDの自己管理設定

### 概要
`helm upgrade`（手動）でしか反映できなかったArgoCDの設定変更を、ArgoCDのApplicationとして自己管理させることで、`values.yaml` の変更をGitOpsフローで自動反映できるようにした。

### 手順

#### Step 1: 既存構成の確認

```bash
cat ~/platform-gitops/bootstrap/root.yaml
cat ~/platform-gitops/platform/argocd/values.yaml
```

- `root.yaml` が `platform/` 配下を `recurse: true` で管理していることを確認
- `platform/argocd/application.yaml` を作成するだけでArgoCDが自動検知する構成であることを確認

#### Step 2: application.yaml の作成

`$values` 構文を使うため `sources`（複数ソース）形式で定義。

```bash
cat > ~/platform-gitops/platform/argocd/application.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://argoproj.github.io/argo-helm
      chart: argo-cd
      targetRevision: 9.5.2
      helm:
        valueFiles:
          - $values/platform/argocd/values.yaml
    - repoURL: git@github.com:okccl/platform-gitops.git
      targetRevision: HEAD
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
      - RespectIgnoreDifferences=true
EOF
```

#### Step 3: Gitにpushしてsync

```bash
cd ~/platform-gitops
git add platform/argocd/application.yaml
git commit -m "feat: add argocd self-managed application"
git push origin main
argocd app sync root --async
```

#### Step 4: 動作確認

`argocd app list` で `argocd/argocd` が `Synced / Healthy` で出現することを確認。

```bash
argocd app list
```

#### Step 5: 自己管理の動作テスト

`values.yaml` にコメントを追加してpushし、ArgoCDが自動でsyncすることを確認。

```bash
sed -i 's/timeout.reconciliation: 180s/timeout.reconciliation: 180s # self-managed test/' \
  ~/platform-gitops/platform/argocd/values.yaml

cd ~/platform-gitops
git add platform/argocd/values.yaml
git commit -m "test: verify argocd self-managed sync"
git push origin main
```

確認後、コメントを元に戻す。

```bash
sed -i 's/timeout.reconciliation: 180s # self-managed test/timeout.reconciliation: 180s/' \
  ~/platform-gitops/platform/argocd/values.yaml

cd ~/platform-gitops
git add platform/argocd/values.yaml
git commit -m "revert: remove self-managed test comment"
git push origin main
```

### 補足: ArgoCD CLIのIngress経由ログイン

Windows再起動後はport-forwardが切れる。Ingress経由でログインすることでport-forward不要になる。

```bash
argocd login argocd.localhost \
  --username admin \
  --password $(argocd admin initial-password -n argocd | head -1) \
  --grpc-web
```

---

## 課題2: クラスタ再作成手順のbootstrapスクリプト化

### 概要
手動だったクラスタ再作成後の復旧手順（ArgoCD install → Root App適用 → リポジトリ登録 → 段階的sync）を `platform-infra/k3d/Makefile` に組み込んだ。

### 手順

#### Step 1: Makefileの更新

```bash
cat > ~/platform-infra/k3d/Makefile << 'EOF'
CLUSTER_CONFIG  := cluster.yaml
CLUSTER_NAME    := dev
GITOPS_REPO     := git@github.com:okccl/platform-gitops.git
SSH_KEY         := ~/.ssh/id_ed25519
ARGOCD_NS       := argocd

.PHONY: cluster-create cluster-delete cluster-status bootstrap bootstrap-argocd bootstrap-sync

cluster-create:
	k3d cluster create --config $(CLUSTER_CONFIG)

cluster-delete:
	k3d cluster delete --config $(CLUSTER_CONFIG)

cluster-status:
	kubectl get nodes -o wide

# クラスタ再作成からsyncまでを一括実行
bootstrap: cluster-create bootstrap-argocd bootstrap-sync

# ArgoCDインストール・Root App適用・リポジトリ登録
bootstrap-argocd:
	# ArgoCDインストール
	helm repo add argocd https://argoproj.github.io/argo-helm
	helm repo update
	helm install argocd argocd/argo-cd \
		-n $(ARGOCD_NS) \
		--create-namespace \
		-f ~/platform-gitops/platform/argocd/values.yaml \
		--wait
	# Root App適用
	kubectl apply -f ~/platform-gitops/bootstrap/root.yaml
	kubectl apply -f ~/platform-gitops/bootstrap/apps-root.yaml
	# ArgoCD CLIログイン
	@ARGOCD_PASS=$$(argocd admin initial-password -n $(ARGOCD_NS) | head -1) && \
	argocd login argocd.localhost \
		--username admin \
		--password $$ARGOCD_PASS \
		--grpc-web
	# リポジトリ登録
	argocd repo add $(GITOPS_REPO) --ssh-private-key-path $(SSH_KEY)

# コンポーネントの段階的sync（CRDの依存順序に従う）
bootstrap-sync:
	# cert-manager
	argocd app sync root --resource argoproj.io:Application:cert-manager --async
	argocd app sync cert-manager --async
	kubectl wait --for=condition=established \
		crd/clusterissuers.cert-manager.io --timeout=120s
	# ingress-nginx
	argocd app sync root --resource argoproj.io:Application:ingress-nginx --async
	argocd app sync ingress-nginx --async
	# external-secrets
	argocd app sync root --resource argoproj.io:Application:external-secrets --async
	argocd app sync external-secrets --server-side --async
	kubectl wait --for=condition=established \
		crd/clustersecretstores.external-secrets.io --timeout=120s
	# argocd自己管理・残りのコンポーネント
	argocd app sync root --server-side --async
	@echo "bootstrap-sync 完了。'argocd app list' で状態を確認してください。"
EOF
```

#### Step 2: Gitにpush

```bash
cd ~/platform-infra
git add k3d/Makefile
git commit -m "feat: add bootstrap targets to Makefile"
git push origin main
```

### 設計上の決定事項

- **SSH鍵パスの固定:** `~/.ssh/id_ed25519` にハードコード。異なる環境でも鍵パスを合わせることで対応する。README等に前提条件として記載すること。
- **sleepからkubectl waitへの変更:** 固定待機をCRD確立の確認に置き換え、より堅牢な実装にした。
- **検証:** クラスタ削除・再作成による実動作検証はスキップ。元手順がPhase 3で実績済みのため、次回クラスタ再作成時（Phase 8前等）に自然に検証される。
