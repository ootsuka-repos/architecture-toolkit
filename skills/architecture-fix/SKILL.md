---
name: architecture-fix
description: >
  architecture-review が出力した構造化 JSON(findings) を入力に、修正を「並列で効率よく・安全に」
  適用するスキル。findings の fix 仕様(before/after/refs/verify)と deps・ファイル重複から
  衝突しないグループ(wave)に分け、Workflow で並列適用し、各 wave ごとに検証してコミットする。
  ユーザーが「(レビューで)出した指摘を直して」「architecture-review.json を適用して」
  「指摘を並列で一気に修正して」「findings を反映して」「リファクタ計画を実行して」
  「レビュー結果を反映して」などと言ったときに使う。
  architecture-review の続きとして、所見の一括適用・実行フェーズを担う相棒スキル。
  単発の小修正ではなく、複数 finding をまとめて安全に適用する場面で必ず使うこと。
---

# Architecture Fix（レビュー所見の並列・安全適用）

`architecture-review` が生成した正本 JSON(`architecture-review.json`)を読み、`findings[]` の
**実行可能修正仕様**をコードベースに適用するスキル。ゴールは **速さ(並列)と安全(衝突回避＋段階検証)の
両立**。レビューが「何をどう直すか」を構造化済みなので、本スキルはそれを機械的に・壊さずに反映する。

中核アイデア: findings を **衝突しないグループ(wave)** に分割する。
- 同じファイルに触れる finding は同じ wave に入れない(並列編集の競合を防ぐ)→ wave 内はファイル素集合。
- `deps` で先行が要る finding は後の wave に回す(依存順)。
- **リネーム/移動(R1)など全域波及する finding は単独 wave で中央逐次**(共有 importer 衝突を防ぐ)。

wave 内はファイルが重ならないので、worktree を使わず作業ツリー上で直接並列編集して問題ない
(変更が即統合され、マージ不要で速い)。重なりが避けられない時だけ worktree 隔離にフォールバックする。

## いつ使うか
- architecture-review 直後に「この指摘を直して/適用して/反映して」と言われたとき。
- `architecture-review.json`(または同等スキーマの findings)が手元にあり、複数の修正をまとめて入れたいとき。
- 「並列で一気に」「効率よくリファクタを実行して」など、多数の修正の実行フェーズ。

単一 finding・1ファイルで明白な修正だけなら本スキルは不要(普通に直す)。レビュー JSON が無ければ
先に `architecture-review` を回して正本 JSON を作る。

## 進め方（4フェーズ）

### Phase 0 — 入力ロードと前提確認
- `architecture-review.json` を読む(場所が不明なら聞く/リポジトリ直下を探す)。スキーマは
  architecture-review スキルの `references/output-schema.md` と同一。
- **適用対象 findings を決める**: 既定は引数で指定(`now` / `next` / `later` / `all` / `F01,F05` など)。
  指定が無ければ全 finding を依存順で対象にする。
- **適用前ゲート**(壊れた状態で始めない):
  - git 作業ツリーがクリーンか確認。未コミット変更があれば退避/コミットを促す。
  - main 等の保護ブランチ上なら作業ブランチを切る。
  - ベースラインの検証が通ることを確認(`references/verify.md` のコマンド)。最初から赤なら原因を切り分け。
- JSON の妥当性を軽く点検(ID 一意・`deps`/対象 ID の参照先が実在・`fix.before/after/verify` が埋まっているか)。
  着手対象なのに fix が空の finding は適用不可として除外し、ユーザーに報告。

### Phase 1 — グループ化と順序付け（本スキルの核心）
findings を wave に分割する。アルゴリズムと衝突判定の詳細は **`references/grouping.md` を読む**。要点:
1. `fix.recipe` に `R1`(リネーム/移動)を含む、または全域波及する finding は **SERIAL(単独中央逐次)**。
2. 残りは PARALLEL 候補。各 finding の「触れるファイル集合」= `location.files ∪ fix.refs から解決したファイル`。
3. 依存順(`deps` と roadmap)を尊重しつつ、**ファイルが重ならない finding を貪欲に同一 wave へ詰める**。
4. 結果は `[wave0, wave1, ...]` の列。各 wave は並列実行可能、wave 間は逐次(前段の検証 PASS 後に次段)。

