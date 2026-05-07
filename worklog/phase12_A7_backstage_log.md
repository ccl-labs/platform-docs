# Phase 12-A7 作業ログ（Backstage Platform Team活用）

## 概要

Backstage を PE 向けのポータルとして活用するための実装。ホームページ（ArgoCD 異常検知・GitHub PR カード・クイックリンク）、GitHub App/OAuth App 統合、ArgoCD プロキシ設定を実装。

---

## 1. PE 向けホームページの実装

### ArgoCDWidget.tsx（新規）

`~/backstage/packages/app/src/components/home/ArgoCDWidget.tsx`

- ArgoCD API `/api/proxy/argocd/api/v1/applications` を呼び出し
- `syncStatus !== 'Synced' || healthStatus !== 'Healthy'` のアプリのみ表示
- `useApi(fetchApiRef)` を使用

### HomePage.tsx（新規）

`~/backstage/packages/app/src/components/home/HomePage.tsx`

- `@roadiehq/backstage-plugin-github-pull-requests` の `HomePageRequestedReviewsCard` / `HomePageYourOpenPullRequestsCard`
- `ArgoCDWidget`、`HomePageToolkit`（クイックリンク）を Grid レイアウトで配置
- クイックリンク: ArgoCD / Keycloak / Grafana / Prometheus / Loki / MinIO

### App.tsx・Root.tsx の変更

- デフォルトルートを `/catalog` → `/home` に変更
- `<Route path="/home" element={<HomePage />} />` を追加
- サイドバーの Home リンクを `home` に変更

---

## 2. GitHub 統合（2-App 構成）

### 背景

GitHub App のユーザートークンは OAuth スコープ（`repo`, `read:org`）を付与できないため、
PR カードプラグイン（`@roadiehq/backstage-plugin-github-pull-requests`）がサインインに使えない（Backstage issue #9163）。
API 連携と ユーザー認証を別の App に分離する 2-App 構成を採用。

| 用途 | App |
|------|-----|
| API 連携・Scaffolder | GitHub App `okccl-platform-backstage` |
| ユーザーサインイン・PR カード | GitHub OAuth App `okccl-platform-backstage-oauth` |

### values.yaml の変更

`~/platform-gitops/platform/backstage/values.yaml`

```yaml
integrations:
  github:
    - host: github.com
      apps:
        - appId: ${GITHUB_APP_ID}
          clientId: ${GITHUB_APP_CLIENT_ID}
          clientSecret: ${GITHUB_APP_CLIENT_SECRET}
          privateKey: ${GITHUB_APP_PRIVATE_KEY}

auth:
  providers:
    github:
      development:
        clientId: ${GITHUB_OAUTH_CLIENT_ID}
        clientSecret: ${GITHUB_OAUTH_CLIENT_SECRET}
        signIn:
          resolvers:
            - resolver: usernameMatchingUserEntityName
```

### external-secret.yaml の変更

`~/platform-gitops/platform/backstage/app-config/external-secret.yaml`

- `GITHUB_TOKEN` 削除
- `GITHUB_APP_ID` / `GITHUB_APP_CLIENT_ID` / `GITHUB_APP_CLIENT_SECRET` / `GITHUB_APP_PRIVATE_KEY` 追加（`backstage-github-app-source` から参照）
- `GITHUB_OAUTH_CLIENT_ID` / `GITHUB_OAUTH_CLIENT_SECRET` 追加（同上）

### SOPS シークレット追加

`backstage-github-app-source` に以下のキーを `sops edit` で追加:

| キー | 内容 |
|------|------|
| `appId` | GitHub App ID |
| `clientId` | GitHub App Client ID |
| `clientSecret` | GitHub App Client Secret |
| `privateKey` | GitHub App Private Key（PEM形式） |
| `githubOauthClientId` | GitHub OAuth App Client ID |
| `githubOauthClientSecret` | GitHub OAuth App Client Secret |

---

## 3. ArgoCD プロキシ設定

### values.yaml の変更

```yaml
proxy:
  endpoints:
    '/argocd':
      target: https://argocd.platform.local
      changeOrigin: true
      secure: false
      headers:
        Authorization: Bearer ${ARGOCD_TOKEN}
```

> **注意**: endpoint key は `/argocd`（`/argocd/api` にすると Backstage がパスストリップして `/api` が欠落する）

### ArgoCD ローカルアカウント追加

`~/platform-gitops/platform/argocd/values.yaml`

