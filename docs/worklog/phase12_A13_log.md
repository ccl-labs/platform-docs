# Phase12-A13 作業ログ

| # | 作業内容 |
|---|---|
| 1 | PlatformTeam DB（Keycloak・Backstage）MinIO バックアップ設定 |
| 2 | CNPG PodMonitor 対応（common-db Helm Chart 更新） |
| 3 | CNPG 監視アラート実装（PrometheusRule + ArgoCD Application） |
| 4 | apps-gitops 移行不具合の調査・修正 |
| 5 | Discord 通知設定（AlertmanagerConfig + シークレット管理） |
| 6 | AlertmanagerConfig namespace スコープ問題の修正 |
| 7 | PROJECT.md 更新 |
| 8 | MinIO クラウドバックアップ設計 |
| 9 | CNPG ScheduledBackup スケジュール変更（2時→21時） |

---

## 1. PlatformTeam DB（Keycloak・Backstage）MinIO バックアップ設定

### 背景

Phase12-A12 の持ち越し課題として、CNPG 管理の Keycloak・Backstage DB をローカル MinIO にバックアップする設定が残っていた。MinIO は Docker コンテナ（`minio-external`）として WSL 上で稼働中で、k3d クラスタからは `host.k3d.internal:9000` で到達できる。

「ローカル MinIO ディストリビューション分離」タスクについては、この後に実装するクラウドバックアップで WSL 全損に対するデータ保護の目的が達成できるため、対応不要と判断してタスクを削除した。

### 実施手順

Keycloak・Backstage の CNPG Cluster マニフェストに `backup` セクションと `monitoring` セクションを追加した。

```yaml
# 追加内容（keycloak/db-config/cnpg-cluster.yaml・backstage/db-config/cluster.yaml 共通）
backup:
  barmanObjectStore:
    endpointURL: "http://host.k3d.internal:9000"
    destinationPath: "s3://cnpg-backup/keycloak-db"  # backstage は s3://cnpg-backup/backstage-db
    s3Credentials:
      accessKeyId:
        name: minio-backup-secret
        key: ACCESS_KEY_ID
      secretAccessKey:
        name: minio-backup-secret
        key: ACCESS_SECRET_KEY
    wal:
      compression: gzip
    data:
      compression: gzip
  retentionPolicy: "7d"
monitoring:
  enablePodMonitor: true
```

各 namespace に MinIO 認証情報を展開するための ExternalSecret を新規作成した（`minio-backup-secret-source` → `minio-backup-secret`）。sync-wave: "4" を設定し、DB Cluster（wave: "5"）より前に Secret が存在するようにした。

1日1回（午前2時）バックアップを取得する ScheduledBackup を新規作成した（sync-wave: "6"）。

---

## 2. CNPG PodMonitor 対応（common-db Helm Chart 更新）

### 背景

`enablePodMonitor: true` を CNPG Cluster に設定するには、common-db Helm Chart のテンプレートが対応している必要があった。sample-backend は common-db を使っておりすでに PodMonitor を使いたい状況だったため、合わせて対応した。

### 実施手順

`platform-charts/charts/common-db/templates/_helpers.tpl` の cluster テンプレートに `monitoring` セクションを追加した。

```
monitoring:
  enablePodMonitor: {{ .Values.db.monitoring.enablePodMonitor }}
```

`platform-charts/charts/common-db/values.yaml` にデフォルト値を追加した。

```yaml
db:
  monitoring:
    enablePodMonitor: false
```

common-db を利用している `charts/sample-backend/` でも `helm dependency update` を実行して vendored tgz を更新した。

---

## 3. CNPG 監視アラート実装（PrometheusRule + ArgoCD Application）

### 背景

WAL アーカイブ失敗やバックアップ未取得を検知するアラートが必要だった。kube-prometheus-stack が管理する Prometheus にルールを追加するには、`release: kube-prometheus-stack` ラベルが付いた PrometheusRule を作成する必要がある。

### 実施手順

