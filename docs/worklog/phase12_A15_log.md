# Phase12-A15 作業ログ

| # | 作業内容 |
|---|---|
| 1 | トラブルシュート：cilium status --wait フラグが存在しないため削除 |
| 2 | トラブルシュート：WSL 再起動後の kubelet TLS 証明書 IP 不一致を cluster-start で自動修正（serving-kubelet.crt 削除 + docker restart） |
| 3 | トラブルシュート：cluster-start 後に k3s がノード再登録で CoreDNS NodeHosts を上書きし host.k3d.internal が消える問題を特定。cluster-start 末尾に fix-coredns を追加 |
| 4 | トラブルシュート：クラスター再作成後に CNPG WAL アーカイブが "Expected empty archive" で失敗する原因調査 |
| 5 | CNPG バックアップ・リストア設計の整理（bootstrap イミュータブル制約・GitOps と DR 手順の分離・ignoreDifferences・PVC Retain の本番設計論） |
| 6 | DR 設計書骨子を platform-docs/docs/design/DR-design.md として新規作成（design/ ディレクトリ新設） |
| 7 | PROJECT.md に CNPG DR 実装タスク 3 点・制約事項を追記 |
