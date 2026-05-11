# Phase 12-A9 作業ログ（GitHub org移行・Backstage完成）

## 概要

GitHub 個人アカウント（okccl）から新規 org（okccl）へのリポジトリ移行と、
Backstage の残課題（Keycloak ログイン・GitHub PR プラグイン・CSP・カタログ・TechDocs）を完了。

---

## 1. GitHub org 移行（okccl）

### 背景

`okccl` 個人アカウントを org に変換する方針だったが、GitHub 側の制約により断念。

- **問題①**: org 変換後に同名（okccl）の個人アカウントを新規作成できない（メールアドレスが使用中になる）
- **問題②**: 既存リポジトリと同名のリポジトリは新 org で再作成できない（GitHub Actions の artifact 名衝突）  
  → 影響リポジトリ: platform-infra / platform-gitops / platform-charts / sample-backend / platform-docs の5本

**解決策**: 完全新規の org `okccl` を作成し、既存リポジトリを Transfer。

### 実施手順

1. `okccl` org 作成
2. 全リポジトリを Transfer（okccl → okccl）
3. SSH config 設定（`~/.ssh/config`）
4. git remote を SSH に更新（全リポジトリ）
5. ArgoCD リポジトリ登録を更新（`argocd repo add/rm`・`argocd app set`）
6. Backstage GitHub App を okccl org に移管（Transfer ownership → Install on org）
7. ghcr.io イメージを okccl に retag・push
8. gh CLI ログイン確認
9. okccl org プロフィール（`.github` リポジトリ）作成

### 遭遇した問題

| 問題 | 原因 | 対処 |
|---|---|---|
| org 変換時に okccl が使用中と表示 | 個人アカウントが残っているため | okccl アカウントのまま okccl を新規作成する方針に変更 |
| ArgoCD Application が旧 repoURL のまま | app set だけでは Application オブジェクトが更新されない | 旧リポジトリを一時再登録 → sync → app set で新 URL に変更 → 旧リポジトリ削除 |
| GitHub App が okccl にインストールできない | "Only on this account" 設定 | App の Transfer ownership で okccl org に移管後にインストール |
| ghcr.io push 権限なし | `gh auth token` の write:packages スコープなし | 別ターミナルで PAT を使って docker login |

---

## 2. Backstage: Keycloak ログイン修正

### 問題

```
Login failed; caused by Error: Failed to sign-in, unable to resolve user identity
```

OIDC リゾルバー `emailMatchingUserEntityProfileEmail` が `platform-admin@platform.local` の
User エンティティを catalog に要求するが、存在しなかった。

### 原因

`backstage/org.yaml` の `platform-admin` User エンティティが以前のコミットで誤って削除されていた。

### 対処

`~/platform-gitops/backstage/org.yaml` に再追加:

```yaml
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: platform-admin
spec:
  profile:
    email: platform-admin@platform.local
  memberOf:
    - platform-team
```

---

## 3. Backstage: ExternalSecret OutOfSync 修正

### 問題

ArgoCD で backstage Application が OutOfSync。`backstage-secret` ExternalSecret の
`ignoreDifferences` が index 0-2 のみカバーしており、9エントリある data 全体をカバーしていなかった。

### 対処

`~/platform-gitops/platform/applications/root-3-others/backstage.yaml` の `ignoreDifferences` を index 0-8 まで拡張。

---

## 4. Backstage: GitHub PR プラグイン修正

### 問題

```
Error: NotImplementedError
```

`githubPullRequestsApiRef` が未登録だった。

### 対処

`~/backstage/packages/app/src/apis.ts` に `GithubPullRequestsClient` の `createApiFactory` を追加:

```typescript
import {
  githubPullRequestsApiRef,
  GithubPullRequestsClient,
} from '@roadiehq/backstage-plugin-github-pull-requests';

createApiFactory({
  api: githubPullRequestsApiRef,
  deps: { configApi: configApiRef, scmAuthApi: scmAuthApiRef },
  factory: ({ configApi, scmAuthApi }) =>
    new GithubPullRequestsClient({ configApi, scmAuthApi }),
}),
```

---

## 5. Backstage: CSP 修正

### 問題

GitHub PR プラグインがブラウザから `api.github.com` に直接リクエストするが CSP でブロックされた。

### 対処

`~/platform-gitops/platform/backstage/values.yaml` に `connect-src` を追加:

```yaml
csp:
  upgrade-insecure-requests: false
  connect-src: ["'self'", 'https://api.github.com']
```

---

## 6. Backstage: カタログ登録

`~/platform-gitops/backstage/catalog/platform-components.yaml` を新規作成。
プラットフォーム構成コンポーネント 20 種を Component として登録。

| カテゴリ | 登録コンポーネント |
|---|---|
| GitOps & Config | argocd / external-secrets / crossplane |
| Networking | cert-manager / envoy-gateway / cilium |
| Identity | keycloak / backstage |
| Observability | grafana / prometheus / loki / tempo / alloy |
| Policy & Security | kyverno / trivy |
| Data | cnpg |
| Autoscaling | keda / vpa / goldilocks / argo-rollouts |

各コンポーネント: `owner: group:platform-team`、`system: platform`、主要 UI には `metadata.links` を付与。

`values.yaml` の `catalog.locations` にエントリを追加して Backstage に取り込み。

---

## 7. Backstage: TechDocs セットアップ

### Dockerfile 変更

`~/backstage/packages/backend/Dockerfile` に mkdocs インストールを追加:

