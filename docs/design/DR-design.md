# DR 設計書

> **ステータス**: 作成中（バックアップ設計・CNPG リストア設計のみ詳細記述済み。その他章は骨子のみ）

---

## 1. 設計方針

### 1.1 何を守るか

本基盤における DR の主な保護対象は PostgreSQL データ（Keycloak・Backstage・アプリ DB）。
その他のコンポーネント（ArgoCD・Prometheus 等）はすべて GitOps から再構築可能なため、DR 対象外。

### 1.2 障害シナリオの分類

| シナリオ | 内容 | 対応方針 |
|---|---|---|
| **Pod / PV 障害** | CNPG Pod 異常・PVC 破損 | CNPG の自己修復 + WAL リプレイ |
| **クラスター全損** | `k3d cluster delete` 等でクラスター喪失 | MinIO（ローカル）からリストア |
| **WSL 全損** | WSL ディストリビューション全体の消失 | GCS（クラウド）からリストア |

### 1.3 RTO / RPO 目標

| シナリオ | RPO | RTO |
|---|---|---|
| Pod / PV 障害 | WAL ラグ分（数十秒以内） | 数分 |
| クラスター全損 | 最終 ScheduledBackup 時刻（21:00） | 1 時間以内（未計測） |
| WSL 全損 | 最終 GCS 同期時刻（翌日 00:00） | 数時間（未計測） |

> RTO/RPO は現時点では目標値。DR 手順書完成後に実測で検証する。

---

## 2. バックアップ設計

### 2.1 PostgreSQL（CNPG）

CNPG の barman-cloud を使い、MinIO（WSL ローカル）に継続的バックアップを取得する。

| 種別 | 内容 | スケジュール |
|---|---|---|
| WAL アーカイブ | PostgreSQL の変更ログをリアルタイムで転送 | 随時（CNPG が自動管理） |
| ScheduledBackup | ベースバックアップ（フルダンプ相当）| 毎日 21:00 |
| 保持期間 | `retentionPolicy: 7d` | 7 日分 |

```
CNPG Pod
  → barman-cloud-wal-archive（WAL）  ─→ MinIO: s3://cnpg-backup/<cluster-name>/wals/
  → barman-cloud-backup（ベース）    ─→ MinIO: s3://cnpg-backup/<cluster-name>/data/
```

**接続設定**:
- エンドポイント: `http://host.k3d.internal:9000`（k3d コンテナから WSL ホストへ）
- 認証情報: `minio-backup-secret`（ESO 経由で各 namespace に配布）
- `host.k3d.internal` は k3d 起動時に CoreDNS NodeHosts へ動的登録（`make cluster-start` に組み込み済み）

### 2.2 クラウドへのオフサイトバックアップ（MinIO → GCS）

MinIO 上のバックアップを GCS（us-central1）へ rclone で同期し、WSL 全損に備える。

> **詳細設計**: `~/internal/claude/tmp_minio_cloud_backup.md` 参照
> **実装状況**: 未実装（Phase 13 前に対応予定）

---

## 3. リストア設計

### 3.1 CNPG リストア設計方針

#### bootstrap セクションの性質

CNPG の `spec.bootstrap` はクラスター初回作成時（PVC が空の場合）のみ実行される一度きりの初期化イベントであり、PVC にデータが存在すれば以後は無視される。また、CNPG の validating webhook により **bootstrap のメソッド変更はイミュータブル**（`initdb` ↔ `recovery` の変更は実行時に拒否される）。

この性質から、bootstrap セクションは「GitOps の定常管理対象」ではなく「初期化イベントの記述」として扱う。

#### GitOps 定常状態と DR 手順の分離

| | 役割 | bootstrap |
|---|---|---|
| **GitOps マニフェスト** | クラスターの定常状態を宣言 | `initdb`（PVC が空の場合のデフォルト） |
| **DR マニフェスト** | 障害復旧時のみ使用（ArgoCD 管理外） | `bootstrap.recovery` + `externalClusters` |

