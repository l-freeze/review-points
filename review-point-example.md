# レビュー判断の領域別具体例

## 0. この文書の役割

この文書は `review-point.md` を適用するための補助資料であり、レビュー項目の網羅的な catalog ではない。対象の領域名が本文にあるという理由だけで参照したり、例に一致するという理由だけで finding を成立させたりしてはならない。

次の場合に限り参照する。

- `review-point.md`の原則だけでは、対象の境界、状態、正本、failure を具体化できない
- 候補を `finding`、`verification item`、`decision required`、`non-issue` のどれへ分類するか迷う
- 必要な保証と過剰な保証の境界を、類似構造から検討する必要がある
- 利用者が領域別の具体例を求めた

参照前に、解決したい問いを一文で定める。必要な例を確認したら `review-point.md`へ戻り、対象自身の契約と事実によって結論を出す。

この文書は、次の目的には使用しない。

- 例を上から順に全件照合する
- finding の件数を増やす
- 対象の正本文書にない要件、threat、対応環境を追加する
- 例の修正方法をそのまま対象へ移植する
- 例にない問題を scope 外とみなす

具体例と `review-point.md`、対象の正本文書が衝突する場合、具体例を捨てる。類似性は、技術名ではなく、責務、境界、状態遷移、failure 後の事後条件によって判断する。

## 1. 状態分類と error handling

### 1.1 設定が存在しない場合

設定 file が存在せず、仕様が documented default を定めているなら、default の使用は正式な正常経路になり得る。この場合、fallback という名前だけを理由に問題視しない。

一方、parse error、permission error、unknown schema を「存在しない」へ丸めて同じ default を使うと、安全制限や利用者設定を意図せず解除する可能性がある。呼出側の処置が異なるなら状態を区別する。

判断に必要な問い:

- default は正式な契約か
- 読取 failure と不存在を識別できるか
- default によって permission、scope、保持期間などが広がらないか
- error が直ちに分かり、利用者が回復できるか

### 1.2 空結果と dependency failure

検索結果が0件であることと、検索 service が timeout したことは通常異なる。後者を空配列へ変換すると、利用者は「存在しない」と誤認する可能性がある。

ただし、補助的な recommendation の failure を空表示として扱うことが明示契約であり、主要結果、診断、再試行可能性を損なわないなら、同じ UI 表現が許容される場合もある。

### 1.3 広い catch

広い `catch` が常に問題なのではない。process boundary で例外を統一 error response へ変換し、内部では分類、cause、retryability を保持するなら妥当になり得る。

parse error、permission error、not found、outcome unknown を同じ成功値へ変換し、呼出側が異なる処置を選べなくなるなら問題候補である。

## 2. 認証、認可、所有権、同意

### 2.1 認証 service の failure

認証必須操作で認証 service が応答しないとき、匿名または過去の権限へ自動的に落とすと trust boundary を迂回する。合意した契約が本人性を要求するなら処理を停止する。

元から匿名利用を正式対応している公開読取は、別契約として継続できる。認証 failure 時だけ匿名契約を新設してはならない。

### 2.2 UI で操作を隠す

UI で button を隠すことは操作支援にはなるが、server-side authorization の代わりにはならない。利用者が request を直接送れるなら、信頼する側が permission を確認する。

閉じた desktop application などで server boundary が存在しない場合でも、保護対象、攻撃者、local user の権限を threat model で確認する。Web の例をそのまま適用して不要な server 構成を要求しない。

### 2.3 permission の cache

短時間の cache が常に誤りではない。失効反映時間、操作の重大性、cache key の identity・tenant・resource、強制再確認が必要な操作を契約で決める。

即時失効が必要な破壊操作で、古い cache だけを正本として許可するなら finding になり得る。表示用の低risk情報で、契約が短い遅延を許すなら non-issue になり得る。

### 2.4 consent

説明文や prompt が「利用者の同意を得る」と書いていても、mutation が先に実行されるなら gate ではない。必要な同意は、対象、scope、時点を識別し、副作用前に確認できる経路へ置く。

すべての操作へ確認 dialog を要求してはならない。明示的な操作自体が consent として契約されている場合や、可逆で低riskな操作では追加確認が過剰になることがある。

## 3. fallback、failover、縮退運転

### 3.1 primary database から secondary への切替

primary の結果が不明なまま secondary へ write すると、正本分裂や二重 commit が起こり得る。同期方式、generation、昇格手順、書込排他、復帰条件が定義された正式な failover か確認する。

read-only query で、許容する stale 期間が契約され、利用者に整合性水準が明示されるなら replica read は妥当な縮退になり得る。