```yaml
configs:
  cm:
    accounts.backstage: apiKey
  rbac:
    policy.csv: |
      p, role:backstage-reader, applications, get, */*, allow
      g, backstage, role:backstage-reader
```

### ArgoCD トークン発行

```bash
argocd account generate-token --account backstage
```

発行したトークンを `backstage-secret-source` の `ARGOCD_TOKEN` に `sops edit` で追加。

---

## 4. GitHub サインインのデバッグ

### 問題①: Authorize ボタンのグレーアウト

- ブラウザキャッシュによるポップアップ制限が原因
- ブラウザ再起動で解消

### 問題②: User entity not found

ログ:
```
GET /api/catalog/entities/by-name/User/default/okccl 404
```

`usernameMatchingUserEntityName` リゾルバーが catalog の User エンティティを必要とするが、
`org.yaml` に `okccl` が定義されていなかった。

**対応**: `~/platform-gitops/backstage/org.yaml` に User・Group を追加

```yaml
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: okccl
spec:
  profile:
    displayName: okccl
    email: cclina12@gmail.com
  memberOf:
    - platform-team
---
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: platform-team
spec:
  type: team
  members: [okccl]
---
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: app-developer
spec:
  type: team
  members: []
```

---

## 5. GitHub Org Provider の検討と見送り

複数人環境に備えて `@backstage/plugin-catalog-backend-module-github-org` の導入を検討。
GitHub Organization の org メンバーを User/Group として自動同期できる機能。

**見送り理由**: `okccl` は個人アカウントであり GitHub Organization が存在しない。
`okccl` という名前の org は個人アカウントと同名のため作成不可。

**将来の移行計画**:
- `okccl` 個人アカウントを GitHub Organization に変換（アプローチ B）
  - URL が変わらないため既存設定への影響が最小
  - org 変換後に GitHub Org Provider を有効化して org.yaml 手動管理を廃止

---

## 6. セキュリティクリーンアップ

`~/platform-gitops/secrets/templates/` に平文 PAT を含むファイルが存在（git 管理外）。

削除したファイル:
- `backstage-github-token-source.yaml`（旧 PAT、移行済みで不要）
- `ghcr-pat.yaml`（`platform/secrets/sources/ghcr-pat.yaml` に SOPS 暗号化済みの正式版あり）

---

## 7. Backstage ビルド・push

```bash
cd ~/backstage
mise exec -- yarn build:all
docker build . -f packages/backend/Dockerfile --tag ghcr.io/okccl/backstage:latest
docker push ghcr.io/okccl/backstage:latest
```

---

## 関連ファイル一覧

| ファイル | 変更内容 |
|---------|---------|
| `backstage/packages/app/src/components/home/ArgoCDWidget.tsx` | 新規（ArgoCD 異常検知ウィジェット） |
| `backstage/packages/app/src/components/home/HomePage.tsx` | 新規（PE向けホームページ） |
| `backstage/packages/app/src/App.tsx` | デフォルトルート → /home、HomePage ルート追加 |
| `backstage/packages/app/src/components/Root/Root.tsx` | サイドバー Home リンク修正 |
| `backstage/packages/backend/src/index.ts` | GitHub auth provider module 追加 |
| `backstage/.mise.toml` | 新規（node = "24"） |
| `platform-gitops/platform/backstage/values.yaml` | GitHub App/OAuth App 統合、ArgoCD プロキシ、extraEnvVars 追加 |
| `platform-gitops/platform/backstage/app-config/external-secret.yaml` | GITHUB_TOKEN → App/OAuth credentials に更新 |
| `platform-gitops/platform/secrets/sources/secret-generator.yaml` | backstage-github-token-source → backstage-github-app-source に変更 |
| `platform-gitops/platform/argocd/values.yaml` | backstage アカウント追加、RBAC 追加 |
| `platform-gitops/backstage/org.yaml` | okccl User、platform-team/app-developer Group 追加 |

---

## 未完了・次セッションの作業

**前提: GitHub org 移行完了後**

1. `kubectl rollout restart deployment/backstage -n backstage` → GitHub サインイン・PR カード動作確認
2. gh CLI を新個人アカウントで `gh auth login`
3. git remote を HTTPS → SSH に切り替え
4. `ghcr-pat` SOPS ファイルを新個人アカウントの PAT に更新
5. catalog 充実（Component 登録）
6. TechDocs セットアップ
7. ADR 記録（Permission Framework 不採用）
