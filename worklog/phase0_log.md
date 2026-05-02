## ゴール

`git clone` 後に `make init` 一発で全ツールが揃う状態を作る。

## 前提環境

| 項目 | 値 |
|---|---|
| OS (Windows) | Windows 11 |
| OS (WSL) | Ubuntu 24.04 LTS |
| WSL バージョン | 2 |
| CPU割当 | 12コア |
| メモリ割当 | 24GB |

---

## Step 0-1: WSL2環境の整備

### 1-1. WSL2バージョン確認

PowerShellで実行：

```powershell
wsl --list --verbose
```

Ubuntu が VERSION 2 で動いていることを確認する。

### 1-2. パッケージの最新化

```bash
sudo apt update && sudo apt upgrade -y
```

### 1-3. 基本ツールのインストール

```bash
sudo apt install -y git curl make unzip
```

動作確認：

```bash
git --version && curl --version && make --version && unzip -v
```

### 1-4. `.wslconfig` によるリソース設定

PowerShellで実行：

```powershell
@"
[wsl2]
memory=24GB
processors=12
"@ | Out-File -FilePath "$env:USERPROFILE\.wslconfig" -Encoding utf8
```

WSLを再起動して設定を反映：

```powershell
wsl --shutdown
```

WSL再起動後、割り当て確認：

```bash
free -h
nproc
```

### 1-5. デフォルト作業ディレクトリの設定

WSLネイティブのホームディレクトリ（`/home/<user>/`）で作業する。
`/mnt/c/...` 配下はWindowsファイルシステムのため、I/Oパフォーマンスが低く
DockerやK8sのワークロードに悪影響を及ぼす。

起動時に自動でホームに移動するよう設定：

```bash
echo 'cd ~' >> ~/.bashrc
source ~/.bashrc
```

---

## Step 0-2: ツールの手動インストール

インストール順序：

```
Homebrew → mise → kubectl / helm / k3d / argocd CLI → direnv → Docker Engine
```

### 2-1. Homebrew (Linuxbrew) のインストール

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

PATHを通す：

```bash
echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.bashrc
source ~/.bashrc
```

動作確認：

```bash
brew --version
# Homebrew 5.1.6
```

### 2-2. mise のインストール

```bash
curl https://mise.run | sh
```

シェルへの統合：

```bash
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc
source ~/.bashrc
```

動作確認：

```bash
mise --version
# 2026.4.16 linux-x64
```

### 2-3. CLIツールのインストール（mise）

| ツール | バージョン | 備考 |
|---|---|---|
| kubectl | 1.35.3 | 最新安定版 |
| helm | 3.20.1 | v4は出ているがエコシステム互換性からv3を選択。セキュリティサポートは2026年11月まで |
| k3d | 5.8.3 | 最新安定版 |
| argocd | 3.2.9 | 最新安定版 |

```bash
mise use -g kubectl@1.35.3 helm@3.20.1 k3d@5.8.3 argocd@3.2.9
```

動作確認：

```bash
kubectl version --client
helm version
k3d version
argocd version --client
```

### 2-4. direnv のインストール

```bash
sudo apt install -y direnv
```

シェルへの統合：

```bash
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc
```

動作確認：

```bash
direnv version
# 2.32.1
```

### 2-5. Docker Engine のインストール

Docker Desktopは不要。WSL2がVM相当の役割を担っているため、
Docker Engine（daemon + CLI）を直接インストールする。

```bash
# Add Docker's GPG key
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker apt repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin
```

sudoなしでDockerを使えるようにする：

```bash
sudo usermod -aG docker $USER
```

WSLを再起動してグループを反映（PowerShellで）：

```powershell
wsl --shutdown
```

動作確認：

```bash
docker --version
docker run hello-world
# Hello from Docker! が表示されればOK
```

---

## Step 0-3: コード化

### 3-1. リポジトリの作成

```bash
mkdir -p ~/platform-infra && cd ~/platform-infra
git init
git config --global init.defaultBranch main
git branch -m main
```

### 3-2. `.mise.toml` の作成

```bash
cat > .mise.toml << 'EOF'
[tools]
kubectl = "1.35.3"
helm    = "3.20.1"
k3d     = "5.8.3"
argocd  = "3.2.9"
EOF
```

miseの信頼設定：

```bash
mise trust
mise install
```

### 3-3. `.envrc` の作成

```bash
cat > .envrc << 'EOF'
mise_layout() {
  eval "$(mise hook-env 2>/dev/null)"
}
mise_layout
EOF
direnv allow .
```

### 3-4. `Makefile` の作成

```bash
cat > Makefile << 'EOF'
.DEFAULT_GOAL := help

.PHONY: help init check

help:
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-10s\033[0m %s\n", $$1, $$2}'

init: ## Install all tools via mise
	mise install

check: ## Show versions of all tools
	@echo "kubectl : $$(kubectl version --client -o json | grep gitVersion | head -1 | tr -d '\" ,')"
	@echo "helm    : $$(helm version --short)"
	@echo "k3d     : $$(k3d version | head -1)"
	@echo "argocd  : $$(argocd version --client --short 2>/dev/null | head -1)"
EOF
```

動作確認：

```bash
make help
make init
make check
```

### 3-5. `README.md` の作成

```bash
cat > README.md << 'EOF'
# platform-infra

Platform Engineering portfolio — Infrastructure as Code repository.

## Phase 0: Local Foundation

### What problem does this solve?

Eliminates "works on my machine" by codifying the entire local development
environment. Any engineer can reproduce the exact same toolset with a single command.

### Usage

​```bash
# Install all tools
make init

# Verify tool versions
make check
​```

### Tools managed by mise

| Tool    | Version |
|---------|---------|
| kubectl | 1.35.3  |
| helm    | 3.20.1  |
| k3d     | 5.8.3   |
| argocd  | 3.2.9   |

### Requirements

- WSL2 (Ubuntu 24.04 LTS)
- mise
- direnv
- Docker Engine
EOF
```

### 3-6. 初回コミット

```bash
git config --global user.email "your@email.com"
git config --global user.name "Your Name"
git add .
git commit -m "feat: Phase 0 - local foundation"
```

---

## Phase 0 完了確認

| 確認項目 | コマンド | 期待結果 |
|---|---|---|
| ツール一括インストール | `make init` | `all tools are installed` |
| バージョン表示 | `make check` | 全ツールのバージョンが表示される |
| 再現性 | `.mise.toml` にバージョン宣言あり | `git clone` 後に `make init` で環境再現可能 |

## 最終的なリポジトリ構成

```
platform-infra/
├── .mise.toml   # ツールとバージョンの宣言
├── .envrc       # miseの自動有効化
├── Makefile     # make init / make check
└── README.md    # Phaseの目的と使い方
```