### 3.2 外部 provider の切替

通知や決済が timeout した直後に別 provider へ送ると、元の処理が成功していた場合に重複する。結果照会、共通 operation ID、idempotency、deduplication がなければ `outcome unknown` として確認を待つ。

画像取得など副作用のない処理では、同じ data policy、利用権、品質契約を満たす代替 provider への retry が妥当な場合がある。副作用の有無と必要な postcondition を確認する。

### 3.3 rate limit、fraud check、audit の failure

これらが合意した安全契約の一部なら、failure 時に無制限または記録なしで続けてはならない。

ただし、すべての telemetry failure が主要操作を止めるとは限らない。監査必須記録と best-effort metrics を区別し、同じ failure policy を機械的に適用しない。

### 3.4 degraded mode

縮退運転では、下げる保証、維持する保証、対象 scope、開始条件、期限、表示、監視、停止条件、復帰条件を明示する。通常成功と同じ status や表示へ隠すと、利用者と運用者が誤判断する。

## 4. transaction、永続化、file 更新の確認項目例

この列挙は、`review-point.md`の原則だけでは判断を具体化できない場合に限り使用する。

- authoritative journal または manifest は一つに定まるか
- lock 取得前の snapshot を mutation 判断へ使っていないか
- staging と backup は atomic operation に必要な filesystem 条件を満たすか
- commit 直前に対象 identity、hash、generation を再確認するか
- unfinished journal がある間、reader が mixed state を成功扱いしないか
- rollback 不能時は推測で続行せず `recovery-required` になるか
- rollback は利用者が変更した非管理領域を巻き戻さないか
- uninstall と cleanup は所有を確認できるものだけを削除するか
- atomic rename を使えない環境への copy fallback が、中断時の曖昧な状態を増やさないか
- 各 target-local atomic operation と一つの正本で契約を満たせるのに、secondary receipt や root 別 journal を追加していないか
- 外部適用を観測できないのに、receipt や phase 名だけで完了を保証したことにしていないか
- sentinel を保護対象より先に消していないか
- timeout や expiry を、外部処理や lock owner の終了証明として扱っていないか

## 5. API、外部副作用、dependency

### 5.1 timeout と cancel

client が timeout または cancel しても、server の副作用が完了している場合がある。未実行、failure、success、outcome unknown のどれかを契約で区別する。

副作用のない request なら単純な retry でよい場合がある。副作用があるなら idempotency key、結果照会、deduplication の必要性を確認する。

### 5.2 optional field

field の欠落が documented default を意味するなら妥当である。不正値、parse failure、unknown version を欠落と同じ扱いにすると、意図しない default が選ばれる可能性がある。

### 5.3 pagination、ordering、consistency

順序や一貫性が利用者の目的に影響する場合は契約を明示する。単なる表示用一覧へ強い global ordering や linearizable consistency を要求することが常に正しいわけではない。

### 5.4 retry と circuit breaker

client、gateway、service、queue がそれぞれ retry すると、同じ障害を増幅する可能性がある。retry の所有者、回数、backoff、idempotency、deadline を確認する。

circuit breaker があるという理由だけで安全とは判断しない。authentication や permission failure を成功へ変換していないか、open 中の契約を確認する。

### 5.5 beta または undocumented behavior

必要な安全境界や主要契約を、安定性が保証されない behavior へ依存させない。早期 feasibility test、非対応化、version pin、監視などから必要十分な対応を選ぶ。

補助機能が beta API を使うこと自体を finding にしない。failure が主要目的や境界へ伝播するかを確認する。

## 6. data、schema、migration

### 6.1 unknown schema

unknown schema を旧 version と推測して mutation すると正本を壊し得る。対応 version ごとの明示変換と round-trip test を用意し、対応外は mutation 前に拒否する。

best-effort import が明示要件なら、隔離、preview、非破壊性、利用者確認、失敗 report の契約を設ける。best effort という名前だけで拒否しない。

### 6.2 mixed-version deployment

migration 中に旧 code と新 code が同時稼働するなら、双方が読書きする schema、切替順序、rollback 条件を確認する。単一processを停止して一括移行する製品へ、不要な mixed-version protocol を要求しない。

### 6.3 cache と derived data

cache や再生成可能な derived data を正本として mutation しない。再生成可能性、許容 stale 期間、invalidaton の所有者を確認する。

cache failure 時に元の正本から読めるなら正式な fallback になり得る。正本 failure を古い cache で隠す場合は、契約と利用者への表示を確認する。

