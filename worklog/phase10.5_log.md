# Phase 10.5 作業ログ（DR演習）

---

## 事前確認・準備

### クラスタ状態確認
```bash
k3d cluster list
kubectl get nodes
```

### WSL再起動後のSecret復旧（演習前の準備）
```bash
sops -d ~/platform-gitops/secrets/encrypted/minio-auth.yaml | kubectl apply -f -
sops -d ~/platform-gitops/secrets/encrypted/minio-backup-secret.yaml | kubectl apply -f -
sops -d ~/platform-gitops/secrets/encrypted/ghcr-pat.yaml | kubectl apply -f -
```

### CNPGクラスタのローリングリスタート（cache miss解消）
```bash
kubectl annotate cluster -n sample-app sample-backend-db \
  kubectl.kubernetes.io/restartedAt="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --overwrite
```

---

## STEP 1: DBの現在データ確認

### DBパスワード取得・接続確認
```bash
kubectl exec -n sample-app \
  $(kubectl get pod -n sample-app -l cnpg.io/cluster -o jsonpath='{.items[0].metadata.name}') \
  -- bash -c "PGPASSWORD=$(kubectl get secret -n sample-app sample-backend-db-app -o jsonpath='{.data.password}' | base64 -d) psql -h 127.0.0.1 -U app -d app -c '\dt'"
```

### ダミーデータ投入
```bash
kubectl exec -n sample-app \
  $(kubectl get pod -n sample-app -l cnpg.io/cluster -o jsonpath='{.items[0].metadata.name}') \
  -- bash -c "PGPASSWORD=$(kubectl get secret -n sample-app sample-backend-db-app -o jsonpath='{.data.password}' | base64 -d) psql -h 127.0.0.1 -U app -d app -c \"
INSERT INTO items (name) VALUES ('DR-test-1'), ('DR-test-2'), ('DR-test-3');
SELECT * FROM items;\""
```

**基準値**: items テーブル 3件（DR-test-1〜3）

---

## STEP 2: MinIOバックアップ確認・手動バックアップ取得

### バックアップリソース確認
```bash
kubectl get backup -n sample-app
kubectl get scheduledbackup -n sample-app
```

### 手動バックアップ取得
```bash
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: dr-test-backup
  namespace: sample-app
spec:
  method: barmanObjectStore
  cluster:
    name: sample-backend-db
EOF
kubectl get backup -n sample-app -w
```

### MinIOバックアップ内容確認
```bash
kubectl exec -n minio deploy/minio -- bash -c "
  mc alias set myminio http://localhost:9000 <USER> <PASSWORD> && \
  mc ls --recursive myminio/cnpg-backup/sample-backend-db/"
```

---

## STEP 3: 外部MinIO構築（設計欠陥の発見と対処）

### 問題発覚
クラスタを消去するとクラスタ内MinIOも消え、バックアップが消滅することが判明。

### WSL上に外部MinIOコンテナを起動
```bash
mkdir -p ~/minio-data
docker run -d \
  --name minio-external \
  --restart unless-stopped \
  -p 9000:9000 \
  -p 9001:9001 \
  -v ~/minio-data:/data \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD=minioadmin123 \
  quay.io/minio/minio:latest \
  server /data --console-address ":9001"
```

### バケット作成
```bash
docker exec minio-external mc alias set local http://localhost:9000 minioadmin minioadmin123
docker exec minio-external mc mb local/cnpg-backup
```

### クラスタからの接続確認
```bash
kubectl run test-minio --rm -it --restart=Never \
  --image=curlimages/curl:8.5.0 \
  --overrides='{"spec":{"containers":[{"name":"test-minio","image":"curlimages/curl:8.5.0","command":["curl","-s","-o","/dev/null","-w","%{http_code}","http://host.k3d.internal:9000/minio/health/live"],"resources":{"limits":{"cpu":"100m","memory":"64Mi"}}}]}}' \
  -n default 2>/dev/null
```

### platform-gitopsのCNPG endpointURL変更
```bash
sed -i 's|endpointURL: "http://minio.minio.svc.cluster.local:9000"|endpointURL: "http://host.k3d.internal:9000"|' \
  ~/platform-gitops/apps/sample-backend/values.yaml
```

### クラスタ内MinIOのApplication削除
```bash
rm ~/platform-gitops/platform/minio/application.yaml
```

---

## STEP 4: DR演習本番（クラスタ消去→復旧）

