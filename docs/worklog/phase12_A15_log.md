# Phase12-A15 作業ログ

| # | 作業内容 |
|---|---|
| 1 | トラブルシュート：cilium status に存在しない --wait フラグを削除 |
| 2 | トラブルシュート：WSL 再起動後の kubelet TLS 証明書 IP 不一致を cluster-start で自動修正 |
| 3 | トラブルシュート：cluster-start 後に host.k3d.internal が CoreDNS から消える問題を修正 |
| 4 | トラブルシュート：クラスター再作成後の CNPG WAL "Expected empty archive" エラー原因調査 |
| 5 | CNPG バックアップ・リストア設計の整理（bootstrap イミュータブル制約・GitOps と DR 手順の分離・ignoreDifferences・PVC Retain） |
| 6 | CNPG DR 実装：ArgoCD Application に ignoreDifferences を追加（3 Application） |
| 7 | CNPG DR 実装：PVC Retain 対応（local-path-retain StorageClass 作成・既存 PV パッチ・CNPG 設定更新） |
| 8 | CNPG DR 実装：DR マニフェスト動的生成スクリプト実装（make generate-dr-manifests） |

---

## 1. cilium status に存在しない --wait フラグを削除

### 背景

PC シャットダウンにより中断した前セッションの続き。`make cluster-start` 末尾の `cilium status` コマンドにオプションを追加していたが未コミットのまま終了していた。

### 問題

```
Error: unknown flag: --wait
```

### 原因

`cilium-dbg status` に `--wait` フラグは存在しない。Cilium のバージョンや CLI のサブコマンド体系を確認せずに追加したため。

### 対処

Makefile から `--wait` を削除した。

```makefile
# 変更前
kubectl exec -n kube-system ds/cilium -- cilium status --wait
# 変更後
kubectl exec -n kube-system ds/cilium -- cilium status
```

---

## 2. WSL 再起動後の kubelet TLS 証明書 IP 不一致を cluster-start で自動修正

### 背景

WSL 再起動後に `kubectl exec` が失敗するという問題が継続的に発生していた。`make cluster-start` での恒久対応を実装した。

### 問題

```
error: Internal error occurred: error executing command in container:
  failed to exec in container: failed to start exec "...":
  OCI runtime exec failed: exec failed: unable to start container process:
  x509: certificate is valid for 127.0.0.1, 172.19.0.5, not 172.19.0.3
```

### 原因

WSL 再起動時に Docker がノードコンテナに別の IP を再割り当てすることがある。kubelet の TLS serving 証明書（`/var/lib/rancher/k3s/agent/serving-kubelet.crt`）は旧 IP で発行されているため、新しい IP からの `kubectl exec` リクエストで x509 エラーが発生する。

### 対処

`cluster-start` 内で各ノードの kubelet 証明書を削除し `docker restart` することで、k3s に現在の IP で証明書を再生成させる。

```makefile
@for node in $$(k3d node list --no-headers | grep -v loadbalancer | awk '{print $$1}'); do \
    docker exec $$node rm -f /var/lib/rancher/k3s/agent/serving-kubelet.crt \
                              /var/lib/rancher/k3s/agent/serving-kubelet.key; \
    docker restart $$node > /dev/null; \
done
@sleep 10
```

ロードバランサーノードは kubelet を持たないため除外している。`sleep 10` は docker restart 後に k3s プロセスが再起動するまでの待機時間。

---

## 3. cluster-start 後に host.k3d.internal が CoreDNS から消える問題を修正

### 背景

`make cluster-start` 後、CNPG Pod が MinIO（`http://host.k3d.internal:9000`）に接続できない事象が発生。`host.k3d.internal` の DNS 解決が失敗していた。

### 問題

`make bootstrap` の `fix-coredns` ステップで CoreDNS NodeHosts に `host.k3d.internal` を登録しているにもかかわらず、`cluster-start` 後に消えていた。

### 原因

k3s はノード再登録イベント時に CoreDNS ConfigMap の NodeHosts セクションをノード IP 一覧で上書きする。`fix-coredns` は `make bootstrap`（クラスター初回作成時）のみに含まれており、`cluster-start`（WSL 再起動後の再開時）には含まれていなかった。

### 対処

`cluster-start` の末尾に `fix-coredns` を追加した。

```makefile
@echo ">>> CoreDNS に host.k3d.internal を再登録中..."
$(MAKE) -C k3d fix-coredns
@echo ">>> クラスター起動完了"
```

---

## 4. クラスター再作成後の CNPG WAL "Expected empty archive" エラー原因調査

### 背景

クラスターを再作成した後、CNPG の WAL アーカイブが失敗し続けていた。

### 問題

```
barman-cloud-check-wal-archive: Expected empty archive
```

### 原因

