# Phase12-A11 作業ログ

| # | 作業内容 |
|---|---|
| 1 | Backstage Software Template 設計 |
| 2 | DOCS_RULE.md に設計ドキュメントルールを追加 |
| 3 | Backstage Software Template 実装 |
| 4 | テスト前の設計見直し（fullstack テンプレートへの再設計） |
| 5 | Namespace ラベル設計の見直し |
| 6 | 設計ドキュメント修正（自己レビュー対応） |
| 7 | fullstack テンプレート実装 |
| 8 | Scaffolderテスト・バグ修正 |
| 9 | GHCR push 権限問題の調査・修正（続き） |
| 10 | platform-gitops push 方式の検討 |
| 11 | ArgoCD Application 配置の設計見直し・user-apps App-of-Apps 実装 |
| 12 | Scaffolderテスト再実行・追加バグ修正 |
| 13 | platform-gitops auto-merge ワークフロー実装 |
| 14 | common-app chart の db.enabled バグ修正 |
| 15 | Scaffolder エンドツーエンドテスト完了・設計ドキュメント整理 |

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

---

## 3. Backstage Software Template 実装

設計ドキュメントに従い、以下の順序で実装。

**① common-db に db.enabled フラグを追加**

library chart（`common-db`）の `values.yaml` デフォルト値は親 chart に引き継がれないため、`sample-backend/values.yaml` にも `db.enabled: true` を追加する必要があった。`_helpers.tpl` の `common-db.cluster` を `{{- if .Values.db.enabled }}` で囲み、`scheduledBackup` の条件を `and db.enabled backup.enabled` に変更。`helm template` で enabled/disabled 両ケースを確認済み。

**② generic-backend / generic-frontend chart を新規作成**

`sample-backend` / `sample-frontend` と同じ wrapper chart 構成で作成。`generic-frontend/values.yaml` に `db.name: ""` が必要（`common-app.deployment` が `.Values.db.name` を参照するため）。実装時にエラーで判明し追加。

**③ backend / frontend テンプレートを実装**

`backstage/templates/backend/` と `backstage/templates/frontend/` に template.yaml + app-skeleton + gitops-skeleton を作成。

実装時に判明した設計との差異:

- app-skeleton のワークフローは `build.yaml` 1本で完結（`update-gitops.yaml` は platform-gitops 側に既存のため app 側には不要）
- `publish:github:pull-request` は `sourcePath: ./gitops` + `targetPath: .` の両方を指定する必要がある（`sourcePath` 未指定だとワークスペース全体が対象になる）
- `catalog:register` は `publish:github` の出力 `repoContentsUrl` + `catalogInfoPath` を使う（URL を手動構築する必要はなかった）
- `build.yaml` 内で Backstage テンプレート変数と GitHub Actions 変数が混在するため、GitHub Actions 変数は `${{ "{{" }} github.sha {{ "}}" }}` 形式でエスケープが必要

**④ platform/backstage/values.yaml に template URL を追加・Backstage 動作確認**

ArgoCD sync 後に Backstage hard refresh。Scaffolderにテンプレートが2件表示されることを確認。

---

## 4. テスト前の設計見直し（fullstack テンプレートへの再設計）

Scaffolderのテスト実行前に Namespace 設計の問題が判明。現在の設計では backend/frontend がそれぞれ独立した Namespace を持つが、同一アプリの front/back は常にセットで作成するため同一 Namespace に同居させる構成が自然。

### 設計変更の決定事項

| 項目 | 変更前 | 変更後 |
|---|---|---|
| テンプレート構成 | backend + frontend の2テンプレート | fullstack 1テンプレート |
| リポジトリ命名 | `<appName>` | `<appName>-backend` / `<appName>-frontend` |
| Namespace | テンプレートごとに独立 | 1アプリ = 1 Namespace（`<appName>`） |
| ArgoCD Application | 1本 | 2本（`<appName>-backend` / `<appName>-frontend`） |
| HTTPRoute | 各アプリに独立したホスト | 同一ホストでパス分割（`/api` → backend リライト、`/` → frontend） |

