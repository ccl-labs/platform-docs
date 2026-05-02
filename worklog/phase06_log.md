# Phase 6 作業ログ

## 事前準備

### sample-service リポジトリの確認
```bash
ls ~/sample-service
git remote -v
git log --oneline -5
```

### GitHub Actions 権限設定
- `okccl/sample-service` の Settings → Actions → General → Workflow permissions → Read and write permissions
- `okccl/platform-gitops` の Settings → Actions → General → Workflow permissions → Read and write permissions

---

## ステップ1: GitHub Actions ワークフローの作成

### ディレクトリ作成
```bash
mkdir -p ~/sample-service/.github/workflows
```

### build.yaml の作成
```bash
cat > ~/sample-service/.github/workflows/build.yaml << 'YAML'
name: Build and Push

on:
  push:
    branches:
      - main

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: okccl/sample-service

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}
    steps:
      - name: リポジトリのチェックアウト
        uses: actions/checkout@v4

      - name: GHCRへのログイン
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: イメージメタデータの生成
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=,format=short

      - name: イメージのビルドとプッシュ
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}

  trigger-update:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
      - name: update-gitopsワークフローをdispatch
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.actions.createWorkflowDispatch({
              owner: context.repo.owner,
              repo: context.repo.repo,
              workflow_id: 'update-gitops.yaml',
              ref: 'main',
              inputs: {
                image_tag: '${{ needs.build-and-push.outputs.image_tag }}',
                image_repository: 'ghcr.io/okccl/sample-service'
              }
            })
YAML
```

### update-gitops.yaml の作成
```bash
cat > ~/sample-service/.github/workflows/update-gitops.yaml << 'YAML'
name: Update GitOps

on:
  workflow_call:
    inputs:
      image_tag:
        required: true
        type: string
      image_repository:
        required: true
        type: string
  workflow_dispatch:
    inputs:
      image_tag:
        required: true
        type: string
      image_repository:
        required: true
        type: string

jobs:
  update-gitops:
    runs-on: ubuntu-latest
    steps:
      - name: platform-gitopsのチェックアウト
        uses: actions/checkout@v4
        with:
          repository: okccl/platform-gitops
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: imageの更新
        run: |
          sed -i "s|^\(\s*repository:\s*\).*|\1${{ inputs.image_repository }}|" apps/sample-service/values.yaml
          sed -i "s/^\(\s*tag:\s*\).*/\1${{ inputs.image_tag }}/" apps/sample-service/values.yaml

      - name: 変更をcommit & push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add apps/sample-service/values.yaml
          git commit -m "chore: update sample-service image to ${{ inputs.image_repository }}:${{ inputs.image_tag }}"
          git push
YAML
```

### ワークフローのpush
```bash
cd ~/sample-service
git add .github/
git commit -m "feat: add CI/CD workflows"
git push origin main
```

---

## ステップ2: クロスリポジトリアクセス用PATの設定

### GITOPS_TOKEN（Classic PAT）の作成
- `https://github.com/settings/tokens/new` でClassic PATを作成
- Scopes: `repo`（platform-gitopsへのpush権限）
- `okccl/sample-service` の Settings → Secrets → Actions → New repository secret
  - Name: `GITOPS_TOKEN`
  - Value: 発行したトークン

### update-gitops.yaml のトークンを GITOPS_TOKEN に変更
```bash
cd ~/sample-service
git add .github/workflows/update-gitops.yaml
git commit -m "fix: use GITOPS_TOKEN for cross-repo access"
git push origin main
```

---

## ステップ3: imagePullSecret の設定

### ghcr-pull 用 Classic PAT の作成
- `https://github.com/settings/tokens/new` でClassic PATを作成
- Scopes: `read:packages`

### クラスタへの Secret 登録
```bash
kubectl create secret docker-registry ghcr-pull-secret \
  --namespace sample-service \
  --docker-server=ghcr.io \
  --docker-username=okccl \
  --docker-password=<TOKEN>
```

### common-app Library Chart に imagePullSecrets を追加
`~/platform-charts/charts/common-app/templates/_helpers.tpl` の Deployment テンプレートに以下を追加：
```yaml
spec:
  {{- if .Values.image.pullSecretName }}
  imagePullSecrets:
    - name: {{ .Values.image.pullSecretName }}
  {{- end }}
```

```bash
cd ~/platform-charts
git add .
git commit -m "feat: add imagePullSecrets support to common-app"
git push origin main
```

### platform-gitops の values.yaml に pullSecretName を追加
```bash
cd ~/platform-gitops
sed -i "s/pullPolicy: IfNotPresent/pullPolicy: IfNotPresent\n  pullSecretName: ghcr-pull-secret/" apps/sample-service/values.yaml
git add .
git commit -m "feat: add pullSecretName to sample-service values"
git push origin main
```

### ArgoCD sync
```bash
argocd app sync sample-service --grpc-web
```

---

## エンドツーエンド動作確認

### v0.2.0 への更新
```bash
cat > ~/sample-service/src/index.html << 'HTML'
<!DOCTYPE html>
<html>
<body>
  <h1>sample-service</h1>
  <p>version: v0.2.0</p>
</body>
</html>
HTML

cd ~/sample-service
git add .
git commit -m "feat: update to v0.2.0"
git push origin main
```

### 確認
```bash
kubectl get pods -n sample-service
curl -k https://sample-service.localhost
# -> version: v0.2.0 が表示されることを確認
```