```dockerfile
RUN --mount=... apt-get install -y python3 python3-pip g++ build-essential ...
RUN pip3 install mkdocs-techdocs-core --break-system-packages
```

### values.yaml 変更

```yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'local'
```

### platform-docs リポジトリ変更

- `mkdocs.yml` を新規作成
- `catalog-info.yaml` に `backstage.io/techdocs-ref: dir:.` を追加
- `adr/` `runbook/` `worklog/` を `docs/` 配下に移動（MkDocs の制約: `mkdocs.yml` と同ディレクトリを `docs_dir` にできない）
- `docs/index.md` を新規作成

### 遭遇した問題

| 問題 | 原因 | 対処 |
|---|---|---|
| `docs_dir` エラー | `mkdocs.yml` と同ディレクトリを `docs_dir` にできない | `docs/` サブディレクトリを作成して移動 |
| symlink が解決できない | TechDocs が内部 temp dir にコピーする際にシンボリックリンクが切れる | `git mv` で実ファイルを `docs/` に移動 |

---

## 8. 全リポジトリ README リンク修正

org 移行に伴い全リポジトリの README の `okccl` → `okccl` 参照を修正。

| リポジトリ | 修正内容 |
|---|---|
| okccl/.github | org プロフィール README の全リポジトリリンク |
| platform-docs | ADR/Runbook/Worklog リンクのパス修正（docs/ 配下移動対応）・org URL 修正 |
| sample-backend | GitHub URL・ghcr.io パス修正 |
| platform-charts | GitHub URL 修正 |
| sample-frontend | GitHub URL・ghcr.io パス修正 |

---

## 9. Backstage イメージ再ビルド・push

```bash
cd ~/backstage
mise exec -- yarn build:all
docker build ~/backstage -f packages/backend/Dockerfile --tag ghcr.io/okccl/backstage:latest
docker push ghcr.io/okccl/backstage:latest
kubectl rollout restart deployment/backstage -n backstage
```

---

## 10. sample-app: ghcr-pull-secret 暫定対応

### 問題

`sample-app` Namespace に `ghcr-pull-secret` が存在せず、Pod が ImagePullBackOff。

### 背景

`platform/secrets/external-secrets/namespace-labels.yaml`（ArgoCD 管理外）に NS ラベル定義はあったが、
対応する Kyverno generate ポリシーが未実装だった。

### 暫定対応

`~/platform-gitops/apps/sample-backend/manifests/ghcr-pull-secret.yaml` に ExternalSecret を追加。
`platform-secrets/ghcr-pat` を `kubernetes-store` 経由で参照し `kubernetes.io/dockerconfigjson` 型 Secret を生成。

**本対応**: Golden Path 実装時に Kyverno ClusterPolicy（generate）で自動配布に移行。

---

## 11. sample-backend: GitHub Actions 修正

### 問題①: パッケージ新規作成権限エラー

```
denied: installation not allowed to Create organization package
```

個人アカウント（okccl）時代は暗黙で付与されていた `packages: write` が、
org（okccl）では明示的な指定が必要になった。

**対処**: `~/sample-backend/.github/workflows/build.yaml` に `permissions` ブロックを追加:

```yaml
permissions:
  contents: read
  packages: write
```

### 問題②: GITOPS_TOKEN が 401

```
curl: (22) The requested URL returned error: 401
```

`GITOPS_TOKEN` が org 移行前に発行された fine-grained PAT で、
`okccl/platform-gitops` へのアクセス権がない。

**暫定**: 未対応（後回し）。  
**本対応**: Golden Path 実装時に org レベル Secret + GitHub App トークン（`tibdex/github-app-token`）に移行。

---

## 関連ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `backstage/packages/app/src/apis.ts` | GithubPullRequestsClient の apiFactory 追加 |
| `backstage/packages/backend/Dockerfile` | python3-pip・mkdocs-techdocs-core 追加 |
| `platform-gitops/backstage/org.yaml` | platform-admin User エンティティ再追加 |
| `platform-gitops/backstage/catalog/platform-components.yaml` | 新規（20 Component 登録） |
| `platform-gitops/platform/backstage/values.yaml` | CSP connect-src・techdocs・catalog location 追加 |
| `platform-gitops/platform/applications/root-3-others/backstage.yaml` | ignoreDifferences を index 0-8 に拡張 |
| `platform-gitops/apps/sample-backend/manifests/ghcr-pull-secret.yaml` | 新規（ExternalSecret 暫定対応） |
| `platform-docs/mkdocs.yml` | 新規 |
| `platform-docs/catalog-info.yaml` | techdocs-ref アノテーション追加 |
| `platform-docs/docs/` | 新規（adr/ runbook/ worklog/ を移動・index.md 追加） |
| `sample-backend/.github/workflows/build.yaml` | permissions: packages: write 追加 |

---

## 未完了・次セッションの作業

1. **GITOPS_TOKEN 更新**（優先）: `okccl/platform-gitops` に対して `contents: write` を持つ PAT を新規作成し、sample-backend・sample-frontend の Actions Secrets を更新。CI/CD フロー全体（build → dispatch → values.yaml 更新 → ArgoCD sync）の動作確認
2. **sample-frontend の同様修正**: build.yaml への `permissions: packages: write` 追加と GITOPS_TOKEN 更新
3. **Golden Path（Backstage Software Template）**: Kyverno generate ポリシー・org レベル Secret・GitHub App トークンを含む本格実装