backend-only が必要な場合は別途テンプレートを作成する方針。

### 次セッションで判断が必要な事項

- **`managedNamespaceMetadata` の扱い**: backend/frontend 2つの Application が同一 Namespace を管理する場合、どちらが Namespace ラベルを持つか。frontend が先に sync した場合に `ghcr-pull-secret` ラベルなし Namespace が一時的に作られ Pod 起動が失敗する可能性がある。sync-wave での順序制御も選択肢。
- **同一ホスト・複数 HTTPRoute の挙動**: Envoy Gateway での `/api` + `/` の分割ルーティングは仕様上可能だが実装で確認が必要。

設計ドキュメントは fullstack 設計に更新済み。既存の backend/frontend テンプレートは次セッションで削除・置き換え予定。

---

---

## 5. Namespace ラベル設計の見直し

「アプリ開発者が使用する NS に `policy-enforced: "true"` ラベルを自動付与して Kyverno 監視対象にする」という要件に対して、`managedNamespaceMetadata` 単体では ordering の保証ができないことが判明。

**問題の構造:**
- 2つの ArgoCD Application（backend/frontend）が同一 Namespace を `CreateNamespace=true` で管理する場合、frontend が先に sync すると `policy-enforced: "true"` ラベルなしの Namespace が一時作成される
- Kyverno は admission webhook のため Pod 作成時点でラベルがないと監視対象外になり、ポリシー違反を検知できない
- `selfHeal: true` で後から修復されるが、その間に不正な Pod が乗る可能性がある

**解決策: `namespace.yaml` を gitops-skeleton に明示配置 + App-of-Apps sync-wave**

- `apps/<appName>-backend/manifests/namespace.yaml` にラベル付きの Namespace リソースを明示管理
- backend Application に `sync-wave: "1"`、frontend Application に `sync-wave: "2"` を付与
- backend（Namespace 含む）が Healthy になってから frontend が sync 開始されるため、Pod より先にラベル付き NS が確実に存在する
- platform NS は gitops-skeleton の外にあるため影響なし（Kyverno mutate の「全 NS 対象」問題が不要になる）
- `managedNamespaceMetadata` と `CreateNamespace=true` は不要になり削除

---

## 6. 設計ドキュメント修正（自己レビュー対応）

`~/internal/claude/tmp_backstage_software_template.md` を以下の観点で修正。

| 修正箇所 | 内容 |
|---|---|
| ArgoCD Project `sourceRepos` | app リポジトリ2件を削除（Application が参照しないため不要だった） |
| backend Application | `CreateNamespace=true` / `managedNamespaceMetadata` を削除、sync-wave: "1" を追加、namespace.yaml セクションを新規追加 |
| frontend Application | spec を完全記載（「backend と同構成だが」の省略を解消）、sync-wave: "2" を追加 |
| HTTPRoute（backend/frontend） | `parentRefs`（Gateway `eg` へのアタッチ）を追加（省略されていた） |
| 未確認事項 | `managedNamespaceMetadata` の件を削除、sync-wave 挙動確認・複数 HTTPRoute 挙動確認に更新 |

---

## 7. fullstack テンプレート実装

設計ドキュメントに従い `backstage/templates/fullstack/` を新規作成し、旧 `backend/` / `frontend/` を削除。

