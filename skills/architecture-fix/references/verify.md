# 検証コマンドと成功条件

各 finding の `fix.verify[]` が一次情報。ここでは **wave 後 / 全体** に走らせる定番チェックを言語別にまとめる。
成功条件を満たさなければ「緑」と見なさない(コミット・次 wave に進まない)。

## 共通の合否観点
- **import 解決**: 移動/リネーム/分割後にモジュールが解決できる。
- **リンタ(ベースライン比較)**: 未使用 import・未定義参照が出ない。**適用前のリンタ結果を控えておき、
  新規に増えたエラーだけを問題とする**(既存の整形系ノイズ=import 並び替え等を新規回帰と誤認しない)。
  特に **構文エラー / import 位置エラー(Python の E402 相当)が新規に出ていないか**を必ず見る。
- **型**(あれば): 型エラーが増えていない。
- **実 import スモーク**: 主要エントリ/パッケージが実際に import できる(`from pkg import name` 形の漏れ検出)。
- **旧名 grep = 0**: リネーム/移動の旧 dotted パス・旧シンボルが残っていない。
- **テスト(ベースライン比較)**: 挙動変更系(R2/R4/R6)はリファクタ前後で同結果。**観測手段を変えた場合
  (print→logging 等)、出力先を見ている既存テストが失敗しうる** — その場合はテスト側を新方式に追従させる
  (例: stderr 検査 → caplog 検査)。失敗が「テストの前提が古い」のか「コードのバグ」かを必ず切り分ける。

## よくある回帰（適用エージェントへの必須注意）
- **print→logging 置換**: `import logging` はファイル先頭の import 群に置き、`logger = logging.getLogger(__name__)`
  の代入は **必ず全 import 文より後ろ**に置く。import ブロックの途中(特に `from x import (` の複数行 import の
  内側)に代入を挟むと **構文エラー / import-not-at-top(E402) を誘発する**。挿入位置を機械置換する場合は
  括弧付き複数行 import の閉じ括弧の後であることを確認する。
- **後方互換シンボルの早すぎる削除**: private→public 化したら旧名は当面 alias で残す。スコープ外パッケージや
  テストが旧名を参照していることがあり、削除は「同パッケージ外からの参照 0 件」を grep で確認してから。

## Python（このリポジトリの主言語）
このプロジェクトは root の `.venv` を使う。コマンドは環境に合わせて調整すること。
```bash
# リンタ(未使用 import など F 系)
ruff check src

# 型(導入されていれば)
# mypy src    または    pyright

# 実 import スモーク(対象パッケージ/モジュールを実際に読み込む)
python -c "import importlib; importlib.import_module('doujin_forge.<対象>')"

# 旧名 grep が 0 件であること(リネーム/移動時)
#   Grep ツールで '旧dottedパス' を検索し 0 件を確認
```
- import 到達可能性をまとめて見たいときは、対象パッケージ配下を `compileall` で素早く構文チェック:
  `python -m compileall -q src/doujin_forge/<対象>`

## TypeScript / Node（該当時）
```bash
npm run lint        # or: npx eslint .
npx tsc --noEmit    # 型チェック
npm test            # テストがあれば
```

## 旧名 grep の手順（リネーム/移動の必須チェック）
1. 旧 dotted パス(例 `image.thumb`)と旧シンボル名を Grep ツールで全文検索。
2. 0 件であること。残っていれば置換漏れ → 修正して再検証。
3. 動的参照(`importlib`/`__getattr__`/`python -m ...` 文字列/設定ファイルの文字列)も忘れず検索対象に含める。

## 失敗時の原則
- 検証が赤 → **その wave で停止**。原因 finding を特定(どのファイル変更で壊れたか)。
- 直せるなら直して再検証、構造的に困難なら当該 finding を `failed` として残し、他の緑な wave は活かす。
- 壊れた変更を含むコミットは作らない(グループ単位なので切り戻しは `git restore`/`git reset` で局所化できる)。
