# Phase 12 作業ログ（セッション A6）

## 実施日
2026-05-06

---

## 概要

- **S-1**: minio-backup-secret の Push 型 → Pull 型移行（ESO ExternalSecret）
- **F-2**: Backstage に Keycloak OIDC ログインボタンを追加。カスタムイメージのビルド・デプロイから OIDC ログイン完全動作まで。

---

## S-1: minio-backup-secret の Pull 型移行

### 背景
bootstrap 時に `sops decrypt | kubectl apply` で直接投入していた minio-backup-secret を、ESO ExternalSecret 経由の Pull 型に移行。

### 実施内容

#### 1. SOPS 暗号化ソースシークレット作成
**`platform-gitops/platform/secrets/sources/minio-backup-secret-source.yaml`** を新規作成（namespace: platform-secrets）:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-backup-secret-source
  namespace: platform-secrets
stringData:
  ACCESS_KEY_ID: <value>
  ACCESS_SECRET_KEY: <value>
```
SOPS 暗号化:
```bash
cp minio-backup-secret-source.yaml platform-gitops/platform/secrets/sources/
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops --encrypt --in-place \
  platform-gitops/platform/secrets/sources/minio-backup-secret-source.yaml
```
> ※ `path_regex` はファイルパスで一致するため、`--in-place` でターゲットパスに直接暗号化する必要がある（tmpファイル経由だとマッチしない）。

#### 2. ksops ジェネレーターに追加
**`platform-gitops/platform/secrets/sources/secret-generator.yaml`** に `./minio-backup-secret-source.yaml` を追記。

#### 3. ExternalSecret 作成
**`platform-gitops/apps/sample-backend/manifests/minio-backup-secret.yaml`**:
```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: minio-backup-secret
  namespace: sample-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: kubernetes-store
    kind: ClusterSecretStore
  target:
    name: minio-backup-secret
  data:
    - secretKey: ACCESS_KEY_ID
      remoteRef:
        key: minio-backup-secret-source
        property: ACCESS_KEY_ID
    - secretKey: ACCESS_SECRET_KEY
      remoteRef:
        key: minio-backup-secret-source
        property: ACCESS_SECRET_KEY
```

#### 4. Makefile から Push 型ステップを削除
**`platform-infra/k3d/Makefile`** の `bootstrap-apps` ターゲットから以下を削除:
- `sample-app` namespace の wait
- `sops decrypt | kubectl apply` による minio-backup-secret の直接投入

---

## 1. notifications/signals 削除（白画面クラッシュ修正）

### 原因
`@backstage/plugin-notifications` の `NotificationsSidebarItem` が legacy frontend (`createApp` from `@backstage/app-defaults`) では動作しない `apiRef{core.toast}` を使用しており、React ツリー全体がクラッシュしていた。

### 修正ファイル

**`packages/app/src/components/Root/Root.tsx`**
- `import { NotificationsSidebarItem } from '@backstage/plugin-notifications'` を削除
- `<NotificationsSidebarItem />` と前後の `<SidebarDivider />` を削除

**`packages/app/src/App.tsx`**
- `import { NotificationsPage } from '@backstage/plugin-notifications'` を削除
- `import { SignalsDisplay } from '@backstage/plugin-signals'` を削除
- `/notifications` ルートを削除
- `<SignalsDisplay />` を削除
- SignInPage の `auto` prop を削除（サインインページ選択画面を表示するため）

---

## 2. oidcAuthApiRef 未定義エラーの修正

### 原因
`oidcAuthApiRef` をインポートしていた `@backstage/core-plugin-api` のインストール済みバージョンにはこのエクスポートが存在せず、`undefined` になっていた。`useApi(undefined)` → `undefined.id` でクラッシュ。

### 修正

**`packages/app/src/apis.ts`**
```ts
import { OAuth2 } from '@backstage/core-app-api';
import { createApiRef, discoveryApiRef, oauthRequestApiRef, ... } from '@backstage/core-plugin-api';

export const oidcAuthApiRef = createApiRef<OAuthApi & OpenIdConnectApi & ProfileInfoApi & BackstageIdentityApi & SessionApi>({
  id: 'auth.oidc',
});