**ファイル構成:**
```
backstage/templates/fullstack/
  template.yaml
  backend-skeleton/        # FastAPI スケルトン（src/main.py, Dockerfile, build.yaml, catalog-info.yaml）
  frontend-skeleton/       # React + Vite スケルトン（src/, index.html, package.json, nginx.conf, Dockerfile, build.yaml, catalog-info.yaml）
  gitops-skeleton/
    platform/applications/root-3-others/<appName>-project.yaml   # sync-wave なし
    platform/applications/root-3-others/<appName>-backend.yaml   # sync-wave: "1"
    platform/applications/root-3-others/<appName>-frontend.yaml  # sync-wave: "2"
    apps/<appName>-backend/values.yaml
    apps/<appName>-backend/manifests/namespace.yaml              # ラベル付き NS（新規）
    apps/<appName>-backend/manifests/httproute.yaml              # /api → URLRewrite
    apps/<appName>-frontend/values.yaml
    apps/<appName>-frontend/manifests/httproute.yaml             # /
```

**Scaffolderフロー（8ステップ）:**
```
1. fetch:template（backend スケルトン）
2. publish:github → <appName>-backend リポジトリ作成
3. fetch:template（frontend スケルトン）
4. publish:github → <appName>-frontend リポジトリ作成
5. fetch:template（gitops-skeleton）
6. publish:github:pull-request → platform-gitops PR 作成（update: true）
7. catalog:register（backend、optional: true）
8. catalog:register（frontend、optional: true）
```

---

## 8. Scaffolderテスト・バグ修正

Backstage から実行テストを行い、複数のエラーを修正。

| エラー | 原因 | 対処 |
|---|---|---|
| リポジトリ作成 403（1回目） | GitHub App に Administration 権限なし | GitHub App に Administration / Contents / Pull requests / Workflows / Packages: R&W を追加 |
| build.yaml 構文エラー | `${{ "{{" }} github.sha {{ "}}" }}` が正しく展開されない（`{{` になり `$` が落ちる） | `${{ "${{" }} github.sha ${{ "}}" }}` に修正（Backstage のデリミタは `${{` のため） |
| PR 作成の重複エラー | 途中失敗で branch が残存 | `publish:github:pull-request` に `update: true` を追加 |
| catalog:register 409 | 途中失敗で登録済みのロケーションが残存 | `catalog:register` に `optional: true` を追加 |
| GHCR push 権限エラー | `GITHUB_TOKEN` では org の GHCR への push 不可 | GitHub App トークン（`owner: ccl-labs`）を生成して GHCR ログインに使用 |
| GitHub App トークン生成失敗 | org variable は無料プランで private repo から参照不可 | リポジトリを public に変更（`repoVisibility: public`） |

---

## 9. GHCR push 権限問題の調査・修正（続き）

前セッションで GitHub App トークンによる GHCR push に切り替えたが、引き続きエラーが発生。

| エラー | 原因 | 対処 |
|---|---|---|
| `installation not allowed to Write organization package` | `ccl-labs-gitops` App のインストール権限に `packages` が未設定 | GitHub App → Organization permissions → Packages: R&W を追加・org 承認 |
| 承認後も同エラー継続 | GitHub App トークンは GHCR org パッケージへの書き込みが構造的に不可（既知の制限、community discussion #57724 等で確認） | GHCR push を `GITHUB_TOKEN`（`packages: write`）に切り替え、GitHub App トークンは dispatch 専用に限定 |
| `permission_denied: write_package` | org の Default workflow permissions が read-only だった | org Settings → Actions → General → Workflow permissions を "Read and write permissions" に変更 |
| frontend のみ引き続き失敗 | org 設定変更前に rerun が開始されていた | 再 rerun で解消 |
| 再削除・再実行後も `write_package` | 前回の失敗 push で GHCR に不完全なパッケージが残存し、Backstage App の所有として登録されていた | `https://github.com/orgs/ccl-labs/packages` から該当パッケージを手動削除 → 再実行で成功 |

**確定した build.yaml 構成:**
- `permissions: packages: write` をジョブに付与
- GHCR ログイン: `GITHUB_TOKEN`（標準推奨方法）
- dispatch: `ccl-labs-gitops` GitHub App トークン（`owner: ccl-labs`）

---

## 10. platform-gitops push 方式の検討

