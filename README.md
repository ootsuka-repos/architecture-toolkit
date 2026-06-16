# architecture-toolkit

コードベースのアーキテクチャを体系的にレビューし所見を並列・安全に適用し、収束するまで再帰反復し、さらに新機能を設計から並列実装まで進めるための Claude Code プラグイン。修正(review/fix)・反復改善(loop)・開発(build)をカバーする4つのスキルを内包します。

## 内包スキル

### `architecture-review`
コードベースを「まず設計として理解し」、その理解を基準に 14 観点 × 名前付き原則(DRY/KISS/YAGNI/SoC/SOLID 等)で改善点を網羅的・優先度付きに洗い出す。出力は構造化 JSON(正本)＋ 人間向け Markdown。

- 「アーキテクチャをレビューして」「技術的負債はどこ」「リファクタの方針を立てて」などで起動。
- 各指摘は `file:line` の根拠・深刻度・労力・AI 実行可能な修正仕様まで落とす。

### `architecture-fix`
`architecture-review` が出力した構造化 JSON(findings)を入力に、修正を衝突しないグループ(wave)へ分割し、Workflow で並列適用＋段階検証＋グループコミットする相棒スキル。

- 「出した指摘を直して」「findings を並列で一気に修正して」などで起動。
- 同一ファイルに触れる finding は別 wave に分離し、リネーム等の全域波及は単独中央逐次で安全に適用。

### `architecture-loop`
`architecture-review`(調査)と `architecture-fix`(改善)を**1周として収束するまで再帰反復する**オーケストレーター。両スキルを再実装せず合成し、周回・状態・停止だけを司る上位レイヤ。

- 「レビューと修正を繰り返して」「収束するまでリファクタして」「指摘がなくなるまで回して」「品質ゲートを満たすまで直して」などで起動。
- 1周 = review → 対象選定 → 並列適用 → 検証 → 進捗(デルタ)測定。品質ゲート達成・進捗の頭打ち・反復予算の消尽で停止し、回帰(検証赤化)・振動(指摘の出戻り)を検知して安全に止める。
- 台帳で finding を `applied/failed/deferred/wontfix` 管理し、失敗・wontfix を無限に再挑戦しない。各周の review JSON とメトリクスを残し、収束の軌跡を証跡化する。

### `feature-build`
新機能・機能追加を「設計から実装まで」体系的に進める開発スキル。要件を確定し、既存アーキテクチャに整合する設計を立て、実装を衝突しないタスク群(wave)に分割し、Workflow で並列実装＋段階検証＋グループコミットする。

- 「この機能を追加して」「〜を実装して」「〜できるようにして」などで起動。
- 既存パターンを踏襲して設計し、基盤(型/IF)を先行 wave、機能本体を並列 wave、配線(routing/DI/export)を最終 wave で中央集約。テスト同伴で受け入れ条件を担保。
- review/fix が「既存コードを直す」のに対し、本スキルは「新しい振る舞いをゼロから足す」開発フェーズを担う。

## 使い方の流れ

**既存コードの改善(修正):**
1. `architecture-review` でコードベースを評価 → `architecture-review.json` を生成。
2. `architecture-fix` でその JSON を読み込み、修正を並列適用。

**既存コードの改善(収束まで反復):**
1. `architecture-loop` に品質ゲートと反復予算を伝える → review→fix を収束(ゲート達成/頭打ち/予算消尽)まで自動で周回。
2. 各周の review JSON・メトリクス・グループコミットが残り、収束の推移を確認できる。

**新機能の開発:**
1. `feature-build` に作りたい機能を伝える → 設計 → wave 分割 → 並列実装 → 段階検証まで一気通貫。

## インストール

ターミナルで以下を実行します（マーケットプレイスを追加 → プラグインを導入）。

```bash
claude plugin marketplace add ootsuka-repos/architecture-toolkit
claude plugin install architecture-toolkit@architecture-toolkit
```

Claude Code 内なら `/plugin` で対話メニューからも追加できます。導入後はセッションを再起動すると
4スキル(architecture-review / architecture-fix / architecture-loop / feature-build)が有効になります。

## 更新（入れ直し）

リポジトリを更新したら、マーケットプレイスを再取得してプラグインを更新します（反映には再起動が必要）。

```bash
claude plugin marketplace update architecture-toolkit
claude plugin update architecture-toolkit@architecture-toolkit
```

すでに古い版が入っている場合もこの手順で最新に入れ替わります。完全に入れ直すなら
`claude plugin uninstall architecture-toolkit@architecture-toolkit` の後に上記の install を実行します。

## ライセンス

MIT
