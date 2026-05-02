# Phase 10 作業ログ

## 対象リポジトリ
- `platform-gitops` : `~/platform-gitops`
- `platform-infra`  : `~/platform-infra`
- `platform-charts` : `~/platform-charts`

---

## タスク1: SOPS×Age による Secrets as Code

### ツールインストール（mise管理）
```bash
mise use --global sops@3.12.2
mise use --global age@1.3.1

sops --version
age --version
age-keygen --version
```

### Ageキーペアの生成
```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
cat ~/.config/sops/age/keys.txt
```

### SOPSの設定ファイル作成
```bash
cat > ~/platform-gitops/.sops.yaml << 'EOF'
creation_rules:
  - path_regex: secrets/.*\.yaml$
    age: age1unp3mcue0u65eu28ls4l037jmn3umv6fusa7ggjeduep59tsvussxdurgs
EOF
```

### ディレクトリ作成・gitignore設定
```bash
mkdir -p ~/platform-gitops/secrets/encrypted
mkdir -p ~/platform-gitops/secrets/templates

cat >> ~/platform-gitops/.gitignore << 'EOF'

# SOPSの平文テンプレート（暗号化前）
secrets/templates/
EOF
```

### テンプレートファイル作成
```bash
# ghcr-pat
cat > ~/platform-gitops/secrets/templates/ghcr-pat.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: ghcr-pat
  namespace: argocd
type: Opaque
stringData:
  username: ccl
  password: <GitHubのPAT>
  email: cclima12@gmail.com
EOF

# minio-auth
cat > ~/platform-gitops/secrets/templates/minio-auth.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: minio-auth
  namespace: minio
type: Opaque
stringData:
  rootUser: minioadmin
  rootPassword: minioadmin123
EOF

# minio-backup-secret
cat > ~/platform-gitops/secrets/templates/minio-backup-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: minio-backup-secret
  namespace: sample-app
type: Opaque
stringData:
  ACCESS_KEY_ID: minioadmin
  ACCESS_SECRET_KEY: minioadmin123
EOF
```

### 暗号化
```bash
sops encrypt ~/platform-gitops/secrets/templates/ghcr-pat.yaml \
  > ~/platform-gitops/secrets/encrypted/ghcr-pat.yaml

sops encrypt ~/platform-gitops/secrets/templates/minio-auth.yaml \
  > ~/platform-gitops/secrets/encrypted/minio-auth.yaml

sops encrypt ~/platform-gitops/secrets/templates/minio-backup-secret.yaml \
  > ~/platform-gitops/secrets/encrypted/minio-backup-secret.yaml
```

### 動作確認（dry-run）
```bash
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops decrypt ~/platform-gitops/secrets/encrypted/ghcr-pat.yaml \
  | kubectl apply -f - --dry-run=client

SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops decrypt ~/platform-gitops/secrets/encrypted/minio-auth.yaml \
  | kubectl apply -f - --dry-run=client

SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops decrypt ~/platform-gitops/secrets/encrypted/minio-backup-secret.yaml \
  | kubectl apply -f - --dry-run=client
```

### Makefile修正（platform-infra）
- `GITHUB_PAT` 引数を廃止
- 3つのSecret投入をすべてSOPS復号（`sops decrypt | kubectl apply -f -`）に置き換え
- `goldilocks用namespaceラベル付与` を `bootstrap-sync` に追加
- 変数 `SECRETS_DIR` / `SOPS_AGE_KEY` を追加

### コミット
```bash
# platform-gitops
cd ~/platform-gitops
git add .sops.yaml secrets/encrypted/
git commit -m "feat: SOPS×AgeによるSecrets as Codeの導入"
git push origin main

git add .gitignore
git commit -m "chore: secrets/templates/をGitignoreに追加"
git push origin main

# platform-infra
cd ~/platform-infra
git add k3d/Makefile
git commit -m "feat: MakefileのSecret投入をSOPS復号に置き換え"
git push origin main
```

---

## タスク2: Developer Onboarding最適化

### mise.toml の更新（platform-infra）
```bash
cat > ~/platform-infra/.mise.toml << 'EOF'
[tools]
kubectl = "1.35.3"
helm    = "3.20.1"
k3d     = "5.8.3"
argocd  = "3.2.9"
sops    = "3.12.2"
age     = "1.3.1"
EOF
```

### .devcontainer の作成（platform-infra）
```bash
mkdir -p ~/platform-infra/.devcontainer

cat > ~/platform-infra/.devcontainer/devcontainer.json << 'EOF'
{
  "name": "platform-infra",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu-24.04",
  "features": {
    "ghcr.io/devcontainers/features/docker-outside-of-docker:1": {}
  },
  "onCreateCommand": "curl https://mise.run | sh && echo 'eval \"$(~/.local/bin/mise activate bash)\"' >> ~/.bashrc",
  "postCreateCommand": "mise install",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-kubernetes-tools.vscode-kubernetes-tools",
        "signageos.signageos-vscode-sops",
        "redhat.vscode-yaml"
      ]
    }
  },
  "mounts": [
    "source=${localEnv:HOME}/.ssh,target=/root/.ssh,type=bind,readonly",
    "source=${localEnv:HOME}/.config/sops,target=/root/.config/sops,type=bind,readonly"
  ]
}
EOF
```

