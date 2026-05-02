
**目的：** WSLを完全に破棄・再構築し、`git clone` + `make` で環境が復元できることを実証する

---

## bootstrap.sh の設計思想

### 冪等性

何度実行しても同じ結果になるよう設計されている。
既にインストール済みのツールは `[SKIP]` でスキップし、未インストールのものだけ `[INFO]` → `[OK]` で処理する。

### カバー範囲

| 処理 | 内容 |
|---|---|
| System packages | git / curl / make / unzip / ca-certificates / direnv |
| Homebrew | Linuxbrew のインストール |
| mise | ツールバージョン管理のインストール |
| Docker Engine | daemon + CLI のインストール、dockerグループへの追加 |
| .bashrc integrations | Homebrew / mise / direnv / cd ~ の自動設定 |
| mise trust | リポジトリの `.mise.toml` を信頼済みに設定 |

### カバーしない範囲（手動が必要）

| 処理 | 理由 |
|---|---|
| SSH鍵の作成・GitHubへの登録 | 秘密鍵をGitに含めることはセキュリティ上不可 |
| git config（name / email） | 個人情報のためスクリプトにハードコードできない |
| `.wslconfig` のリソース設定 | WSL外（Windows側）の設定のため |

---

## WSL再構築の実施ログ

### 1. WSLの破棄

PowerShellで実行：

```powershell
wsl --unregister Ubuntu
```

### 2. Ubuntu 24.04の再インストール

```powershell
wsl --install -d Ubuntu-24.04
```

ユーザー名 `ccl` / パスワードを設定して初期セットアップ完了。

### 3. `.wslconfig` の再設定

PowerShellで実行：

```powershell
@"
[wsl2]
memory=24GB
processors=12
"@ | Out-File -FilePath "$env:USERPROFILE\.wslconfig" -Encoding utf8

wsl --shutdown
```

WSL再起動後に確認：

```bash
free -h   # 23Gi
nproc     # 12
```

### 4. SSH鍵の再作成とGitHub登録

```bash
ssh-keygen -t ed25519 -C "your@email.com" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```

GitHubのSettings → SSH and GPG keys → New SSH key に公開鍵を登録。

接続確認：

```bash
ssh -T git@github.com
# Hi okccl! You've successfully authenticated
```

### 5. git clone

```bash
cd ~
git clone git@github.com:okccl/platform-infra.git
cd platform-infra
```

### 6. bootstrap実行

`make` がまだ入っていないため、初回のみ直接スクリプトを実行：

```bash
bash scripts/bootstrap.sh
```

出力：

```
[INFO]  Updating apt packages...
[OK]    Homebrew installed
[OK]    mise installed
[OK]    Docker Engine installed (re-login required for group to take effect)
[OK]    Added Homebrew to .bashrc
[OK]    Added mise to .bashrc
[OK]    Added direnv to .bashrc
[OK]    Added default cd to .bashrc
[OK]    mise trust applied

Bootstrap complete. Next steps:

  0. Set git identity (if not yet configured):
     git config --global user.email "your@email.com"
     git config --global user.name "Your Name"

  1. source ~/.bashrc
  2. cd ~/platform-infra
  3. make init
  4. make check
```

### 7. git config

```bash
git config --global user.email "your@email.com"
git config --global user.name "Your Name"
```

### 8. 環境反映とツールインストール

```bash
source ~/.bashrc
cd ~/platform-infra
make init
make check
```

出力：

```
kubectl : gitVersion:v1.35.3
helm    : v3.20.1+ga2369ca
k3d     : k3d version v5.8.3
argocd  : argocd: v3.2.9+54e42d2
```

---

## 発見した問題点と修正内容

### 問題1: `mise trust` が必要

**現象：** 新しいWSLでは `.mise.toml` が未信頼状態のため `make init` が失敗する。

```
mise ERROR Config files in ~/platform-infra/.mise.toml are not trusted.
```

**修正：** `bootstrap.sh` にリポジトリルートへの `mise trust` を追加。

```bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="$(dirname "$SCRIPT_DIR")"
mise trust "$REPO_ROOT"
```

---

### 問題2: `source ~/.bashrc` で `cd ~` が発動する

**現象：** `.bashrc` に `cd ~` を追記していたため、`source ~/.bashrc` 実行時にホームディレクトリに飛ばされ、直後の `make` コマンドが失敗する。

```bash
source ~/.bashrc   # → cd ~ が発動してホームに移動
make check         # → platform-infra にいないためエラー
```

**修正：** 初回ログイン時のみ `cd ~` を実行するよう条件を追加。

```bash
# Before
cd ~

# After
[ -z "$BASH_SOURCED" ] && export BASH_SOURCED=1 && cd ~
```

---

### 問題3: git config が未設定

**現象：** WSL再構築後、git config が未設定のためコミットができない。

**対応：** スクリプトにハードコードはできないため、`bootstrap.sh` の完了メッセージに設定手順の案内を追加。

```
  0. Set git identity (if not yet configured):
     git config --global user.email "your@email.com"
     git config --global user.name "Your Name"
```

---

## 確定した復元手順（WSL再構築後）

```bash
# 1. PowerShell: .wslconfig の再設定
# 2. SSH鍵の再作成 + GitHubへの登録
ssh-keygen -t ed25519 -C "your@email.com" -f ~/.ssh/id_ed25519 -N ""
# → GitHubのSettings → SSH and GPG keys → New SSH key に登録

# 3. clone & bootstrap
git clone git@github.com:okccl/platform-infra.git
cd platform-infra
bash scripts/bootstrap.sh

# 4. git identity の設定
git config --global user.email "your@email.com"
git config --global user.name "Your Name"

# 5. 環境反映 & ツールインストール
source ~/.bashrc
cd ~/platform-infra
make init
make check
```

---

## 冪等性の確認

環境構築済みの状態で `make bootstrap` を実行し、全項目が `[SKIP]` になることを確認：

```
[INFO]  Updating apt packages...
[SKIP]  Homebrew already installed
[SKIP]  mise already installed
[SKIP]  Docker already installed
[OK]    mise trust applied
```

---

## リポジトリ構成（Phase 0.5完了時点）

```
platform-infra/
├── scripts/
│   └── bootstrap.sh    # WSL素の状態から全ツールを揃えるスクリプト
├── .mise.toml
├── .envrc
├── Makefile            # make bootstrap / make init / make check
└── README.md
```
