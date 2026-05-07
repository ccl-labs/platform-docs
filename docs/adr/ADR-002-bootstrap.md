# ADR-002: bootstrap 順序制御の設計（App-of-Apps 分割と Makefile による明示的制御）

## Context

プラットフォームの bootstrap では `cert-manager → Envoy Gateway → Keycloak → Backstage`
という明確な起動順序があり、前段が Ready になる前に後段を sync すると
webhook 未登録・namespace 未作成などのエラーが発生する。

当初は2段階の App-of-Apps と `sync-wave` アノテーションで順序制御を試みたが、
想定通りに動かないケースが発生した。

原因を調査した結果、ArgoCD の **カスタムリソース（CR）に対するデフォルトの health check** の
挙動に起因することがわかった。ArgoCD は Deployment などの標準リソースについては
実際に Ready になるまで次の wave へ進まないが、CR については
**apply されてクラスター上に存在する時点で Healthy と判定する**。
そのため、cert-manager や CNPG の Operator が wave 1 で apply された直後に
wave 2 が開始され、webhook サーバがまだ起動していない状態で
次のリソースが apply されてエラーになっていた。

この問題は CR ごとに Lua スクリプトで health check を定義することで回避できるが、
管理対象のカスタムリソースが多く、定義・保守のコストが高い。
加えて、`root → group-roots → applications` の2段階構造は wave 制御のために設けたものであり、
「bootstrap の初回実行時にだけ順序制御が必要」という事実を踏まえると、
wave に頼るより Makefile で明示的に制御する方がシンプルかつ確実という結論に至った。

---

## Options

### Option 1: wave の設定を精緻化して順序を制御し続ける

- group-roots の wave を調整し、health check の条件を厳格化する
- **Pros:** 既存構成を大きく変えずに済む
- **Cons:** CR ごとに Lua スクリプトで health check を定義・保守する必要があり、
  管理対象のカスタムリソースが増えるほどコストが増大する。
  定義漏れがあれば wave の順序保証が崩れるため、網羅性の担保が難しい。

### Option 2: App-of-Apps を3つに分割し、Makefile で順次実行する

- root を廃止し、root-1-gateway / root-2-auth / root-3-others の
  3つの App-of-Apps を作成する
- Makefile がそれぞれを順番に apply し、完了（Synced かつ Healthy）を
  確認してから次を実行する
- 完了判定は ArgoCD の health check ではなく、重要コンポーネント
  （keycloak, keycloak-config-cli）の Pod 起動を個別に待機する
- **Pros:** bootstrap の順序保証と運用時の selfHeal を両立できる
- **Cons:** bootstrap が `make bootstrap` 経由に限定される。
  root-app が3つになり、背景を知らないと構成の意図がわかりにくい。

---

## Decision

**Option 2 を採用する。App-of-Apps を3つに分割し、Makefile で順次実行する。**

Option 1 は CR ごとの health check 定義が必要で管理コストが高く、
定義漏れによる順序保証の崩壊リスクも残る。
Option 2 は Makefile が bootstrap 時の順序保証を担い、
運用時の selfHeal による自動修復は ArgoCD に委ねることで両者を両立する。

---

## Reasons

1. **「bootstrap 時の順序保証」と「運用時の自動修復」を分離できる**

   bootstrap 時の順序制御は Makefile が担い、
   bootstrap 完了後の状態維持は ArgoCD の selfHeal が担う、
   という役割分担が明確になる。
   Option 1 では CR ごとの health check 定義を wave に組み合わせる形になり
   構造が複雑化する上、定義漏れのリスクが常に残る。

2. **ArgoCD の GUI を「最初の安全網」として保証できる**

   bootstrap の中で最優先したいのは、
   「何か問題が発生したときに ArgoCD の GUI にアクセスできる状態」を
   いち早く確保することと考えている。
   現場での経験上、障害時やハング時のトラブルシュートにおいて
   ArgoCD の GUI は CLI より圧倒的に情報が把握しやすい。
   CLI では sync 中に別の操作をしようとすると「another operation is already in progress」
   と返されるだけで、待つべきか強制終了すべきかがわかりにくい。
   GUI であれば処理がどこで詰まっているかが一目でわかり、
   TERMINATE して再実行するといった判断がすぐにできる。

   Makefile で root-1-gateway（ArgoCD の HTTPRoute を含む）を先に完了させることで、
   `https://argocd.platform.local` へのアクセスが保証された状態で
   以降の作業に進める。

3. **root の2段階構造に存在意義がなかった**

   既存の `root → group-roots → applications` という2段階の App-of-Apps は、
   wave による順序制御のために設けられていたが、
   CR の health check が正確でない以上、wave だけでは順序保証が成立しなかった。
   分割後は `root-1-gateway / root-2-auth / root-3-others` が
   直接 `applications/` を指す1段階の構造になり、見通しが改善した。

---

## Consequences

**ポジティブ:**

- bootstrap の順序制御が Makefile に集約され、挙動が予測しやすくなった
- ArgoCD の GUI へのアクセスが bootstrap の早い段階で保証される
- bootstrap 完了後は `automated: selfHeal: true` によるドリフト自動修復が維持される
- App-of-Apps の階層が1段階減り、ArgoCD GUI 上での見通しが改善した

**ネガティブ・トレードオフ:**

- **bootstrap は `make bootstrap` 経由での実行が必須。**
  ArgoCD GUI からの手動 root sync という操作は存在しない。
  → DR手順書を作成することで周知する。
- **root-app が3つ存在し、背景を知らないと構成の意図がわかりにくい。**
  → 本 ADR で設計意図を記録することで補完する。

**許容済みリスク:**

- **一部 Application（KEDA など）で bootstrap 時に初回 Sync が失敗することがある。**
  - 原因: 内部 webhook サーバの証明書自動生成完了を待たずに init pod が起動するため
  - 影響: タイムアウト後に ArgoCD が自動 retry して収束する。
  - 手動回避: Sync 停止 → 再実行でタイムアウトまでの待ち時間を短縮できる
  - 許容判断: 多少の時間はかかるが自動収束が確認できているため。また、根本解決（証明書の事前発行）は管理コストに見合わない
