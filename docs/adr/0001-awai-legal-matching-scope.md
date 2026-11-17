# ADR 0001: Awai の法律相談・弁護士マッチング（bot profile と Awai app）の範囲と規制の境界

- 状態: **提案（下書き）**。法的判断が要る項目はすべて「Owner 決定待ち」として未決のまま残す。
- 日付: 2026-11-18
- 追跡: Linear AWAI-3232（親 AWAI-3230）
- 後続: AWAI-3233（弁護士マッチング・書類支援の bot profile）、AWAI-3235（法律事務所業務の bot profile）、
  AWAI-3236（Tauri v3 Awai app）、AWAI-3238（Awai Android OS での受け入れ）

この文書は法的助言ではない。ここで「採る」と書くものは、この repo の法域カタログ
（`src/legalsupport/facts.cljk`）と governor がすでに実装している判定を、Awai の bot profile と
app にそのまま適用するという設計上の選択に限る。カタログにない法的判断はこの ADR では決めない。

## 背景

AWAI-3230 は、Itonami の bot profile が法律事務所の業務と、利用者と弁護士とのマッチングを行い、
利用者と弁護士が対応する Awai app（Tauri v3）を Awai Android OS 上で使うことを目標にしている。

この repo はその業務ロジックの土台になる。LegalSupportAdvisor ⊣ LegalSupportGovernor は、
法域カタログの判定を毎回の実行で hard check にし、紹介料を全法域で拒否し、
弁護士の精査を要する出力では「その法域の資格を持ち、自ら精査・修正した弁護士」の記録を要求する。

日本（JPN）についてカタログが今言っていること（2026-07-26 取得の一次資料に基づく）:

| 項目 | 判定 | 根拠（rule id） |
|---|---|---|
| `:mode/template-selection` | admissible（条件つき） | jpn.moj-ai-contract-guideline-2023 §3(1)イ |
| `:mode/document-assembly` | admissible（条件つき） | 同上 |
| `:mode/legal-analysis` | conditional | 同 §3、jpn.bengoshi-ho-72 |
| `:mode/lawyer-reviewed` | admissible | 同 §4(1)（弁護士が自ら精査・修正） |
| `:mode/inhouse-reviewed` | admissible | 同 §4(2) |
| `:mode/lawyer-directory` | **unsettled**（未検証） | なし。日弁連の業務広告規程と 72 条「周旋」を原典で未検証 |
| `:mode/referral-placement` | **prohibited** | jpn.bengoshi-ho-72（周旋を業とすること） |
| `:revenue/law-firm-saas-license` | admissible | ガイドライン §4(1) |
| `:revenue/none` | admissible（誘導・連動がない場合） | ガイドライン §1(1) |
| `:revenue/consumer-document-fee` | conditional | ガイドライン、jpn.gyoseishoshi-ho |
| `:revenue/consumer-subscription` | conditional | ガイドライン §1(2)ウ |
| `:revenue/directory-listing-flat` | unsettled | なし |
| `:revenue/per-referral-fee`・`:revenue/success-fee-share` | prohibited | 72 条、ガイドライン §1(2)イ |

**つまり、日本で「弁護士とのマッチング」として想像されやすい形（個別の相談を特定の弁護士へ取り次ぐ）は、
カタログ上すでに prohibited であり、弁護士の一覧を見せる形も unsettled である。**
この ADR はその判定を変えない。

## 決めること（設計上の選択。カタログの判定をそのまま使う）

### D1. bot がする範囲

- bot profile と app は、カタログが admissible とする service mode の中でだけ動く。
  JPN では template-selection・document-assembly、および弁護士が自ら精査・修正する
  lawyer-reviewed / inhouse-reviewed に限る。
- 個別事案に対する法的リスクの表示や修正案の提示（legal-analysis）は、JPN では lawyer-reviewed に
  載せない限り出さない。governor の `:admissibility` hold をそのまま使い、bot 側で緩めない。