`platform-gitops/platform/monitoring/alerts/cnpg-alerts.yaml` を新規作成した。

| アラート名 | 条件 | for | severity |
|---|---|---|---|
| `CNPGWalArchivingFailing` | `cnpg_collector_pg_wal_archiving_failing_count > 0` | 5m | critical |
| `CNPGBackupNotTaken` | `(time() - cnpg_collector_last_available_backup_timestamp) > 90000` | 15m | warning |
| `CNPGLastBackupFailed` | `cnpg_collector_last_failed_backup_timestamp > cnpg_collector_last_available_backup_timestamp` | 5m | warning |

`CNPGBackupNotTaken` の閾値 90000 秒（25時間）は、1日1回（2:00am）のスケジュールに対して余裕を持たせた値。

ArgoCD Application `platform-alerts` を `root-3-others` に追加し、`platform/monitoring/alerts/` ディレクトリを管理対象とした。

---

## 4. apps-gitops 移行不具合の調査・修正

### 背景

Phase12-A12 で apps-gitops 分離を行ったが、sample-backend の ArgoCD sync が失敗していた。

### 問題

sample-backend Application で以下のエラーが発生していた。

```
apps/sample-backend/values.yaml: no such file or directory
```

### 原因

多層的な原因があった。

1. **apps-root App が旧構成のまま残存**: `user-apps` Application が `apps-gitops` を指すよう変更したが、手動作成の `apps-root` Application（`tracking-id` アノテーション付き）が旧 `platform-gitops` の `apps/` ディレクトリを指したまま存在し続けていた。ArgoCD はリソースの `tracking-id` アノテーションを見てどの App が管理するかを決定するため、`user-apps` が新しいマニフェストを apply しても `apps-root` が owner として上書きし続けた。

2. **Application ファイルの配置パターン**: Phase12-A12 では `apps-gitops/apps/sample-backend.yaml` のように配置したが、`user-apps` App-of-Apps が検出するパターンは `apps/*/application.yaml` だった。

3. **`$values` ソースの repoURL**: multi-source Application の `$values` ref が `platform-gitops` を指していたが、values ファイルは `apps-gitops` に移行済みだった。

### 対処

1. `apps-root` Application を ArgoCD GUI から削除
2. `sample-backend`・`sample-frontend`・`app-developer-rbac` の Application CR を ArgoCD GUI から削除（apps-root が owner だったため ArgoCD 外で再作成が必要だった）
3. Application マニフェストを `apps/*/application.yaml` パターンに再配置（`apps/sample-backend.yaml` → `apps/sample-backend/application.yaml`）
4. 全 Application の `$values` ソース `repoURL` を `apps-gitops` に修正
5. `platform/user-apps-infra/` を新規作成し、`sample-app` Namespace・`sample-apps` AppProject・user-apps ArgoCD Application を GitOps 管理に移行

---

## 5. Discord 通知設定（AlertmanagerConfig + シークレット管理）

### 背景

CNPG アラートを受信したら Discord に通知する設定を行った。AlertManager v0.27+ には Discord ネイティブの receiver（`discordConfigs`）がある。

### 実施手順

**シークレット管理:**

Discord Webhook URL は平文保存せず SOPS 暗号化で管理した。ユーザーが別ターミナルで以下を実行して暗号化ファイルを作成した。

```bash
sops platform/secrets/sources/discord-webhook-secret-source.yaml
```

`platform-gitops/platform/secrets/sources/secret-generator.yaml` に `./discord-webhook-secret-source.yaml` を追記し、ksops 管理下に加えた。

ESO ExternalSecret（`platform/monitoring/alerts/discord-webhook-secret.yaml`）を新規作成し、`platform-secrets` namespace の source secret から `monitoring` namespace に `url` キーを展開するよう設定した。

**AlertmanagerConfig:**

`platform/monitoring/alerts/alertmanagerconfig.yaml` を新規作成した。