GitOps と DR 手順を同一マニフェストで解決しようとしないことが重要。ArgoCD はクラスターの定常運用を管理し、DR 手順は別途スクリプト・Runbook として管理する。

#### ArgoCD ignoreDifferences の設定

DR 実行後、ArgoCD はクラスターの `spec.bootstrap` が `recovery` になっていることを検知し `initdb` に戻そうとするが、CNPG webhook がこれを拒否する。これを防ぐため、対象の ArgoCD Application に以下の設定を追加する（**実装済み**）。

```yaml
spec:
  ignoreDifferences:
    - group: postgresql.cnpg.io
      kind: Cluster
      jsonPointers:
        - /spec/bootstrap
        - /spec/externalClusters
```

`bootstrap` は一度きりの初期化イベントであり GitOps の管轄外という設計思想を ArgoCD に明示するものであり、これは本番標準のパターン。

対象 Application（keycloak-db / backstage-db / sample-backend）に設定済み。`RespectIgnoreDifferences=true` を syncOptions に合わせて設定することで、sync 実行時にも該当フィールドへのパッチを抑止している。

#### PVC Retain ポリシー

DR クラスター（`recovery`）でリストア完了後、DR クラスターを削除して GitOps に戻す際、PVC を残しておくことでデータを保全する。ArgoCD が `initdb` マニフェストでクラスターを再作成した際、CNPG は PVC 上の既存データを検知し bootstrap をスキップする。

**実装済み**:
- `local-path-retain` StorageClass（`reclaimPolicy: Retain`）を GitOps で管理（root-1-gateway wave 0 で適用）
- 全 CNPG クラスターの `storage.storageClass: local-path-retain` を設定済み（新規 PVC から自動適用）
- 既存 PV は `kubectl patch` で Retain に変更済み

#### DR マニフェスト生成（実装済み）

DR マニフェストは静的ファイルとして管理せず、`make generate-dr-manifests` で GitOps ソースから動的生成する。これにより、クラスター設定変更時のドリフトを防ぐ。

```bash
# DR 手順の最初のステップとして実行（クラスターが落ちていても動作する）
cd ~/platform-infra
make generate-dr-manifests
# → k3d/dr/<cluster-name>-recovery.yaml を生成
```

生成スクリプト（`k3d/scripts/generate-dr-manifests.py`）は以下を自動スキャンする:
- `platform-gitops/platform/**/*.yaml` — `kind: Cluster` + `barmanObjectStore` を持つもの
- `apps-gitops/apps/*/values.yaml` — `db.backup.enabled: true` のもの

新しいクラスターが追加されても手動メンテナンス不要。生成ファイルは gitignore 済み（`k3d/dr/*.yaml`）。

WSL 全損時は生成後に `endpointURL` を GCS エンドポイントに書き換えてから apply する（Runbook 参照）。

### 3.2 クラスター内障害（Pod / PV 障害）からの復旧

> **未記述**: CNPG の自動修復フロー、手動介入が必要なケースの手順

### 3.3 クラスター全損からの復旧（k3d 再作成）

> **未記述**: DR マニフェストを使った手動リストア手順（3.1 の設計に基づく）

### 3.4 WSL 全損からの復旧

> **未記述**: GCS からローカル MinIO への復元 → 3.3 に続く手順

---

## 4. 実装状況

| 項目 | 状態 | 参照 |
|---|---|---|
| `ignoreDifferences` 設定 | **完了** | 3.1 節 |
| PVC Retain ポリシー設定 | **完了**（`local-path-retain` SC + 既存 PV パッチ） | 3.1 節 |
| DR マニフェスト生成スクリプト | **完了**（`make generate-dr-manifests`） | 3.1 節 |
| クラウドバックアップ実装 | 未実装（Phase 13 前に対応予定） | `tmp_minio_cloud_backup.md` |
| DR 手順書（Runbook）作成 | 未実装（クラウドバックアップ完了後） | 3.2〜3.4 節 |
| RTO/RPO 実測 | 未実装（DR 手順確立後に計測） | 1.3 節 |
