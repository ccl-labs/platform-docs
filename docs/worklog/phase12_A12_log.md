# Phase12-A12 作業ログ

| # | 作業内容 |
|---|---|
| 1 | Backstage Software Template 実装（common-db フラグ追加・generic chart 作成・テンプレート実装） |
| 2 | テスト前の設計見直し（fullstack テンプレートへの再設計） |

---

## 1. Backstage Software Template 実装

設計ドキュメント（`~/internal/claude/tmp_backstage_software_template.md`）に基づき実装。

### 実装順序と実施内容

**① common-db に db.enabled フラグを追加（platform-charts）**

`common-db` に `db.enabled` フラグが存在しなかったため追加。library chart の values は親 chart に引き継がれないため、`sample-backend/values.yaml` にも `db.enabled: true` を追加した。

- `charts/common-db/values.yaml`: `db.enabled: true` を追加
- `charts/common-db/templates/_helpers.tpl`: `common-db.cluster` を `{{- if .Values.db.enabled }}` で囲む。`scheduledBackup` の条件を `and db.enabled backup.enabled` に変更
- `charts/sample-backend/templates/all.yaml`: db インクルードを `{{- if .Values.db.enabled }}` ブロックに移動
- `charts/sample-backend/values.yaml`: `db.enabled: true` を追加

`helm template` で `db.enabled=true`（Cluster あり）と `db.enabled=false`（Cluster なし）の両ケースを確認済み。

**② generic-backend / generic-frontend chart を新規作成（platform-charts）**

`sample-backend` / `sample-frontend` と同じ wrapper chart 構成で作成。`generic-frontend/values.yaml` に `db.name: ""` が必要な点は実装時に確認（`common-app.deployment` が `.Values.db.name` を参照するため）。

**③ backend テンプレートを実装（platform-gitops）**

`backstage/templates/backend/` に template.yaml + app-skeleton + gitops-skeleton を作成。

設計ドキュメントとの差異（実装時に判明）:
- app-skeleton のワークフロー: `build.yaml` 1本に統合（`update-gitops.yaml` は platform-gitops 側に既存）
- `publish:github:pull-request`: `sourcePath: ./gitops` + `targetPath: .` の両指定が必要
- `catalog:register`: `repoContentsUrl` + `catalogInfoPath` を使用（`publish:github` の出力を活用）
- `build.yaml` 内の GitHub Actions 変数は `${{ "{{" }} github.sha {{ "}}" }}` 形式でエスケープが必要

**④ frontend テンプレートを実装（platform-gitops）**

`backstage/templates/frontend/` に backend と対称な構成で作成。

**⑤ platform/backstage/values.yaml に template URL を追加**

`catalog.locations` に backend・frontend template URL を追加。

**⑥ Backstage hard refresh → 動作確認**

ArgoCD で Backstage を sync 後、Backstage GUI で hard refresh。Scaffolderに backend・frontend の2テンプレートが表示されることを確認。

---

## 2. テスト前の設計見直し（fullstack テンプレートへの再設計）

Scaffolderのテスト実行前に、Namespace 設計の問題が判明。

### 問題

現在の設計では backend/frontend それぞれが独立した Namespace を持つ。しかし、同一アプリの front/back は常にセットで作成されるため、同一 Namespace に同居させる構成が自然。

### 設計変更の決定事項

| 項目 | 変更前 | 変更後 |
|---|---|---|
| テンプレート構成 | backend + frontend の2テンプレート | fullstack 1テンプレート |
| リポジトリ命名 | `<appName>` | `<appName>-backend` / `<appName>-frontend` |
| Namespace | テンプレートごとに独立 | 1アプリ = 1 Namespace（`<appName>`） |
| ArgoCD Application | 1本 | 2本（`<appName>-backend` / `<appName>-frontend`） |
| HTTPRoute | 各アプリに独立したホスト | 同一ホスト、パスで分割（`/api` → backend、`/` → frontend） |

backend-only が必要な場合は別途テンプレートを作成する方針。

### 未解決の設計課題（次セッションで判断）

1. **`managedNamespaceMetadata` の扱い**: backend/frontend 2つの ArgoCD Application が同一 Namespace を管理する場合、`managedNamespaceMetadata`（Namespace ラベル設定）をどちらに持たせるか。frontend が先に sync した場合、ラベルなし Namespace が一時的に作られ `ghcr-pull-secret` 注入が失敗する可能性がある。sync-wave での順序制御も選択肢。
2. **同一ホスト・複数 HTTPRoute の挙動**: Envoy Gateway での実際の動作は未確認。

### この時点でのファイル状態

- `backstage/templates/backend/` と `backstage/templates/frontend/` は作成済みだが、次セッションで fullstack テンプレートへの置き換え予定
- 設計ドキュメント（`~/internal/claude/tmp_backstage_software_template.md`）は fullstack 設計に更新済み

---

## 変更ファイル一覧

| リポジトリ | ファイル | 変更内容 |
|---|---|---|
| platform-charts | `charts/common-db/values.yaml` | `db.enabled: true` を追加 |
| platform-charts | `charts/common-db/templates/_helpers.tpl` | `enabled` フラグで条件分岐を追加 |
| platform-charts | `charts/sample-backend/templates/all.yaml` | db インクルードを `if db.enabled` ブロックに移動 |
| platform-charts | `charts/sample-backend/values.yaml` | `db.enabled: true` を追加 |
| platform-charts | `charts/generic-backend/` | 新規作成（wrapper chart） |
| platform-charts | `charts/generic-frontend/` | 新規作成（wrapper chart） |
| platform-gitops | `backstage/templates/backend/` | 新規作成（次セッションで fullstack に置き換え予定） |
| platform-gitops | `backstage/templates/frontend/` | 新規作成（次セッションで fullstack に置き換え予定） |
| platform-gitops | `platform/backstage/values.yaml` | backend/frontend template URL を追加 |
| internal/claude | `tmp_backstage_software_template.md` | fullstack 設計に更新 |