```yaml
spec:
  route:
    receiver: discord-cnpg
    matchers:
      - name: alertname
        matchType: =~
        value: "CNPG.*"
    groupBy: [alertname, namespace, cluster_name]
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 12h
  receivers:
    - name: discord-cnpg
      discordConfigs:
        - apiURL:
            name: discord-webhook-secret
            key: url
```

当初 `webhookURL` フィールド名を使用したところ CRD バリデーションエラーになった。AlertManager の Discord receiver では `apiURL` が正しいフィールド名だった。

---

## 6. AlertmanagerConfig namespace スコープ問題の修正

### 問題

Discord 通知が届かなかった。Alertmanager の設定を確認すると、`AlertmanagerConfig` から生成されたルートに `namespace="monitoring"` マッチャーが自動付与されていた。

```
- alertname=~"CNPG.*"
- namespace="monitoring"   ← 自動付与
```

CNPG アラートの namespace は `keycloak`・`backstage`・`sample-app` であるため、この matcher によってすべてのアラートが弾かれていた。

### 原因

`AlertmanagerConfig` は namespace スコープのリソースであり、デフォルトでは所属 namespace 外のアラートを受け取れないようにするため、`namespace` ラベルの matcher が自動で追加される仕様になっている。

### 対処

kube-prometheus-stack の `values.yaml` に以下を追加し、namespace matcher の自動付与を無効化した。

```yaml
alertmanager:
  alertmanagerSpec:
    alertmanagerConfigMatcherStrategy:
      type: None
```

`type: None` にすると全 namespace のアラートを受け取れるようになる。この設定は全クラスタ共通で適用されるため、将来 AlertmanagerConfig を追加する場合は自分で namespace 等の matcher を明示する必要がある。

修正後、Discord への通知が正常に届くことを確認した。初回発火まで約5分かかったのは、直前のテスト実行で同一アラートグループが既に作成されており `groupWait` ではなく `groupInterval`（5分）が適用されたためで、期待通りの動作だった。

---

## 7. PROJECT.md 更新

以下を更新した。

