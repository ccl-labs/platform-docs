# Phase 11-5 作業ログ


## スコープ
- sample-backend DB リトライロジック実装
- gh CLI 導入（mise 経由）
- CoreDNS host.k3d.internal 登録（bootstrap 組み込み）

---

## 1. 事前確認

```bash
# コード構成確認
find ~/sample-backend -type f -name "*.py" | sort
cat ~/sample-backend/src/requirements.txt

# コード内容確認
cat ~/sample-backend/src/main.py

# クラスタ状態確認
kubectl get nodes
kubectl get pods -n sample-app
```

---

## 2. tenacity の追加

```bash
cd ~/sample-backend
echo "tenacity==9.1.2" >> src/requirements.txt
cat src/requirements.txt  # 追記確認
```

---

## 3. main.py の更新

```bash
# main.py を丸ごと置き換え（コネクションプール・リトライ・/health DB ping）
cat > ~/sample-backend/src/main.py << 'EOF'
# （省略：コネクションプール・tenacityリトライ・/health DB ping 追加版）
EOF

# 差分確認
git -C ~/sample-backend diff src/main.py | cat
```

**変更内容：**
- `get_conn()` を廃止 → `asyncpg.create_pool()` によるプール共有に変更
- `db_retry` デコレータ追加（最大5回・指数バックオフ 1〜16秒）
- リトライ対象：`PostgresConnectionError` / `TooManyConnectionsError` / `OSError`
- `/health` に `SELECT 1` による DB ping を追加（readinessProbe 対応）
- `shutdown` イベントでプールのクローズを追加

---

## 4. gh CLI の導入

```bash
# sample-backend .mise.toml に追加
cd ~/sample-backend
cat >> .mise.toml << 'EOF'
gh      = "2.87.2"
EOF

mise install
gh --version

# GitHub 認証（PAT 方式）
gh auth login --git-protocol https --hostname github.com
# → "Paste an authentication token" を選択
# → repo / read:org / workflow スコープの PAT を貼り付け
# → Logged in as okccl を確認

# 他リポジトリにも追加
echo 'gh      = "2.87.2"' >> ~/platform-infra/.mise.toml
echo 'gh      = "2.87.2"' >> ~/platform-gitops/.mise.toml

# 追記確認
grep gh ~/platform-infra/.mise.toml ~/platform-gitops/.mise.toml ~/sample-backend/.mise.toml
```

---

## 5. commit & push

```bash
# platform-infra
cd ~/platform-infra
git add .mise.toml
git commit -m "chore: gh 2.87.2 を mise に追加"
git push origin main

# platform-gitops
cd ~/platform-gitops
git add .mise.toml
git commit -m "chore: gh 2.87.2 を mise に追加"
git push origin main

# sample-backend
cd ~/sample-backend
git add .mise.toml src/main.py src/requirements.txt
git commit -m "feat: DBコネクションプール導入・tenacityリトライ追加"
git push origin main
```

---

## 6. CI / CD 確認

```bash
# CI 状況確認
gh run list --repo okccl/sample-backend --limit 5

# CI 詳細確認
gh run view 25195478529 --repo okccl/sample-backend

# platform-gitops への自動コミット確認
gh run list --repo okccl/platform-gitops --limit 5
git -C ~/platform-gitops pull
git -C ~/platform-gitops log --oneline -5
```

---

## 7. Rollout promote

```bash
# デプロイ状態確認（canary pause 中だったため）
kubectl argo rollouts get rollout sample-backend -n sample-app

# promote して 100% に切り替え
kubectl argo rollouts promote sample-backend -n sample-app

# 完了確認
kubectl argo rollouts get rollout sample-backend -n sample-app --watch
```

---

## 8. フェイルオーバー検証（途中で CoreDNS 問題を発見）

```bash
# DB primary 確認
kubectl get cluster sample-backend-db -n sample-app -o jsonpath='{.status.currentPrimary}'
# → sample-backend-db-1

# primary Pod 削除（フェイルオーバー発生）
kubectl delete pod sample-backend-db-1 -n sample-app

# db-1 のログ確認 → Could not connect to the endpoint URL: host.k3d.internal:9000
kubectl logs sample-backend-db-1 -n sample-app | tail -30
```

---

## 9. CoreDNS host.k3d.internal 修正