// apis 配列に追加
createApiFactory({
  api: oidcAuthApiRef,
  deps: { discoveryApi: discoveryApiRef, oauthRequestApi: oauthRequestApiRef },
  factory: ({ discoveryApi, oauthRequestApi }) =>
    OAuth2.create({
      discoveryApi,
      oauthRequestApi,
      provider: { id: 'oidc', title: 'Keycloak', icon: () => null },
      environment: 'development',
      defaultScopes: ['openid', 'profile', 'email'],
    }),
}),
```

**`packages/app/src/App.tsx`**
```ts
// 変更前
import { oidcAuthApiRef } from '@backstage/core-plugin-api';
// 変更後
import { oidcAuthApiRef } from './apis';
```

---

## 3. OIDC backend モジュール追加

### 原因
`Unknown auth provider 'oidc'` エラー。backend に OIDC プロバイダーモジュールが未登録。

### 修正

```bash
yarn workspace backend add @backstage/plugin-auth-backend-module-oidc-provider
```

**`packages/backend/src/index.ts`**
```ts
backend.add(import('@backstage/plugin-auth-backend-module-oidc-provider'));
```

---

## 4. Keycloak metadataUrl の修正（values.yaml）

複数の試行を経て正しい URL を確定。

| 試行 | URL | 結果 |
|------|-----|------|
| 1 | `http://keycloak.platform.local/auth/realms/platform/...` | 301 (HTTP→HTTPS リダイレクト) |
| 2 | `https://keycloak.platform.local/realms/platform/...` | 404 (パスが存在しない) |
| 3 | `https://keycloak.platform.local/auth/realms/platform/...` | 200 OK ✓ |

### 診断コマンド（Pod 内から実行）
```bash
kubectl exec -n backstage deployment/backstage -- node -e "
const https = require('https');
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0';
https.get('https://keycloak.platform.local/auth/realms/platform/.well-known/openid-configuration', (res) => {
  console.log('status:', res.statusCode);
}).on('error', e => console.error(e.message));
"
```

**`platform-gitops/platform/backstage/values.yaml`** の最終 metadataUrl:
```yaml
metadataUrl: https://keycloak.platform.local/auth/realms/platform/.well-known/openid-configuration
```

---

## 5. 自己署名証明書対応

### 原因
`UNABLE_TO_VERIFY_LEAF_SIGNATURE` — Keycloak が自己署名証明書を使用。

### 修正（ローカル開発環境限定）

**`platform-gitops/platform/backstage/values.yaml`**
```yaml
extraEnvVars:
  - name: NODE_TLS_REJECT_UNAUTHORIZED
    value: "0"
```

> 本番化時は `NODE_EXTRA_CA_CERTS` で CA 証明書を信頼させる正式対応が必要。

---

## 6. セッションサポートの有効化

### 原因
`Authentication failed, authentication requires session support` — `auth.session.secret` が未設定のため session middleware が初期化されなかった。

### 調査
`/app/node_modules/@backstage/plugin-auth-backend/dist/service/router.cjs.js` を確認:
```js
const secret = config.getOptionalString("auth.session.secret");
if (secret) {
  // session middleware + KnexSessionStore(PostgreSQL) を初期化
}
```

`backend.auth.keys` とは別の設定キーであることを確認。

### 修正

**`platform-gitops/platform/backstage/values.yaml`**
```yaml
auth:
  session:
    secret: ${BACKEND_SECRET}
  environment: development
  providers:
    ...
```

---

## 7. User エンティティの追加（サインインリゾルバー対応）

### 原因
`Failed to sign-in, unable to resolve user identity` — サインインリゾルバー `emailMatchingUserEntityProfileEmail` がカタログ内に対応する User エンティティを見つけられなかった。

### 修正

**`platform-gitops/backstage/org.yaml`** を新規作成:
```yaml
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: platform-admin
spec:
  profile:
    email: platform-admin@platform.local
  memberOf: []
```

**`platform-gitops/platform/backstage/values.yaml`** にカタログロケーション追加:
```yaml
- type: url
  target: https://github.com/okccl/platform-gitops/blob/main/backstage/org.yaml
  rules:
    - allow: [User, Group]
```

---

## 8. イメージビルド・デプロイ（合計3回）

```bash
# 作業ディレクトリ: ~/backstage
export PATH="$HOME/.local/share/mise/installs/node/24/bin:$PATH"
yarn build:all
docker build -t ghcr.io/okccl/backstage:latest -f packages/backend/Dockerfile .
docker push ghcr.io/okccl/backstage:latest
kubectl rollout restart deployment -n backstage
kubectl rollout status deployment -n backstage --timeout=120s
```

---

## 最終状態

- Backstage トップページ表示: ✓
- Guest ログイン: ✓
- Keycloak OIDC ログイン: ✓ (`platform-admin@platform.local`)
- ArgoCD sync: 全グリーン

---

## 関連ファイル一覧

| ファイル | 変更内容 |
|---------|---------|
| `~/backstage/packages/app/src/App.tsx` | notifications/signals 削除、oidcAuthApiRef インポート変更 |
| `~/backstage/packages/app/src/apis.ts` | oidcAuthApiRef 定義・OAuth2 ファクトリ登録 |
| `~/backstage/packages/app/src/components/Root/Root.tsx` | NotificationsSideBarItem 削除 |
| `~/backstage/packages/backend/src/index.ts` | plugin-auth-backend-module-oidc-provider 追加 |
| `platform-gitops/platform/backstage/values.yaml` | metadataUrl・NODE_TLS_REJECT_UNAUTHORIZED・auth.session.secret・org.yaml ロケーション追加 |
| `platform-gitops/backstage/org.yaml` | 新規作成（platform-admin User エンティティ） |