- `:effect` は常に `:propose`。bot と app は提出・送達・支払いをしない（既存の hard check）。
- 紛争が生じている案件（事件性）は、どの mode でも資格確認済みの弁護士の精査なしに扱わない（既存の hard check）。

### D2. 紹介料

- `:revenue/per-referral-fee` と `:revenue/success-fee-share` は全法域で拒否する（既存の
  `forbidden-revenue-modes`）。Awai の経済圏（saifu・x402 など）に接続する場合も、
  弁護士への送客や受任に連動する支払いの経路は作らない。
- 収益 mode は store の operator 設定であり、advisor（LLM）や bot は選べない（既存の設計）。

### D3. 「マッチング」の扱い

- JPN では `:mode/referral-placement` を実装しない（カタログで prohibited）。
- JPN の `:mode/lawyer-directory` は、カタログで admissible になるまで実装しない（unsettled）。
- それまでの間、app が JPN で提供できるのは、**利用者が自分で選んだ弁護士へ、利用者自身の操作で
  相談資料を渡す**ことだけ、という形を候補とする。この形が 72 条の「周旋」に当たらないかは
  カタログ上も未検証であり、**Owner 決定待ち（O1）**。決まるまで AWAI-3236 はこの操作を
  descriptor に宣言しない（宣言のない操作は fail closed）。
- 弁護士への紹介は、どの法域でも人の sign-off を必須にする（既存の ESCALATE）。

### D4. 法律事務所側の業務

- 日本で最も確実な経路は、弁護士・弁護士法人が自ら精査・修正する方法で使うツールとしての提供
  （`:revenue/law-firm-saas-license`、ガイドライン §4(1)）である。AWAI-3235 の法律事務所業務の
  bot profile は、この前提で事務所内のツールとして設計する。
- 案件・顧客・利益相反確認の業務ロジックは、事務所自身の案件管理を扱う
  `cloud-itonami-isco-2611` との分担を AWAI-3235 で決める。旧 `lawfirm-admin` / `lawyer` の
  Cloudflare Worker と外部 MCP router への転送は移さない。

### D5. 弁護士の資格確認

- 精査する弁護士は、`cloud-itonami-licensed-operator` の検査
  （`:req/licence-verified`・`:req/same-jurisdiction`・`:req/personally-decided`）を通す（既存）。
- 資格を Awai の credential として持つ形は AWAI-344（資格者モデル）に合わせる。
- 何をもって「資格確認済み」とするか（登録番号の照合元、確認の頻度、懲戒・登録抹消の反映）は
  **Owner 決定待ち（O4）**。照合元として使える公式の API があるかは未確認（unknown）。

### D6. データの分類と保存先

| データ分類 | 例 | 保存先 |
|---|---|---|
| `legal.consultation` | 相談内容、事情の経緯、相手方の情報 | 利用者の端末（ブラウザ・app）だけ |
| `legal.document` | 生成した書類、弁護士の修正版 | 利用者または事務所の端末だけ |
| `legal.matter` / `legal.client` | 事務所の案件・顧客、利益相反確認の記録 | 事務所の端末だけ |
| `person.contact` | 利用者・弁護士の連絡先 | 端末だけ |
| `legal.audit` | governor の判定と引用、精査の記録 | 端末（共有するかは別の同意） |
| `credential.lawyer` | 弁護士の登録番号・所属会・確認結果 | 端末（形式は AWAI-344） |

- app origin は静的配布だけを行い、これらのデータを受け取らない（共有アーキテクチャ契約）。
- 守秘義務と個人情報の扱い（保存期間、削除、事務所間の受け渡し）は **Owner 決定待ち（O8）**。

### D7. 外部操作の一覧

すべて `kotoba.app.client-runtime.v1` の `each-operation` 同意を要し、承認 1 回で 1 操作だけを許す。
endpoint origin が「未定」の操作は、決まるまで descriptor に宣言しない（fail closed）。

