# 手法: 並列探索・import グラフ・検証

Phase 1（理解）と各所見の検証で使う具体手順。**根拠ベース**かつ**大規模でも速く**やるための型。

## A. 大規模コードベースの並列ファンアウト
全体を1人で読むと遅く浅くなる。サブシステム単位で並列に読ませ、構造化要約を集める。

- サブエージェント/ワークフローが使えるなら、パッケージ(または関心領域)ごとに1エージェントを割り当て、
  各自に次を構造化して返させる: 責務 / 主要ファイルと役割 / 「誰が呼ぶか」/ 依存方向 /
  外部I/O / 死にコード候補 / 重複候補 / 命名の問題。
- 横断観点(モデルロード・エラー処理・並行性・命名・重複・死にコード)は専任エージェントを立てると、
  パッケージ跨ぎのパターンを拾える。
- 集約時に**所見を重複排除**し、`file:line` を保持。重要・不確実な所見は別エージェントで**裏取り(検証)**する
  （誤検出を排除＝レビューの信頼性を守る）。

並列が無い環境では、関心領域を順に読み、同じ構造化フォーマットで要約を積む。

## B. import 到達可能性グラフ（死にコード/依存方向の機械的検証）
「未使用ファイル」「循環」「レイヤ違反」は推測せず静的に出す。Python の例（他言語も発想は同じ）:

```python
# 全モジュールを AST 解析し、エントリから到達不能なモジュールを列挙する。
import ast, sys
from pathlib import Path
ROOT = Path("src"); PKG = "your_package"          # ← 調整
SKIP = ("vendor","node_modules","__pycache__",".venv","dist","build",".git")  # ← 対象外(調整可)
def skip(p): return any(x in p.parts for x in SKIP)

mods, edges = set(), {}
for p in (ROOT/PKG).rglob("*.py"):
    if skip(p): continue
    parts = list(p.relative_to(ROOT).with_suffix("").parts)
    if parts[-1]=="__init__": parts=parts[:-1]
    mods.add(".".join(parts))
# パッケージマーカーも既知集合へ
known=set(mods)
for m in list(mods):
    pp=m.split(".")
    for i in range(1,len(pp)): known.add(".".join(pp[:i]))

for p in (ROOT/PKG).rglob("*.py"):
    if skip(p): continue
    parts=list(p.relative_to(ROOT).with_suffix("").parts)
    cur=parts[:-1]                                 # ファイルを含む package
    mod=".".join(parts[:-1] if p.name=="__init__.py" else parts)
    deps=set(); tree=ast.parse(p.read_text(encoding="utf-8",errors="replace"))
    for n in ast.walk(tree):
        if isinstance(n, ast.ImportFrom):
            if n.level:                            # 相対 import を絶対へ解決
                base=cur[:len(cur)-(n.level-1)]
                tgt=".".join(base+([n.module] if n.module else []))
            elif n.module and n.module.startswith(PKG): tgt=n.module
            else: continue
            deps.add(tgt)
            for a in n.names: deps.add(tgt+"."+a.name)
        elif isinstance(n, ast.Import):
            for a in n.names:
                if a.name.startswith(PKG): deps.add(a.name)
    edges[mod]={d for d in deps if d in known} or set()

ENTRIES=[...]                                       # ← CLI/サーバ/エントリを列挙
reach=set(); stack=list(ENTRIES)
while stack:
    m=stack.pop()
    if m in reach: continue
    reach.add(m); stack+=[d for d in edges.get(m,()) if d not in reach]
print("unreached:", sorted(m for m in mods if m not in reach))
```

注意:
- **エントリを正しく列挙**する(pyproject scripts / `python -m` / サーバ app / 動的 import)。漏れると誤検出。
- **動的参照は静的解析の盲点**: `importlib.import_module(文字列)`、`__getattr__`/`_LAZY` 遅延 re-export、
  subprocess の `python -m ...` 文字列、エントリ定義。これらは別途 grep で確認する。
- 「到達不能」でも `python -m` で手動起動するツールは機能。削除前に用途を確認。

## C. リネーム/移動を安全に行う型（命名・構成改善の実行）
import は全域に波及し、並列だと共有 importer で衝突する。**中央で一括・段階検証**が安全。
1. 対象内の相対 import を一旦絶対化(`from .X`→`from pkg.sub.X`)してから移動すると、移動が文字列置換で済む。
2. 1グループずつ: `git mv` → 絶対 dotted パスを全置換(.py/設定/ドキュメント) → 検証 → 次へ。
3. **落とし穴**: `from pkg import name`(パッケージ経由 import 形)は dotted 置換で漏れ、静的チェックを通過し
   実 import で落ちる。各移動後に実 import スモークで確認。
4. 動的参照(B の注意)とエントリ定義・package-data・lint exclude・gitignore・docstring も更新。

## D. 検証コマンド例（言語別に読み替え）
- 静的 import 解決: 上記 B のスクリプト（壊れた import を実行せず検出）。
- 構文: `python -m compileall -q <src>`（ベンダーは除外/無視）。
- リンタ(未定義名・未使用 import): `ruff check <src> --select F`。
- 型: 利用していれば `mypy` / `pyright`。
- 実 import スモーク: `python -c "import a, b, c; print('OK')"`（重い副作用に注意）。
- 旧名/旧パス残存: `grep -rn "<old>" <src> --include=*.py | grep -v vendor`。
- 重複の等価性: 対象関数を抽出して入出力を突き合わせる/特性化テストを書く。
- テスト: あれば実行。無ければ「無い」事実をレポートに明記（挙動変更系の前提になる）。

他言語の対応物: JS/TS=`tsc --noEmit`/eslint/madge(循環), Go=`go vet`/`go build`,
Java=コンパイル+ArchUnit, など。発想は「到達可能性・依存方向・壊れ参照を機械で出す」で共通。

## E. レビューの質を保つ原則
- 所見は `file:line` で示し、断定は裏取り後のみ。未確認は「(推測)」。
- 設計意図(docstring/README/コミット履歴)を尊重し、意図的分割を欠陥と誤認しない。
- 深刻度×労力で優先度化。根本原因(例: god-module)は束ねて根本対処を提案。
- テスト不在では挙動変更を高リスクとし、特性化テスト先行を勧める。