- 持ち越し課題から「PlatformTeam DB Backup」「ローカルMinio ディストリビューション分離」を削除
- 運用メモに「CNPG 監視・Discord 通知」「apps-gitops 移行不具合修正」を追記

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-gitops/platform/keycloak/db-config/cnpg-cluster.yaml` | backup・monitoring セクション追加 |
| `platform-gitops/platform/keycloak/db-config/minio-backup-secret.yaml` | 新規作成（ExternalSecret） |
| `platform-gitops/platform/keycloak/db-config/scheduledbackup.yaml` | 新規作成（ScheduledBackup daily 21:00） |
| `platform-gitops/platform/backstage/db-config/cluster.yaml` | backup・monitoring セクション追加 |
| `platform-gitops/platform/backstage/db-config/minio-backup-secret.yaml` | 新規作成（ExternalSecret） |
| `platform-gitops/platform/backstage/db-config/scheduledbackup.yaml` | 新規作成（ScheduledBackup daily 21:00） |
| `platform-charts/charts/common-db/templates/_helpers.tpl` | monitoring.enablePodMonitor テンプレート追加 |
| `platform-charts/charts/common-db/values.yaml` | db.monitoring.enablePodMonitor デフォルト値追加 |
| `platform-gitops/platform/monitoring/alerts/cnpg-alerts.yaml` | 新規作成（PrometheusRule 3アラート） |
| `platform-gitops/platform/applications/root-3-others/platform-alerts.yaml` | 新規作成（ArgoCD Application） |
| `platform-gitops/platform/user-apps-infra/namespace.yaml` | 新規作成（sample-app Namespace） |
| `platform-gitops/platform/user-apps-infra/appproject.yaml` | 新規作成（sample-apps AppProject） |
| `platform-gitops/platform/applications/root-3-others/user-apps-infra.yaml` | 新規作成（ArgoCD Application） |
| `apps-gitops/apps/sample-backend/application.yaml` | 移動・ソース修正（$values repoURL を apps-gitops に） |
| `apps-gitops/apps/sample-frontend/application.yaml` | 移動・ソース修正（同上） |
| `apps-gitops/apps/app-developer-rbac/application.yaml` | 移動・ソース修正（同上） |
| `platform-gitops/platform/monitoring/alerts/discord-webhook-secret.yaml` | 新規作成（ExternalSecret） |
| `platform-gitops/platform/monitoring/alerts/alertmanagerconfig.yaml` | 新規作成（AlertmanagerConfig） |
| `platform-gitops/platform/secrets/sources/secret-generator.yaml` | discord-webhook-secret-source 追記 |
| `platform-gitops/platform/secrets/sources/discord-webhook-secret-source.yaml` | 新規作成（SOPS 暗号化） |
| `platform-gitops/platform/monitoring/values.yaml` | alertmanagerConfigMatcherStrategy.type: None 追加 |
| `~/internal/claude/PROJECT.md` | 完了タスク削除・運用メモ追記 |
| `platform-gitops/platform/keycloak/db-config/scheduledbackup.yaml` | スケジュール 2:00am → 21:00 に変更（#9） |
| `platform-gitops/platform/backstage/db-config/scheduledbackup.yaml` | スケジュール 2:00am → 21:00 に変更（#9） |
| `~/internal/claude/tmp_minio_cloud_backup.md` | 新規作成（MinIO クラウドバックアップ設計書）（#8） |

---

## 8. MinIO クラウドバックアップ設計

### 背景

WSL 全損時のデータ保護として、ローカル MinIO に蓄積された CNPG バックアップデータをクラウドへ退避する設計を行った。GCS を採用した理由は Phase13/14 で Google Cloud メインの構成を予定しているため。

### 設計内容

**アーキテクチャ選択: WSL ホスト（cron + rclone）**

MinIO データは `~/minio-data/` という WSL ホストのファイルシステム上にあるため、Kubernetes CronJob より WSL cron の方が障害ポイントが少なく適切と判断した。Phase13 でクラウド移行後は MinIO ごと不要になる。

**コピー方式: `rclone copy`（`rclone sync` ではなく）**

MinIO 側は CNPG の `retentionPolicy: 7d` で古いバックアップが自動削除される。`rclone sync` にするとその削除が GCS にも伝播し、クラウドバックアップの意義が薄れる。`rclone copy` で GCS に累積保存し、GCS の Object Lifecycle ルール（30日 Delete）で管理する。

**GCS リージョン: `us-central1`**

GCS の Always Free（5GB/月）は US リージョン限定。`asia-northeast1`（東京）は無料枠対象外のため、レイテンシ不問の非同期バックアップには `us-central1` を選択した。

**MinIO 認証情報の取得**

MinIO コンテナは `MINIO_ROOT_PASSWORD_FILE` を使っており、`docker inspect` からの環境変数取得は不確実。SOPS 管理の `minio-backup-secret-source.yaml` をスクリプト内で復号して使用する。

**スケジュール設計**

個人 PC（WSL）は深夜帯に停止している可能性が高いため、夜間（PC 起動中の時間帯）に設定した。

| ジョブ | スケジュール | 理由 |
|---|---|---|
| CNPG ScheduledBackup | 毎日 21:00 | PC 起動中の時間帯 |
| GCS sync（cron） | 毎日 23:00 | CNPG バックアップ完了後 2時間のバッファ |

確認済み事項: rclone は `aqua:rclone/rclone` で mise 管理可能、MinIO コンテナ名 `minio-external` 確認、cron サービス active 確認、現在のバックアップサイズ 314MB（30日累積でも 5GB 以内の見込み）。

設計書: `~/internal/claude/tmp_minio_cloud_backup.md`

---

## 9. CNPG ScheduledBackup スケジュール変更（2時→21時）

設計（#8）で決定したスケジュールに合わせ、keycloak・backstage の ScheduledBackup を即時変更した。

```yaml
# 変更前
schedule: "0 2 * * *"

# 変更後
schedule: "0 21 * * *"
```

GCS sync 側（cron `0 23 * * *`）は次セッションのクラウドバックアップ実装時に設定する。
