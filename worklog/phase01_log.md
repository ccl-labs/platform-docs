# Phase 1: k3d Cluster IaC 作業ログ

## 完了の定義
- `make cluster-create` 一発でクラスタが立ち上がる
- `kubectl get nodes` で全ノードが `Ready` と表示される
- `make cluster-delete` でクラスタが破棄される
- 破棄→再作成が繰り返せる（使い捨て可能な状態）
- クラスタ構成が `cluster.yaml` に宣言されており、再現性が保証されている

---

## Step 1-1: k3dクラスタ設定ファイルの作成

### 実施内容

```bash
mkdir -p ~/platform-infra/k3d
```

```bash
cat > ~/platform-infra/k3d/cluster.yaml << 'EOF'
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: dev
servers: 1
agents: 2
ports:
  - port: 80:80
    nodeFilters:
      - loadbalancer
  - port: 443:443
    nodeFilters:
      - loadbalancer
options:
  kubeconfig:
    updateDefaultKubeconfig: true
    switchCurrentContext: true
EOF
```

### 設計上の決定事項
- `name: dev` — クラスタ名。後のPhaseでArgoCDが参照する
- `servers: 1 / agents: 2` — control plane 1台 + worker 2台の最小構成
- `ports` — Phase 3（Ingress導入）に備え、ロードバランサーノードにHTTP/HTTPSをマッピング
- `kubeconfig` — 作成時に自動でkubeconfigにマージし、contextも切り替え

---

## Step 1-2: k3d/Makefileの作成

```bash
cat > ~/platform-infra/k3d/Makefile << 'EOF'
CLUSTER_CONFIG := cluster.yaml

.PHONY: cluster-create cluster-delete cluster-status

cluster-create:
	k3d cluster create --config $(CLUSTER_CONFIG)

cluster-delete:
	k3d cluster delete --config $(CLUSTER_CONFIG)

cluster-status:
	kubectl get nodes -o wide
EOF
```

### 各ターゲットの意図
- `cluster-create` — `cluster.yaml` を参照してクラスタを作成（設定ファイルが単一の真実の源）
- `cluster-delete` — 同じ設定ファイルからクラスタ名を解決して破棄
- `cluster-status` — ノードの状態とIPを一覧表示

**注意：** Makefileのインデントはタブ文字が必須。エディタで編集する場合はスペースではなくタブを使用すること。

---

## Step 1-3: クラスタの動作確認

### クラスタの作成と確認

```bash
cd ~/platform-infra/k3d
make cluster-create
make cluster-status
```

**実際の出力：**
```
NAME               STATUS   ROLES                  AGE   VERSION        INTERNAL-IP
k3d-dev-agent-0    Ready    <none>                 70s   v1.31.5+k3s1   172.19.0.3
k3d-dev-agent-1    Ready    <none>                 70s   v1.31.5+k3s1   172.19.0.4
k3d-dev-server-0   Ready    control-plane,master   73s   v1.31.5+k3s1   172.19.0.2
```

### APIサーバーへの接続確認

```bash
kubectl cluster-info
```

**実際の出力：**
```
Kubernetes control plane is running at https://0.0.0.0:41371
CoreDNS is running at https://0.0.0.0:41371/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:41371/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

### 破棄→再作成の検証（使い捨て可能性の確認）

```bash
make cluster-delete
make cluster-status  # 削除後はエラーになる（正常な挙動）
make cluster-create
make cluster-status  # 再度3ノードがReadyになることを確認
```

**削除後の `cluster-status` について：**
クラスタ削除後に `cluster-status` を実行するとAPIサーバーへの接続拒否エラーが出るが、
これはクラスタが存在しない状態での正常な挙動。

```
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

また `make cluster-delete` 実行時、kubeconfigからのエントリ削除も自動で行われる：
```
INFO[0000] Removing cluster details from default kubeconfig...
INFO[0000] Successfully deleted cluster dev!
```

---

## Step 1-4: README.mdの更新

`platform-infra/README.md` にPhase 1のセクションを追記。
既存のPhase 0セクションのコメントも日本語に統一。

**最終的なREADME.md：**

```markdown
# platform-infra
Platform Engineering portfolio — Infrastructure as Code repository.

## Phase 0: Local Foundation

### このPhaseで解決すること
「自分のマシンでは動く」問題を、ローカル開発環境の完全なコード化によって解消する。
エンジニアは1コマンドで同一のツールセットを再現できる。

### 使い方
# 全ツールをインストール
make init

# ツールバージョンを確認
make check

### miseで管理するツール
| ツール    | バージョン |
|---------|---------|
| kubectl | 1.35.3  |
| helm    | 3.20.1  |
| k3d     | 5.8.3   |
| argocd  | 3.2.9   |

### 前提条件
- WSL2 (Ubuntu 24.04 LTS)
- mise
- direnv
- Docker Engine

## Phase 1: k3d Cluster IaC

### このPhaseで解決すること
クラスタ構成を `cluster.yaml` に宣言することで、手動セットアップを排除する。
エンジニアは1コマンドで同一の開発クラスタを作成・破棄・再作成できる。

### 使い方
# クラスタを作成
make -C k3d cluster-create

# ノードの状態を確認
make -C k3d cluster-status

# クラスタを破棄
make -C k3d cluster-delete

### クラスタ構成（k3d/cluster.yaml）
| 項目 | 値 |
|---|---|
| クラスタ名 | dev |
| コントロールプレーンノード数 | 1 |
| エージェントノード数 | 2 |
| HTTPポート | 80 |
| HTTPSポート | 443 |

### 設計上の決定事項
- ポート80/443はPhase 3（Ingress導入）に備え、ロードバランサーノードにマッピング済み。
- クラスタ作成時にkubeconfigへの自動マージとコンテキストの切り替えを行う。
```

### コミット

```bash
cd ~/platform-infra
git add k3d/cluster.yaml k3d/Makefile README.md
git commit -m "feat: Phase 1 - k3d cluster IaC

- Add k3d/cluster.yaml: declarative cluster config (1 server, 2 agents)
- Add k3d/Makefile: cluster-create / cluster-delete / cluster-status
- Update README.md: add Phase 1 section"
git push origin main
```

---

## Phase 1完了時点のリポジトリ構成

```
platform-infra/
├── k3d/
│   ├── cluster.yaml    # クラスタ構成定義
│   └── Makefile        # クラスタ操作の自動化
├── scripts/
│   └── bootstrap.sh
├── .mise.toml
├── .envrc
├── Makefile
└── README.md
```