### コミット
```bash
cd ~/platform-infra
git add .mise.toml .devcontainer/
git commit -m "feat: Developer Onboarding最適化"
git push origin main
```

---

## タスク3: common-app への preStop hook 追加

`platform-charts/charts/common-app/templates/_helpers.tpl` の `livenessProbe` の直前に追加。

```yaml
          lifecycle:
            preStop:
              exec:
                # Endpoints除外の伝搬を待ってからシャットダウン開始
                command: ["/bin/sh", "-c", "sleep 5"]
```

### コミット
```bash
cd ~/platform-charts
git add charts/common-app/templates/_helpers.tpl
git commit -m "feat: common-appにpreStop hookを追加"
git push origin main
```

---

## タスク4: Goldilocksラベルのコード化

`platform-gitops/platform/policy/platform-namespaces.yaml` に `sample-app` namespaceを追記。

```bash
cat >> ~/platform-gitops/platform/policy/platform-namespaces.yaml << 'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: sample-app
  labels:
    # Goldilocksダッシュボードの表示対象
    goldilocks.fairwinds.com/enabled: "true"
EOF
```

### コミット
```bash
cd ~/platform-gitops
git add platform/policy/platform-namespaces.yaml
git commit -m "feat: sample-app namespaceをコード化しGoldilocksラベルを付与"
git push origin main
```

---

## タスク5: ArgoCD Warning類の解消

### SharedResourceWarning 解消（ClusterPolicy重複）

**原因**: `root` App が `platform/` を再帰的に読み込むことで、`kyverno-policies` App が管理する
ClusterPolicy（`add-default-labels` / `disallow-latest-tag` / `require-resource-limits`）と重複していた。

**対処**: `root` App の `directory.exclude` に `policy/policies/*` を追加。

```bash
# bootstrap/root.yaml の directory セクションを修正
# directory:
#   recurse: true
#   exclude: "policy/policies/*"

kubectl apply -f ~/platform-gitops/bootstrap/root.yaml

cd ~/platform-gitops
git add bootstrap/root.yaml
git commit -m "fix: root AppからClusterPolicyを除外しSharedResourceWarningを解消"
git push origin main
```

### RepeatedResourceWarning 解消（sample-app Namespace重複）

**原因**: `platform-namespaces.yaml` の `sample-app` と、`sample-backend` / `sample-frontend` App の
`CreateNamespace=true` が重複していた。

**対処**: `platform-namespaces.yaml` から `sample-app` を削除し、Goldilocksラベルは
`sample-backend` App の `managedNamespaceMetadata` で付与するように変更。

```bash
# apps/sample-backend.yaml に以下を追加
# syncOptions:
#   - ServerSideApply=true
# managedNamespaceMetadata:
#   labels:
#     goldilocks.fairwinds.com/enabled: "true"

# platform-namespaces.yaml から sample-app ブロック（末尾8行）を削除
head -n -8 ~/platform-gitops/platform/policy/platform-namespaces.yaml \
  > /tmp/platform-namespaces.yaml && \
  mv /tmp/platform-namespaces.yaml \
  ~/platform-gitops/platform/policy/platform-namespaces.yaml

cd ~/platform-gitops
git add apps/sample-backend.yaml platform/policy/platform-namespaces.yaml
git commit -m "fix: sample-app NamespaceのRepeatedResourceWarningを解消"
git push origin main
```

### ClusterExternalSecret OutOfSync 解消

**原因**: `ghcr-pull-secret` の `spec.externalSecretSpec.target.type` が ESO v1 のスキーマ外フィールドだった。
ServerSideApply でのsyncがスキーマバリデーションエラーになっていた。

**対処**: `target.type` を削除し、`template.type` のみで指定するように修正。

```bash
# platform/secrets/external-secrets/ghcr-pull-secret.yaml
# spec.externalSecretSpec.target.type を削除（target.name と template のみ残す）

cd ~/platform-gitops
git add platform/secrets/external-secrets/ghcr-pull-secret.yaml
git commit -m "fix: ClusterExternalSecretのtarget.typeを削除しOutOfSyncを解消"
git push origin main
```

---

## Helm v4 移行検討結果

2026年4月時点でArgoCD（v3.3.x）はHelm v4未サポートであることを確認。
Helm v4のSSAデフォルト化とArgoCDのフィールドオーナーシップ競合リスクが依然として存在するため、**v3継続を維持**。
Phase 10.5（DR演習）完了後に再確認する。