```bash
# クラスタ内から MinIO への疎通確認（直接 IP で）
kubectl run minio-check --rm -it --restart=Never \
  --image=curlimages/curl:8.12.1 -n sample-app \
  --overrides='{"spec":{"containers":[{"name":"minio-check","image":"curlimages/curl:8.12.1","args":["-v","http://172.19.0.1:9000/minio/health/live"],"resources":{"limits":{"cpu":"100m","memory":"64Mi"}}}]}}'
# → 200 OK 確認

# k3d ネットワークのゲートウェイ IP 確認
docker network inspect k3d-dev | grep -A5 '"Gateway"'
# → 172.19.0.1

# CoreDNS ConfigMap に host.k3d.internal を追加
kubectl patch configmap coredns -n kube-system --type merge -p '
{
  "data": {
    "NodeHosts": "172.19.0.2 k3d-dev-agent-1\n172.19.0.3 k3d-dev-agent-0\n172.19.0.4 k3d-dev-agent-2\n172.19.0.5 k3d-dev-server-0\n172.19.0.1 host.k3d.internal\n"
  }
}'

# CoreDNS 再起動
kubectl rollout restart deployment coredns -n kube-system
kubectl rollout status deployment coredns -n kube-system

# 疎通再確認（host.k3d.internal で）
kubectl run minio-check --rm -it --restart=Never \
  --image=curlimages/curl:8.12.1 -n sample-app \
  --overrides='{"spec":{"containers":[{"name":"minio-check","image":"curlimages/curl:8.12.1","args":["-v","http://host.k3d.internal:9000/minio/health/live"],"resources":{"limits":{"cpu":"100m","memory":"64Mi"}}}]}}'
# → 200 OK 確認
```

---

## 10. db-1 復旧（WAL empty チェック失敗への対処）

CoreDNS 修正後も `Expected empty archive` エラーで db-1 が Ready にならなかったため、PVC を削除して再クローン。

```bash
# db-1 の Pod と PVC を削除
kubectl delete pod sample-backend-db-1 -n sample-app
kubectl delete pvc sample-backend-db-1 -n sample-app

# CNPG が db-3 を新規作成するのを確認
kubectl get pods -n sample-app -l cnpg.io/cluster=sample-backend-db -w
# → db-3 が Running・db-1 の孤児 Pod が残存したため追加削除
kubectl delete pod sample-backend-db-1 -n sample-app

# クラスタ状態確認
kubectl get cluster sample-backend-db -n sample-app
# → Cluster in healthy state / primary: sample-backend-db-2
```

---

## 11. 2回目フェイルオーバー検証

```bash
# Rollout を promote してから検証
kubectl argo rollouts promote sample-backend -n sample-app
kubectl argo rollouts get rollout sample-backend -n sample-app

# primary（db-2）を削除
kubectl delete pod sample-backend-db-2 -n sample-app

# フェイルオーバー進行確認
kubectl get cluster sample-backend-db -n sample-app
# → Failing over（db-2 PVC も削除が必要に）

# db-3 を強制昇格
kubectl patch cluster sample-backend-db -n sample-app \
  --type merge \
  -p '{"spec":{"primaryUpdateStrategy":"unsupervised"}}'
kubectl patch cluster sample-backend-db -n sample-app \
  --type merge \
  -p '{"status":{"targetPrimary":"sample-backend-db-3"}}'

# db-2 の PVC も削除（同様の WAL エラー）
kubectl delete pod sample-backend-db-2 -n sample-app
kubectl delete pvc sample-backend-db-2 -n sample-app

# db-4 が作成されクラスタ復旧確認
kubectl get cluster sample-backend-db -n sample-app
# → Cluster in healthy state / primary: sample-backend-db-3
```

---

## 12. 動作確認

```bash
# Pod 内から確認（curl がないため python3 を使用）
kubectl exec -n sample-app sample-backend-58446c998-bxmb5 -- \
  python3 -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8000/health').read())"
# → b'{"status":"ok"}'

# 外部から確認（HTTP）
curl -s http://sample-backend.localhost/items
curl -s -X POST http://sample-backend.localhost/items \
  -H "Content-Type: application/json" \
  -d '{"name": "phase11-5-test"}'
# → GET: [] / POST: {"id":1,"name":"phase11-5-test",...}
```

---

## 13. bootstrap への CoreDNS 修正組み込み