| # | 操作 | service | endpoint origin | data categories | 状態 |
|---|---|---|---|---|---|
| X1 | 利用者が選んだ弁護士へ相談資料を送る | 未定（メール、Awai の通信など） | 未定 | `legal.consultation`、`legal.document`、`person.contact` | **O1 が決まるまで宣言しない** |
| X2 | 弁護士の資格を照合する | 未定 | 未定 | `credential.lawyer` | **O4 が決まるまで宣言しない** |
| X3 | 弁護士が修正版を利用者へ返す | X1 と同じ経路 | 未定 | `legal.document` | X1 と同じ |
| X4 | 推論（下書きの生成） | 端末内推論が既定 | なし（端末内） | `legal.consultation` | 端末内は外部操作ではない |
| X5 | リモート推論 | 利用者が選んだ推論 service | 利用者が選ぶ | `legal.consultation` | 別の新しい同意が要る。モデルの取得の同意で代えない |
| X6 | 法令・判例の参照 | 法令 app（AWAI-3132）のデータ | 法令 app の宣言に従う | なし（公開データ） | 法令 app 側で扱う |

### D8. 最初の法域

- カタログ上、法律事務所向けの製品は JPN、利用者が直接使う製品は GBR（England & Wales）が最も広い。
- Awai でどちらから始めるか、両方か、は **Owner 決定待ち（O7）**。

## Owner 決定待ち

ここに挙げた項目は、この ADR では決めない。決まるまで後続の issue は該当する機能を実装しない
（または宣言しない）。法的な判断を要する項目は、弁護士への確認を前提にする。

| # | 決めること | 決まるまで止まるもの |
|---|---|---|
| O1 | JPN で「利用者が自分で選んだ弁護士へ利用者の操作で資料を渡す」形を提供するか。72 条の「周旋」に当たらないかの確認 | X1・X3、AWAI-3236 の送付画面 |
| O2 | 収益 mode（JPN は law-firm-saas-license か none か、Awai の経済圏との接続） | 課金に関わるすべて |
| O3 | 事業として始めるか、誰が operator として資格・責任を負うか | 実運用、AWAI-3238 の公開配布 |
| O4 | 弁護士の資格確認の照合元・頻度・失効の反映 | X2、lawyer-reviewed の実運用 |
| O5 | 弁護士法 27 条（非弁提携）の原文と射程（カタログの known-gaps） | 事務所との提携の形 |
| O6 | 行政書士法・司法書士法の独占業務と、官公署・登記・裁判所提出書類の扱い | 該当する書類の生成 |
| O7 | 最初の法域（JPN / GBR / 両方） | profile と app の初期設定 |
| O8 | 守秘義務・個人情報（保存期間・削除・事務所間の受け渡し） | データの共有・同期 |
| O9 | 日弁連の業務広告規程の取得と、lawyer-directory の検証 | 弁護士一覧の機能 |
| O10 | 公開前に外部の弁護士のレビューを受けるか | 公開 |

## 結果

- 後続の issue は、この ADR の D1〜D8 を前提に作り、O1〜O10 に当たる機能は決定まで実装しない。
- AWAI-3233 の負の対照には、法的判断を求める入力、紹介料を含む収益 mode、JPN での referral-placement を含める。
- カタログが変わったとき（再検証、法域の追加）は、この ADR の表を更新する。

## 調査中に見つけた食い違い（報告のみ、この PR では直さない）

- `src/legalsupport/facts.cljk` の JPN `:jurisdiction/known-gaps` は
  「行政書士法1条の2/19条、司法書士法3条の原文が未取得」としているが、同じファイルの
  `jpn.gyoseishoshi-ho` と `jpn.shihoshoshi-ho-3` は `:rule/verification :primary-source-read`
  （2026-07-26 取得）になっている。README も 2026-07-26 に取得したと書いている。known-gaps の記述が古い可能性がある。
- 最新 commit（5cfff93）の題は「CLAUDE.md を AGENTS.md へ統合」だが、この repo には
  AGENTS.md も CLAUDE.md もない（その commit の変更は `src/legalsupport/actor.cljk` と `store.cljk` だけ）。