MinIO に旧クラスターのバックアップデータ（`s3://cnpg-backup/<cluster-name>/`）が残っているため、CNPG の安全チェック `barman-cloud-check-wal-archive` が新クラスターによる WAL アーカイブを拒否する。これは CNPG が「別クラスターのバックアップに誤って上書きしない」ためのガードである。

新規 `initdb` クラスターがアーカイブ先に既存データを見つけた場合、WAL アーカイブは恒久的に失敗し続ける。

### 対処（暫定）

今回は調査のみ。次の ScheduledBackup（21:00）まで WAL アーカイブのリトライが継続する状態のまま。

**本対応（将来方針）**: DR 実装（`ignoreDifferences` 設定・PVC Retain 確認・DR マニフェスト作成）により根本解決する。詳細は次項および `platform-docs/docs/design/DR-design.md` 参照。

---

## 5. CNPG バックアップ・リストア設計の整理

### 背景

4 の問題をきっかけに、「クラスター再作成時に MinIO のデータを削除してから initdb で再作成する」という従来の運用フローが設計的に誤りであることが判明した。MinIO にバックアップを持たせている意味がなくなるためである。正しい運用フローと本番標準の設計を整理した。

### bootstrap セクションのイミュータブル制約

CNPG の `spec.bootstrap` はクラスター初回作成時（PVC が空の場合）のみ実行される一度きりの初期化イベント。CNPG の validating webhook により、クラスター作成後は `initdb` ↔ `recovery` の変更が拒否される。

```
spec.bootstrap: Forbidden: Only one bootstrap method can be specified at a time
```

この制約により、ArgoCD が GitOps マニフェスト（`initdb`）で DR クラスター（`recovery`）を上書きしようとしても webhook が拒否するため、ArgoCD は永続的に Degraded 状態になる。

### GitOps と DR 手順の分離

本番標準の設計方針は以下の通り。

| | 役割 | bootstrap |
|---|---|---|
| GitOps マニフェスト | クラスターの定常状態を宣言 | `initdb`（PVC が空の場合のデフォルト） |
| DR マニフェスト | 障害復旧時のみ使用（ArgoCD 管理外） | `bootstrap.recovery` + `externalClusters` |

GitOps と DR 手順を同一マニフェストで解決しようとしない。bootstrap は「初期化イベント」であり「GitOps の定常管理対象」ではない。

### ArgoCD ignoreDifferences

DR 実行後に ArgoCD が `spec.bootstrap` の差分を検知して修正しようとするのを防ぐため、対象の ArgoCD Application に以下を設定する（→ 6 節で実装）。

```yaml
spec:
  ignoreDifferences:
    - group: postgresql.cnpg.io
      kind: Cluster
      jsonPointers:
        - /spec/bootstrap
        - /spec/externalClusters
```

### PVC Retain ポリシー

DR クラスター（`recovery`）でリストア完了後、DR クラスターを削除して GitOps に戻す際、PVC を残しておくことでデータを保全する。ArgoCD が `initdb` マニフェストでクラスターを再作成したとき、CNPG は PVC 上の既存データを検知して bootstrap をスキップする（→ 7 節で実装）。

---

## 6. CNPG DR 実装：ignoreDifferences 追加

### 背景

5 節の設計整理を受けて実装に移った。DR 後に ArgoCD が `spec.bootstrap` を `initdb` に戻そうとすると CNPG webhook に拒否されて sync エラーが発生するため、その差分を ArgoCD に無視させる設定が必要。

### 実施内容

keycloak-db / backstage-db / sample-backend の 3 Application に以下を追加した。

```yaml
ignoreDifferences:
  - group: postgresql.cnpg.io
    kind: Cluster
    jsonPointers:
      - /spec/bootstrap
      - /spec/externalClusters
```

`RespectIgnoreDifferences=true` は keycloak-db と sample-backend には既存だったが、backstage-db には未設定だったため合わせて追加した。このオプションがないと sync 実行時にフィールドへのパッチが走り webhook に弾かれる。

---

## 7. CNPG DR 実装：PVC Retain 対応

### 背景

PVC Retain の必要性を整理した上で実装した。k3d クラスター全損（`k3d cluster delete`）では Docker ボリュームごと消えるため PV の reclaimPolicy は関係ない。Retain が必要な場面は **MinIO 復元フロー** における切り戻し時である。

MinIO 復元フロー中の Retain が必要な理由：

```
1. recovery クラスター apply → MinIO からデータを PV に復元
2. recovery クラスターを削除
   ├─ [Retain あり] PV が Released 状態で残る → データ保全
   └─ [Retain なし] PV ごと削除 → 復元データが消える
3. ArgoCD の initdb クラスターが PVC を作成 → 既存 PV にバインド
4. CNPG が PV 上のデータを検知 → bootstrap をスキップ → 通常運用に復帰
```

