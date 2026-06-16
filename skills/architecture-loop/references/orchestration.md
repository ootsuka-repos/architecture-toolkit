# オーケストレーション（review→fix の合成・状態の受け渡し・対象選定）

本スキルは review/fix を**再実装せず合成**する。ここでは1周の中で両スキルをどう繋ぎ、何を周回間で
受け渡すかを定義する。原則: **1周の中身は委譲、本スキルは順番・状態・選定・停止だけを持つ**。

## 1イテレーションの標準フロー
```
review(n)  →  select(n)  →  fix(n)  →  verify(n)  →  record(n)  →  収束判定
   │            │            │           │             │
 review JSON  対象集合     wave適用＋   全体検証     台帳/メトリクス更新
 (構造化)     (台帳で絞る) グループコミット (緑/赤)     (loop-control.md)
```

### 1. review(n) — 調査を委譲
- `architecture-review` を起動し、出力 JSON を `arch-loop/iter-<n>/review.json` に保存する(正本)。
- **入力に台帳を渡す**(2周目以降): `wontfix` の finding は再掲しない、`applied` 済みの箇所は
  「適用済みとして確認」扱い、`failed` は再評価してよいが原因(乖離)を添える。
- **差分スコープ**(効率化): 2周目以降は「前周で変更したファイル/パッケージ＋未着手領域」に焦点を当ててよい。
  ただし `full_sweep_every`(既定3周に1回)は**全体を再スイープ**して、局所修正が他所に生んだ歪みや
  取りこぼしを拾う。差分スコープだけで回し続けると盲点が固定化する。

### 2. select(n) — 今周の対象を選ぶ
review JSON の `findings[]` から `apply_policy` に従って今周直す集合を決める:
- 既定: `severity ∈ {critical, high}` かつ roadmap が `now`/`next`。**roadmap は finding 直下ではなく
  トップレベルの `roadmap{now,next,later}` mapping(正本=SSOT)**。`finding_id → bucket` を解決して絞る
  (per-finding に複製しない。雛形の `roadmapOf` を参照)。
- **台帳で除外**: `wontfix` と、`attempts` 上限に達した `failed` は外す(振動防止)。
- **依存順を尊重**: `fix.deps` の先行が未了の finding は今周に入れない(次周以降)。
- 件数が多すぎる周は上位 severity から件数を絞ってよい(1周を小さく保つと検証と切り分けが楽)。
- 選んだ集合が空なら、ポリシを1段広げる(high→medium、now→next→later)か、収束扱いにする。

### 3. fix(n) — 改善を委譲
- 選んだ findings(部分集合)を `architecture-fix` に渡す。fix スキルが wave 分割→並列適用→各 wave 検証→
  グループコミットまで担う。**本スキルは wave の中身に立ち入らない**(委譲)。
- fix には「この周の対象 ID 集合だけを適用せよ」と明示する(review JSON 全件ではなく select(n) の部分集合)。
- fix の戻り(各 finding の `applied/failed/skipped` と verify 結果)を受け取り、台帳更新に使う。

### 4. verify(n) — 周末の全体検証
- 周内の各 wave は fix スキルが検証済み。周末に**もう一度全体検証**(ビルド/型/リンタ/テスト/スモーク)を
  本体側で実行し、周単位の緑を確認する(委譲先の verify.md のコマンドを使う)。
- 赤なら `loop-control.md` の回帰処理(切り戻し→停止)へ。

### 5. record(n) — 状態を更新
- 台帳(`ledger.json`)に今周の `applied/failed/skipped` を反映。`failed` は `attempts` を加算し、上限到達で
  `wontfix` に昇格。
- `arch-loop/iter-<n>/summary.json` に severity 別件数・resolved・progress・コミットハッシュを記録。
- これらが Phase 2 の収束判定と Phase 4 のレポート(収束カーブ)の入力になる。

## 周回間で受け渡すもの（状態の引き継ぎ）
| 渡すもの | 形 | 用途 |
|----------|----|------|
| 台帳 ledger | `arch-loop/ledger.json` | 再掲抑制・対象除外・振動検知 |
| 各周の review | `arch-loop/iter-<n>/review.json` | 件数差分(デルタ)・収束判定 |
| 各周のサマリ | `arch-loop/iter-<n>/summary.json` | 進捗・コミット・レポート |
| 変更ファイル集合 | 周の fix 戻り or `git diff --name-only` | 次周の差分スコープ review の焦点 |

## 失敗・wontfix の扱い（賢く繰り返すための要）
- **failed を盲目的に再挑戦しない**: 同じ before/after で再適用しても同じく失敗する。`attempts` を数え、
  上限(既定2)で `wontfix` に昇格。原因(実コードとの乖離・テスト不在など)をレポートに残す。
- **wontfix は次周 review に伝える**: 「意図的設計と確認済み/直さないと決定」を review 側に渡し、再掲させない。
  これをしないと毎周同じ指摘が出て progress が伸びず頭打ち/振動と誤判定される。
- **deferred(later/低優先)** は予算が残れば最終周以降で拾う。ゲートが critical/high のみなら触れなくてよい。

## 委譲の境界（やること/やらないこと）
- **本スキルがやる**: 周回(ループ)、対象選定、台帳/状態管理、デルタ測定、収束/回帰/振動の判定、周境界の
  barrier とコミット方針、最終レポート(収束カーブ)。
- **本スキルがやらない(委譲)**: 観点別監査の中身(review)、wave 分割と並列適用の中身(fix)、言語別の検証
  コマンド(各 references/verify.md)。これらを作り直さない——重複は両スキルの改善が二重メンテになる(DRY)。

## 規模に応じた調整
- 指摘が少なく1〜2周で収束する見込みなら、Workflow を使わず「review → fix → 再 review で確認」を素直に
  手番で回してもよい(本スキルの判断基準はそのまま使う)。
- 多数の指摘を数周にわたり決定論的に回す/証跡を残す必要があるときに、ループ全体の Workflow 化が活きる
  (`references/workflow-template.md`)。