```bash
# fix-coredns ターゲットを Makefile に追加
cd ~/platform-infra
cat >> k3d/Makefile << 'EOF'

fix-coredns:
        $(eval K3D_GW := $(shell docker network inspect k3d-$(CLUSTER_NAME) \
                --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'))
        kubectl get configmap coredns -n kube-system -o json \
                | python3 -c "import sys,json; \
                  d=json.load(sys.stdin); \
                  nh=d['data']['NodeHosts']; \
                  entry='$(K3D_GW) host.k3d.internal\n'; \
                  d['data']['NodeHosts']=nh if 'host.k3d.internal' in nh else nh+entry; \
                  print(json.dumps(d))" \
                | kubectl apply -f -
        kubectl rollout restart deployment coredns -n kube-system
        kubectl rollout status deployment coredns -n kube-system
EOF

# bootstrap の依存に fix-coredns を追加
sed -i 's/^bootstrap: cluster-create bootstrap-argocd bootstrap-sync/bootstrap: cluster-create fix-coredns bootstrap-argocd bootstrap-sync/' k3d/Makefile

# 確認
grep -A1 "^bootstrap:" k3d/Makefile
grep -A10 "^fix-coredns:" k3d/Makefile

# commit & push
git add k3d/Makefile
git commit -m "fix: bootstrap に CoreDNS host.k3d.internal 登録を追加"
git push origin main
```

---

## 14. README 更新・commit

```bash
# platform-infra README を更新してコピー後
git -C ~/platform-infra add README.md
git -C ~/platform-infra commit -m "docs: Phase 11-5 の内容を README に追記"
git -C ~/platform-infra push origin main

# platform-gitops README を更新してコピー後
git -C ~/platform-gitops add README.md
git -C ~/platform-gitops commit -m "docs: Phase 11-5 の内容を README に追記"
git -C ~/platform-gitops push origin main
```

---

## 補足メモ（セッション中の質疑）

### Q. これまでなぜ WAL エラーが問題にならなかったのか？

WAL アーカイブの `Expected empty archive` エラーは **旧 primary が replica に降格するとき**に発生する。フェイルオーバーを経験したことがなかったため、今回の検証で初めて顕在化した。通常稼働時は primary のまま動き続けるため DNS 解決問題も表面化しない。

### Q. CoreDNS の設定は ArgoCD で管理すべきか？

**管理しない**のが正解。理由は2つ。
1. CoreDNS ConfigMap は `objectset.rio.cattle.io` アノテーションで k3s が管理しており、ArgoCD が上書きすると次の k3s reconcile で競合する。
2. `host.k3d.internal` の IP はクラスタ再作成で変わる可能性があるため静的な GitOps 管理に向かない。
→ `make bootstrap` への組み込みで冪等性を担保した。

### Q. CNPG 旧 primary の PVC が壊れてデータが取り出せない場合の対処は？

職場での CrunchyDB 障害経験を踏まえた議論。試みるべき手順は以下の通り。
1. **PVC を別 Pod にマウント**して直接データにアクセス（kubectl exec でシェルを起動）
2. **pg_resetwal** で WAL をリセットして PostgreSQL を強制起動（未コミットトランザクションは失われる）
3. **pg_dump** でデータをエクスポートし、稼働中インスタンスに再インポートして差分回収

ただし `pg_resetwal` が途中でエラーになるケース（WAL ファイル自体が壊れている場合）もあり、その場合はデータ回収が困難になる。アプリログからの SQL 再現・再投入という手段も現実的な選択肢のひとつ。

### Q. PR 方式で image tag を更新すべきか？

現状は `Update GitOps` workflow が直接 main に push する設計。個人開発では手間だが、ポートフォリオとしての見栄えと GitOps の原則（変更履歴をすべて Git に残す）を考えると PR 方式が望ましい。Phase 11 完了後の番外編として対応予定。

### Q. HTTPS（443）で curl がレスポンスを返さない件

TLS ハンドシェイクが `SSL_ERROR_SYSCALL` で失敗している。HTTP（80）では正常にレスポンスが返るため、Envoy Gateway の TLS 設定または cert-manager の証明書の問題と思われる。Phase 11-5 のスコープ外のため持ち越し。Phase 11 後半の TLS 対応（cert-manager 連携）で解決予定。