挙動を変える整理(R2 共通化 / R4 分割 / R6 エラー処理)は、特性化テストが無ければ **テスト先行(R8)**を
その finding の前段 wave として差し込む(`assumptions` にテスト不在とあれば特に)。

### Phase 2 — Workflow で並列適用
**Workflow ツール**で wave 列を実行する。スクリプト雛形は **`references/workflow-template.md` を読んで**埋める。
構造(pipeline ではなく wave 単位の barrier が要る: 次 wave は前 wave 統合後に開始):

- SERIAL finding は `agent()` を1個ずつ(中央)。各適用後に検証→コミット。
- 各 PARALLEL wave は `parallel(wave.map(f => () => agent(applyPrompt(f), {schema: RESULT})))`。
  - 各 subagent には finding の `id/fix.summary/before/after/refs/recipe/verify/location` を渡し、
    「指定ファイルだけを編集し、refs(import/設定/docstring/動的参照)も更新し、finding.verify を実行して
    結果を構造化で返せ」と指示。**他 finding のファイルには触れない**ことを明示(wave 設計の前提)。
  - subagent は `{id, status: applied|failed|skipped, files_changed[], verify_passed, verify_output, notes}` を返す。
- wave 完了ごとに **全体検証**(`references/verify.md`: import 解決 / リンタ / 型 / 実 import スモーク /
  旧名 grep=0)を main 側で実行し、PASS したら **その wave をグループコミット**する。

### Phase 3 — 検証・コミット・レポート
- **グループ毎コミット**(確定方針): 各 wave/SERIAL finding の検証 PASS 直後に、その範囲だけをコミット。
  メッセージは `fix(arch): <要約> (F01,F03,…)` 形式で、含む finding.id を列挙して JSON と相互参照可能に。
- どこかで検証が赤くなったら **そこで停止**(次 wave に進まない)。当該 wave の変更を切り分け、原因 finding を
  特定して報告(`status: failed` の finding と verify_output 付き)。安易に進めない。
- 最後に集計レポート: 適用済/失敗/スキップの finding.id 一覧、各コミットハッシュ、残課題(later 等の未適用)。
- 全 finding 適用後、JSON 側に適用状況を追記したい場合は `applied: true` 等のフィールドを付けて追跡可能にする。

## 原則（なぜそうするか）
- **衝突をデータで消す**: 並列の唯一の危険は同一ファイルへの同時編集。wave をファイル素集合にすれば
  worktree もマージも要らず、速くて安全。重なる時だけ隔離する。
- **全域波及は中央逐次**: リネーム/移動は import 全体に効くので並列化すると共有 importer で壊れる。
  単独 wave で一括置換し段階検証する(fix-recipes R1)。
- **壊れた状態で先に進まない**: 各 wave 後に検証して緑を確認してからコミット・次段へ。赤なら停止して切り分け。
- **fix 仕様を信じすぎない**: before/after は指針。適用時に実コードと差異があれば subagent が確認して整合させ、
  乖離が大きければ `failed` で報告(誤適用は信頼を壊す)。
- **挙動変更はテスト先行**: R2/R4/R6 は特性化テスト(R8)を前段に置き、リファクタ前後で同結果を担保する。

## 参考ファイル
- `references/grouping.md` — wave 分割アルゴリズム、ファイル集合の解決、衝突判定、SERIAL 判定、依存順。Phase 1 で読む。
- `references/workflow-template.md` — Workflow スクリプト雛形(wave barrier・適用プロンプト・RESULT スキーマ)。Phase 2 で読む。
- `references/verify.md` — 言語別の検証コマンドと成功条件(import 解決/リンタ/型/スモーク/旧名 grep)。Phase 0/各 wave 後で読む。