PR マージ前に image dispatch が届き `values.yaml` が存在しないエラーが発生したことをきっかけに、PR なし直接 push の方針を検討。

**`github:repo:push` の試行と失敗:**
- パラメータ名エラー（`branch` → `defaultBranch`、`commitMessage` → `gitCommitMessage`）で1回修正
- 修正後も `not-fast-forward` エラー。原因: このアクションは既存リポジトリへのファイル追加ではなく、新規 git 履歴を push する設計のため既存リポジトリに使えない

**設計議論:**
- dev 環境では PR レビューは不要（ガードレールはクラスタ側で担保）
- `publish:github:pull-request` + auto-merge workflow、またはカスタムアクションが現実的な選択肢
- → 後続の設計見直し（セクション11）により push 先ファイル構成が変わるため、push 方式の確定は保留

---

## 11. ArgoCD Application 配置の設計見直し・user-apps App-of-Apps 実装

**問題の発見:** Scaffolderが `platform/applications/root-3-others/` に Application CR を追加していたが、ここは platform team 管理リソースの置き場であり、開発者アプリが混入すべきでない。

**新しい設計:**

| 変更点 | 変更前 | 変更後 |
|---|---|---|
| Application CR の場所 | `platform/applications/root-3-others/<appName>-*.yaml` | `apps/<appName>-*/application.yaml` |
| ArgoCD Project | アプリごとに個別 Project | 全ユーザーアプリ共通の `user-apps` Project（platform team が一度だけ作成） |
| 検出方式 | root-3-others App-of-Apps が直接管理 | `user-apps` App-of-Apps（`apps/*/application.yaml` を自動検出）が管理 |
| Scaffolderの push 先 | `apps/` + `platform/applications/root-3-others/` | `apps/` 配下のみ |

**実装内容:**
- `platform/applications/root-3-others/user-apps-project.yaml` 新規作成（共有 AppProject）
- `platform/applications/root-3-others/user-apps.yaml` 新規作成（App-of-Apps、`project: default`）
- `gitops-skeleton` を新構成に更新：`platform/` を削除、`application.yaml` を `apps/<appName>-*/` 配下に追加
- 設計ドキュメント（`tmp_backstage_software_template.md`）を新設計に更新

**トラブル:** `user-apps.yaml` の `project: platform` が不正（`platform` プロジェクトは未定義、他アプリは `project: default` を使用）→ `project: default` に修正。

---

## 12. Scaffolderテスト再実行・追加バグ修正

| エラー | 原因 | 対処 |
|---|---|---|
| `instance requires property "description"` | `publish:github:pull-request` の `description` が必須フィールド | `template.yaml` に `description` を追加 |
| output リンクが開けない | `catalog:register` の `output.entityRef` は `component:default/xxx` 形式の文字列で URL ではない | `output.catalogInfoUrl` に変更 |

公式ドキュメント（roadie.io / GitHub ソース）と照合してレビューを実施。上記2点以外は問題なし。

---

## 13. platform-gitops auto-merge ワークフロー実装

**背景:** Scaffolderが作成する `app/<appName>` ブランチの PR が未マージの状態で app リポジトリの build Actions が dispatch を送ると、`apps/<appName>-*/values.yaml` が main にまだ存在せず `sed: No such file or directory` で失敗する。

**実装:** `.github/workflows/auto-merge-app-pr.yaml` を platform-gitops に追加。

- トリガー: `pull_request` (opened / synchronize) で `main` ブランチ向け
- 条件: `github.head_ref` が `app/` で始まる場合のみ実行
- 処理: `gh pr merge --squash --delete-branch`（`GITHUB_TOKEN` で実行）
- `gitops/*` ブランチ（image 更新 PR）は条件不一致で自動 skipped

タイミング的な問題は発生しない理由: auto-merge は PR open 直後（数秒）に実行されるのに対し、build Actions の完了は 30〜40 秒かかるため、dispatch が届く頃には values.yaml が main に存在している。

