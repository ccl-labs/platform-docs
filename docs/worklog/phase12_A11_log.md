# Phase12-A11 作業ログ

| # | 作業内容 |
|---|---|
| 1 | Backstage Software Template 設計 |
| 2 | DOCS_RULE.md に設計ドキュメントルールを追加 |

---

## 1. Backstage Software Template 設計

設計フェーズのみ実施。実装は次セッション以降。設計ドキュメントは `~/internal/claude/tmp_backstage_software_template.md` に出力済み。

### 設計決定事項

**テンプレート構成**

| 項目 | 決定内容 |
|---|---|
| テンプレート種類 | backend（FastAPI）/ frontend（React + Vite）の2種 |
| DB オプション | backend のみ、チェックボックスで選択 |
| Scaffolderの操作スコープ | リポジトリ作成 + platform-gitops PR 作成 |
| Namespace | アプリごとに別 Namespace（labels: policy-enforced / ghcr-pull-secret / goldilocks） |
| Helm Chart | platform-charts に `generic-backend` / `generic-frontend` を事前作成 |
| ArgoCD Project | アプリごとに個別 Project（clusterResourceWhitelist に Namespace を許可） |
| カタログ登録 | Scaffolderの `catalog:register` アクションで Backstage DB に直接登録 |

**Scaffolderの実行フロー**

```
1. fetch:template → アプリリポジトリ用スケルトン生成
2. publish:github → ccl-labs/<app-name> リポジトリ作成・スケルトン push
3. fetch:template → platform-gitops 用ファイル生成（リポジトリ URL を変数として使用）
4. publish:github:pull-request → platform-gitops に PR 作成
5. catalog:register → Backstage カタログ登録
```

**platform-gitops PRに含むファイル**

```
platform/applications/root-3-others/<app-name>-project.yaml  # ArgoCD Project
platform/applications/root-3-others/<app-name>.yaml          # ArgoCD Application
apps/<app-name>/values.yaml
apps/<app-name>/manifests/httproute.yaml
```

HTTPRoute のホスト名は `<app-name>.platform.local` で自動生成。

**設計上の重要確認事項**

- `clusterResourceWhitelist` に Namespace を許可する理由: `CreateNamespace=true` を使うため。各 Project の destination が特定 Namespace のみに限定されるのでリスクは限定的。`prune: true` との組み合わせで Application 削除時に Namespace ごと消える動作は許容（app-developer は delete 権限を持たない）。
- `common-db` に `db.enabled` フラグを追加する理由: 現状 `sample-backend/templates/all.yaml` が enabled フラグなしで `common-db.cluster` を常に呼んでいるため、DB オプション化のために改修が必要。sample-backend も Golden Path 産アプリと同列にするため例外対応はせず共通化する。
- `fetch:template` の複数回使用: 公式仕様として可能。`targetPath` を分けることで異なるディレクトリに生成できる。

---

## 2. DOCS_RULE.md に設計ドキュメントルールを追加

`~/internal/claude/DOCS_RULE.md` に設計ドキュメントのセクションを新設。

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `~/internal/claude/tmp_backstage_software_template.md` | 設計ドキュメント（新規作成） |
| `~/internal/claude/DOCS_RULE.md` | 設計ドキュメントのルールを追加 |
