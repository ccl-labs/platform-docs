# Phase12-A15 作業ログ

| # | 作業内容 |
|---|---|
| 1 | トラブルシュート：cilium status に存在しない --wait フラグを削除 |
| 2 | トラブルシュート：WSL 再起動後の kubelet TLS 証明書 IP 不一致を cluster-start で自動修正 |
| 3 | トラブルシュート：cluster-start 後に host.k3d.internal が CoreDNS から消える問題を修正 |
| 4 | トラブルシュート：クラスター再作成後の CNPG WAL "Expected empty archive" エラー原因調査 |
| 5 | CNPG バックアップ・リストア設計の整理（bootstrap イミュータブル制約・GitOps と DR 手順の分離・ignoreDifferences・PVC Retain） |

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

DR 実行後に ArgoCD が `spec.bootstrap` の差分を検知して修正しようとするのを防ぐため、対象の ArgoCD Application に以下を設定する（**未実装**）。

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

DR クラスター（`recovery`）でリストア完了後、DR クラスターを削除して GitOps に戻す際、PVC を残しておくことでデータを保全する。ArgoCD が `initdb` マニフェストでクラスターを再作成したとき、CNPG は PVC 上の既存データを検知して bootstrap をスキップする（**確認・設定が必要、未実装**）。

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-infra/Makefile` | cluster-start: `cilium status --wait` → `cilium status`、kubelet 証明書削除 + `docker restart` ループ追加、末尾に `fix-coredns` 追加 |