---

## 14. common-app chart の db.enabled バグ修正

**問題:** `db.enabled: false` にもかかわらず backend pod が `secret "test-app-db-app" not found` で起動失敗。

**原因:** `common-app/templates/_helpers.tpl` の条件分岐が `{{- if .Values.db.name }}` になっており、`db.name` は `withDb: false` でも常にセットされるため DB 環境変数が常に注入されていた。

**修正:** `_helpers.tpl` の2箇所（deployment / rollout）を `{{- if .Values.db.enabled }}` に変更し、`common-app-0.3.0.tgz` を再パッケージして `generic-backend/charts/` に配置。

---

## 15. Scaffolderエンドツーエンドテスト完了・設計ドキュメント整理

**エンドツーエンドテスト結果（ノンストップ実行）:**

- Backstage Scaffolderからアプリ作成 → GitHub リポジトリ作成・build Actions 実行 → platform-gitops PR 自動マージ → ArgoCD が Application を自動検出・sync → frontend pod 起動（OK）・backend pod 起動（OK）まで一連の流れが動作確認済み

**設計ドキュメント整理:**
- `tmp_backstage_software_template.md` を最終実装と照合・修正（push 方式確定、`project: default`、auto-merge ワークフロー追記、既知注意事項追加）→ `done/` に移動
- `tmp_backstage_teardown_template.md` を新規作成（削除フロー設計、実装は今後）
- `PROJECT.md` を Phase12-A11 完了として更新

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `~/internal/claude/tmp_backstage_software_template.md` | 設計ドキュメント（新規作成・fullstack 設計に更新） |
| `~/internal/claude/DOCS_RULE.md` | 設計ドキュメントのルールを追加 |
| `platform-charts/charts/common-db/values.yaml` | `db.enabled: true` を追加 |
| `platform-charts/charts/common-db/templates/_helpers.tpl` | `enabled` フラグで条件分岐を追加 |
| `platform-charts/charts/sample-backend/templates/all.yaml` | db インクルードを `if db.enabled` ブロックに移動 |
| `platform-charts/charts/sample-backend/values.yaml` | `db.enabled: true` を追加 |
| `platform-charts/charts/generic-backend/` | 新規作成（wrapper chart） |
| `platform-charts/charts/generic-frontend/` | 新規作成（wrapper chart） |
| `platform-gitops/backstage/templates/backend/` | 削除（fullstack に置き換え） |
| `platform-gitops/backstage/templates/frontend/` | 削除（fullstack に置き換え） |
| `platform-gitops/backstage/templates/fullstack/` | 新規作成（template.yaml + 3 skeleton） |
| `platform-gitops/platform/backstage/values.yaml` | fullstack template URL に更新 |
| `~/internal/claude/tmp_backstage_software_template.md` | 自己レビュー対応で修正（namespace.yaml 追加、sourceRepos 修正、frontend Application spec 完全記載、HTTPRoute parentRefs 追加） |
| `platform-gitops/backstage/templates/fullstack/template.yaml` | `description` フィールド追加、`output.entityRef` → `output.catalogInfoUrl` 修正 |
| `platform-gitops/.github/workflows/auto-merge-app-pr.yaml` | 新規作成（`app/*` ブランチの PR を自動 squash merge） |
| `platform-charts/charts/common-app/templates/_helpers.tpl` | `db.name` → `db.enabled` で DB 環境変数注入を制御 |
| `platform-charts/charts/generic-backend/charts/common-app-0.3.0.tgz` | 上記修正を反映して再パッケージ |
| `~/internal/claude/tmp_backstage_software_template.md` | 最終実装との差分修正 → `done/` に移動 |
| `~/internal/claude/tmp_backstage_teardown_template.md` | 新規作成（削除フロー設計ドキュメント） |
| `~/internal/claude/PROJECT.md` | Phase12-A11 完了として更新 |