### 実施内容

**既存 PV のパッチ**（現在稼働中の 4 本を即時対応）

```bash
for pv in pvc-23bb899e-... pvc-72157eb4-... pvc-bfabf66b-... pvc-cb3f2fcd-...; do
  kubectl patch pv $pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
done
```

**local-path-retain StorageClass の作成**

新規 PVC が自動的に Retain になるよう、rancher.io/local-path プロビジョナーベースの StorageClass を platform-gitops で GitOps 管理することにした。root-1-gateway の wave 0 として適用（DB クラスターより先に作成される必要があるため）。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-retain
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

**CNPG クラスター設定の更新**

keycloak-db・backstage-db は raw YAML の `storage.storageClass` に直接追記。sample-backend-db は common-db Helm チャートに `storageClassName` パラメータを新設し values.yaml で指定する形にした。

既存クラスターの既存 PVC は変更されないが、次回 PVC 作成（DR リカバリー時など）から自動的に Retain が適用される。

---

## 8. CNPG DR 実装：DR マニフェスト動的生成スクリプト実装

### 背景

当初は 3 クラスター分の静的な DR マニフェストを `k3d/dr/` に作成したが、クラスター設定（image バージョン・ストレージサイズ等）が変わった際にドリフトが生じるという問題があった。

管理方式として以下を検討した：

| 方式 | 評価 |
|---|---|
| 静的ファイルを置く | ドリフトが発生する |
| Scaffolder に組み込む | PE 側ミドルウェア DB など Scaffolder を通らないケースが漏れる |
| `make generate-dr-manifests` | DR 手順の一ステップとして実行、GitOps ソースから常に最新を生成 |

本番でも「B + 自動実行」パターンは存在するが、生成物を自動コミットすると変更の追跡・承認フローが失われることと、クラスター障害時に生成できないリスクがあるため不適切と判断。DR 手順の中で明示的に実行する形（手動トリガー + GitOps ソースから生成）を採用した。

### 実施内容

`k3d/scripts/generate-dr-manifests.py` を作成。GitOps リポジトリをスキャンして DR マニフェストを生成する。

**スキャン対象の自動検出ロジック**：
- `platform-gitops/platform/**/*.yaml` — `apiVersion: postgresql.cnpg.io/v1` + `kind: Cluster` + `barmanObjectStore` を持つもの
- `apps-gitops/apps/*/values.yaml` — `db.backup.enabled: true` のもの（namespace は同階層の application.yaml から取得）

新しいクラスターが追加されても手動メンテナンス不要。生成ファイルは `k3d/dr/*.yaml`（gitignore 済み）に出力される。

```bash
cd ~/platform-infra
make generate-dr-manifests
# → k3d/dr/keycloak-db-recovery.yaml
# → k3d/dr/backstage-db-recovery.yaml
# → k3d/dr/sample-backend-db-recovery.yaml
```

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-infra/Makefile` | cluster-start: `cilium status --wait` → `cilium status`、kubelet 証明書削除 + `docker restart` ループ追加、末尾に `fix-coredns` 追加 |
| `platform-infra/Makefile` | `generate-dr-manifests` ターゲット追加 |
| `platform-infra/k3d/Makefile` | `generate-dr-manifests` ターゲット追加 |
| `platform-infra/k3d/scripts/generate-dr-manifests.py` | 新規作成：DR マニフェスト動的生成スクリプト |
| `platform-infra/k3d/dr/.gitignore` | 新規作成：生成ファイル（*.yaml）を gitignore |
| `platform-gitops/platform/applications/root-1-gateway/storage.yaml` | 新規作成：local-path-retain StorageClass 管理 Application |
| `platform-gitops/platform/storage/local-path-retain.yaml` | 新規作成：local-path-retain StorageClass |
| `platform-gitops/platform/applications/root-2-auth/keycloak-db.yaml` | CNPG ignoreDifferences 追加 |
| `platform-gitops/platform/applications/root-3-others/backstage-db.yaml` | CNPG ignoreDifferences 追加・RespectIgnoreDifferences=true 追加 |
| `platform-gitops/platform/keycloak/db-config/cnpg-cluster.yaml` | `storage.storageClass: local-path-retain` 追加 |
| `platform-gitops/platform/backstage/db-config/cluster.yaml` | `storage.storageClass: local-path-retain` 追加 |
| `platform-charts/charts/common-db/templates/_helpers.tpl` | `storageClass` フィールドの条件付き出力を追加 |
| `platform-charts/charts/common-db/values.yaml` | `storageClassName` パラメータ追加 |
| `apps-gitops/apps/sample-backend/application.yaml` | CNPG ignoreDifferences 追加 |
| `apps-gitops/apps/sample-backend/values.yaml` | `storageClassName: local-path-retain` 追加 |