### ダミーデータ投入（外部MinIO移行後）
```bash
kubectl exec -n sample-app \
  $(kubectl get pod -n sample-app -l cnpg.io/cluster -o jsonpath='{.items[0].metadata.name}') \
  -- bash -c "PGPASSWORD=$(kubectl get secret -n sample-app sample-backend-db-app -o jsonpath='{.data.password}' | base64 -d) psql -h 127.0.0.1 -U app -d app -c \"
INSERT INTO items (name) VALUES ('DR-test-1'), ('DR-test-2'), ('DR-test-3');
SELECT * FROM items;\""
```

### 最終バックアップ取得
```bash
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: dr-test-backup-final
  namespace: sample-app
spec:
  method: barmanObjectStore
  cluster:
    name: sample-backend-db
EOF
kubectl get backup -n sample-app -w
```

### RTO②計測スクリプト起動
```bash
(
  while true; do
    NOT_READY=$(argocd app list 2>/dev/null | grep -v "^NAME" | grep -v "Synced.*Healthy" | wc -l)
    if [ "$NOT_READY" -eq 0 ]; then
      echo "=== 全App復旧完了: $(date '+%H:%M:%S') ==="
      break
    fi
    sleep 15
  done
) &
```

### bootstrap実行（RTO①計測）
```bash
time make bootstrap
```

---

## Makefile修正内容

### SOPS_AGE_KEY の ~ 展開バグ修正
```bash
sed -i 's|SOPS_AGE_KEY.*:=.*~/.config/sops/age/keys.txt|SOPS_AGE_KEY    := $(HOME)/.config/sops/age/keys.txt|' Makefile
```

### SSHエージェント自動起動追加
```bash
# bootstrap-sync内のリポジトリ登録前に追加
eval $$(ssh-agent -s) && ssh-add $(SSH_KEY)
```

### root app syncの改善（refresh方式）
```bash
argocd app get root --refresh > /dev/null
argocd app sync root --server-side --async || true
```

### ingress-nginx待機をdeployment存在確認方式に変更
```bash
@until kubectl get deployment ingress-nginx-controller -n ingress-nginx 2>/dev/null; do echo "ingress-nginx待機..."; sleep 5; done
kubectl wait --for=condition=available deployment/ingress-nginx-controller \
    -n ingress-nginx --timeout=600s
```

### bootstrap完了時にパスワード表示
```bash
@echo "=== bootstrap完了 ==="
@echo "ArgoCD URL: https://argocd.localhost"
@echo "ArgoCD 初期パスワード: $$(argocd admin initial-password -n $(ARGOCD_NS) | head -1)"
```

---

## platform-gitops構成変更

### sync-wave追加（Application一覧）

| ファイル | wave |
|---|---|
| cert-manager-app.yaml | 1 |
| secrets/application.yaml | 1 |
| policy/kyverno.yaml | 1 |
| ingress/application.yaml | 2 |
| ingress/cert-manager-config-app.yaml | 4 |
| secrets/external-secrets-config-app.yaml | 4 |
| argocd/application.yaml | 3 |
| goldilocks/application.yaml | 3 |
| monitoring/application.yaml | 3 |
| cnpg/application.yaml | 4 |
| logging/loki-application.yaml | 2 |
| logging/alloy-application.yaml | 2 |
| tracing/tempo-application.yaml | 2 |
| vpa/application.yaml | 2 |
| policy/kyverno-policies.yaml | 3 |
| apps/sample-backend.yaml | 4 |
| apps/sample-frontend.yaml | 4 |

### CRD依存リソースの分離
```bash
mkdir -p platform/ingress/config
mkdir -p platform/secrets/config
mv platform/ingress/cluster-issuer.yaml platform/ingress/config/
mv platform/secrets/cluster-secret-store.yaml platform/secrets/config/
mv platform/secrets/external-secrets/ghcr-pull-secret.yaml platform/secrets/config/
```

### root appのexclude追加
```yaml
exclude: "{policy/policies/*,ingress/config/*,secrets/config/*}"
```

### cluster-issuer.yaml内のsync-wave設定
- selfsigned-issuer: wave 3
- local-ca Certificate: wave 3
- local-ca-issuer: wave 4

---

## RTO計測結果

| 指標 | 時間 |
|---|---|
| RTO① `make bootstrap` 完了 | **7分37秒** |
| RTO② 全App Synced/Healthy | **15分24秒** |
| 手動作業 | Ageキーのコピーのみ（新規マシンの場合） |