### 6.4 delete、retention、audit、privacy

soft delete、retention、audit、privacy deletion は目的が異なる。どの義務が優先し、誰が削除でき、backup や derived data へいつ反映するかを定める。

## 7. state、concurrency、idempotency

### 7.1 check-then-act

存在確認後に別処理で mutation する間に対象が変わるなら、lock、transaction、compare-and-swap、generation check など同じ契約を満たす境界へ閉じる。

変更不能な build artifact を一度確認して読む処理へ、理論上の race だけを理由に複雑な排他を要求しない。攻撃者、変更主体、影響を threat model で確認する。

### 7.2 idempotent の意味

同じ response を返すだけでなく、同じ論理 operation の再実行で外部副作用を重複させないことが必要か確認する。操作 ID の範囲、保持期間、競合中の response を定める。

pure read のように元から副作用がない処理へ、永続的な idempotency record を要求しない。

### 7.3 cancel と undo

cancel が「まだ commit していない変更を破棄する」のか、「完了済み操作を補償する」のかを区別する。undo が新しい mutation なら、権限、競合、現在状態を再確認する。

## 8. UI、client、操作設計

### 8.1 局所操作と共通 controller

一つの gesture のために画面全体へ汎用 controller を導入する必要は通常ない。局所 lifecycle を局所へ閉じ、編集規則を pure function として test できる。

複数 gesture の一貫性、accessibility、共通の取消契約、複数画面での実利用が既にあるなら共通化が妥当な場合もある。一度の重複だけで判断しない。

### 8.2 client-side validation

client-side validation は入力支援であり、server-side validation は trust boundary の防御である。同じ条件を両方で確認しても責務が異なるなら重複 guard とは限らない。

同一 process の隣接関数が同じ内部前提を同じ理由で再確認するだけなら、規則の正本が分散していないか確認する。

### 8.3 操作中の対象変更

長い編集操作や破壊的な確定では、開始時と確定時の対象 identity、version、generation が一致するか確認する。単純なhoverや可逆な表示変更へ同じ防御を要求しない。

## 9. AI、agent、automation

### 9.1 prompt と機械的な gate

prompt の「実行前に確認せよ」は model の行動を誘導するが、permission や consent の唯一の gate にはならない。不可逆操作や機密 data access は、tool または host 側で現在の権限と承認を強制する。

一方、文章生成の style や可逆な提案内容まで機械的に完全制約する必要はない。誤った場合の影響と回復可能性で境界を決める。

### 9.2 model、client、worker の自己申告

model や worker が root、tenant、scope、generation を自己申告しても、それだけを正本として信頼しない。信頼する host が現在の session、resource、permission と結び付ける。

内部 pipeline で署名済み metadata が一つの trusted boundary から渡される場合まで、各段へ同じ外部認証を要求しない。どこで信頼が確立し、どこまで伝播するかを確認する。

### 9.3 nondeterminism と evaluation

生成結果が非決定的なら、単一 snapshot の完全一致だけで品質を保証したことにしない。利用目的に対応する受入条件、代表 case、回帰評価、必要な human review を定める。

すべての生成機能へ大規模 benchmark を要求してはならない。影響、利用頻度、失敗の検出可能性に比例させる。

### 9.4 tool failure と自動 fallback

tool が失敗したとき、権限の弱い別経路や未検証の推測結果へ自動的に切り替えない。明示 error、retry、別 tool の同等契約、利用者確認のどれが要件を満たすか判断する。

### 9.5 context、receipt、完了判定

model が context を実際に利用したか外部から観測できない場合、receipt file を作っても利用の証明にはならない。観測できる送信成功、host acceptance、出力評価を区別する。

## 10. 非software領域への適用

### 10.1 組織間 handoff

担当者Aから担当者Bへの引継ぎで、承認者、正本文書、完了条件、未完了状態が不明なら、software の boundary・正本・状態分類と同じ構造の問題である。

引継ぎ checklist を増やす前に、誰が決定し、どの記録が正本で、受領を何によって確認するかを定める。

### 10.2 manual と実際の運用

manual に「必ず承認する」と書いてあっても、承認なしで実行でき、監査もされないなら実効的な gate ではない。必要な保証なら workflow または権限境界で強制する。

低riskで例外判断が多い業務へ、機械的な全面停止を導入することが正しいとは限らない。責任者、記録、事後確認で契約を満たせるか評価する。

### 10.3 外部委託と dependency

外部委託先の failure を「完了」として利用者へ返してはならない。委託範囲、受入条件、結果不明時の照会、再依頼の重複防止を定める。
