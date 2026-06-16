# 出力スキーマ（構造化出力の正本）

レビュー結果の**正本は単一の構造化 JSON**（下記スキーマ準拠）。人間向け Markdown はこの JSON から
描画した二次表現に過ぎない。構造化することで、別の修正エージェントがプログラムで読んで適用でき、
差分追跡・フィルタ・優先度ソートも機械的にできる。

- 正本 JSON を `architecture-review.json`（または指定先）に保存し、同内容を Markdown でも提示する。
- すべての ID は安定（`F01`,`F02`,…）。`deps`/`roadmap` は ID 参照で結合する。
- 値の語彙は固定: `severity ∈ {critical,high,medium,low}`, `effort ∈ {S,M,L}`,
  `confidence ∈ {verified,likely,speculative}`（speculative は必ず根拠不足を意味する）。
- `recipe` は fix-recipes.md の R番号、`principles` は principles.md の ID、`viewpoints` は viewpoints.md の番号。
- **roadmap は finding 直下に持たせず、トップレベルの `roadmap{now,next,later}` を正本(SSOT)にする**。
  fix/loop が「now/next/later で対象選定」する際は、この mapping から `finding_id → bucket` を解決する
  (per-finding に roadmap を複製すると二重管理になり SSOT を崩す)。

## トップレベル構造
```json
{
  "meta": {
    "target": "リポジトリ名/対象パス",
    "scope": "全体 | <package> | <関心領域>",
    "languages": ["python", "typescript"],
    "loc": 27000,
    "module_count": 170,
    "generated_by": "architecture-review skill"
  },
  "architecture": {
    "purpose": "システムの目的(1-2文)",
    "components": [
      {"name": "core", "responsibility": "共通基盤(1行)", "path": "src/pkg/core"}
    ],
    "dependencies": {
      "layers": ["core", "domain", "apps", "ui"],
      "edges": [{"from": "apps", "to": "domain"}],
      "cycles": [],
      "violations": [{"from": "core", "to": "web", "note": "基盤が上位を import"}]
    },
    "flows": [
      {"name": "音声生成", "steps": ["scenario", "tts", "promo", "thumbnail", "publish"]}
    ],
    "entrypoints": [
      {"kind": "cli|server|job|worker", "ref": "pkg.apps.voicework:main", "note": ""}
    ],
    "external_deps": [{"name": "SDXL", "kind": "model", "boundary": "image.sdxl"}],
    "decisions": ["なぜこの構造か(設計判断)"]
  },
  "findings": [
    {
      "id": "F01",
      "title": "1行の要約",
      "viewpoints": [1, 3],
      "principles": ["SRP", "SoC"],
      "severity": "medium",
      "effort": "M",
      "confidence": "verified",
      "location": {"files": ["src/pkg/image/thumb.py"], "anchor": "def render_thumb", "lines": "1-40"},
      "problem": "何が問題か(根拠つき)",
      "impact": "放置するとどうなるか",
      "fix": {
        "summary": "image/thumb* を image/thumbnail/ へ移動・改名",
        "recipe": ["R1", "R4"],
        "before": "from pkg.image.thumb import render_thumb",
        "after": "from pkg.image.thumbnail.render import render_thumb",
        "refs": ["apps/*.py の import(3)", "pyproject scripts", "README 構成図", "docstring"],
        "deps": [],
        "verify": ["import解決 OK", "ruff F 通過", "grep 'image.thumb\\b' src = 0件", "実importスモーク"]
      },
      "evidence": "src/pkg/image/thumb.py:1, 被参照 grep 結果 等"
    }
  ],
  "principle_coverage": [
    {"principle": "DRY", "violated": true, "finding_ids": ["F03", "F07"]},
    {"principle": "YAGNI", "violated": false, "finding_ids": []}
  ],
  "roadmap": {
    "now":   [{"finding_id": "F01", "reason": "高価値・低リスク"}],
    "next":  [{"finding_id": "F05", "reason": "依存是正の前提"}],
    "later": [{"finding_id": "F09", "reason": "大改修・テスト先行が要る"}]
  },
  "strengths": ["維持すべき設計上の強み(壊さないため)"],
  "assumptions": ["テスト不在: 挙動変更系は特性化テスト先行 等の前提"]
}
```

## 妥当性の自己点検（出力前に必ず）
- 全 `findings[].id` は一意で、`deps`/`roadmap`/`principle_coverage.finding_ids` の参照先が実在する。
- `principle_coverage` は principles.md の主要原則を網羅（violated=false も明示）し、観点軸の漏れも別途確認。
- 各 finding は `location.files` が実在し、`confidence != verified` の項目は `evidence` か `(推測)` を持つ。
- `severity`/`effort`/`confidence`/`recipe`/`principles`/`viewpoints` の語彙が固定値に収まる。
- 着手対象（now/next の finding）は `fix.before/after`・`refs`・`verify` が埋まっている（=AI が適用可能）。

## Markdown は JSON から描画する
人間向けには同じ JSON を次の順で描画: ①architecture 要約 → ②findings を severity 降順の表
（列: severity/viewpoints/principles/title/location/fix.summary/effort）→ ③principle_coverage の一覧 →
④roadmap(now/next/later) → ⑤strengths。表の各行は finding.id を持ち、JSON と相互参照できること。
