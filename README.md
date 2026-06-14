# architecture-toolkit

コードベースのアーキテクチャを体系的にレビューし、その所見を並列・安全に適用するための Claude Code プラグイン。2つのスキルを内包します。

## 内包スキル

### `architecture-review`
コードベースを「まず設計として理解し」、その理解を基準に 14 観点 × 名前付き原則(DRY/KISS/YAGNI/SoC/SOLID 等)で改善点を網羅的・優先度付きに洗い出す。出力は構造化 JSON(正本)＋ 人間向け Markdown。

- 「アーキテクチャをレビューして」「技術的負債はどこ」「リファクタの方針を立てて」などで起動。
- 各指摘は `file:line` の根拠・深刻度・労力・AI 実行可能な修正仕様まで落とす。

### `architecture-fix`
`architecture-review` が出力した構造化 JSON(findings)を入力に、修正を衝突しないグループ(wave)へ分割し、Workflow で並列適用＋段階検証＋グループコミットする相棒スキル。

- 「出した指摘を直して」「findings を並列で一気に修正して」などで起動。
- 同一ファイルに触れる finding は別 wave に分離し、リネーム等の全域波及は単独中央逐次で安全に適用。

## 使い方の流れ

1. `architecture-review` でコードベースを評価 → `architecture-review.json` を生成。
2. `architecture-fix` でその JSON を読み込み、修正を並列適用。

## インストール

Claude Code のプラグインマーケットプレイス経由、もしくはこのリポジトリを参照して追加してください。

```
/plugin
```

## ライセンス

MIT
