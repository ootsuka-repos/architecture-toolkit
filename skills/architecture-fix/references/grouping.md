# wave 分割アルゴリズム（並列の安全性をデータで担保する）

並列適用の唯一の致命的危険は **同じファイルを2エージェントが同時に編集** すること。これを
構造的に排除するため、findings を「wave(波)」に分ける。**wave 内 = ファイル素集合で並列可 /
wave 間 = 逐次(前段の検証 PASS 後に次段)**。

## 1. 各 finding の「触れるファイル集合」を解決する
`touched(f)` を次の和集合として求める:
- `f.location.files[]`(必ずファイル)。
- `f.fix.refs[]` のうちファイルに解決できるもの。refs は「`apps/*.py の import(3)`」のような自然文も
  混じるので、glob/ディレクトリ/モジュール dotted パスをできる範囲でファイルへ展開する。
  解決できない自然文 ref は **保守的に「広く触れる」とみなす**(同 wave に他を入れない方向に倒す)。
- `before`/`after` に現れる import 先モジュールも、置換対象なら touched に含める。

判定に迷うときは「重なる」側に倒す。並列度が少し下がるだけで、誤った同時編集より遥かに安全。

## 2. SERIAL（単独・中央・逐次）に隔離する finding
次のいずれかは並列に混ぜず、1個ずつ main 側(または単独 agent)で適用し、各回検証する:
- `fix.recipe` に **R1**(リネーム/移動)を含む。
- `fix.recipe` に **R5**(依存方向是正)を含み、複数モジュールの import を張り替える。
- `touched(f)` が事実上リポジトリ全域(共有 importer・`__init__`・pyproject・設定の同時更新)に及ぶ。
- `refs` に「全域」「全 import」「pyproject」「README 構成図」等、広域波及を示す語がある。

理由: これらは共有 importer/設定/エントリを書き換えるため、他の並列編集と必ず衝突しうる。
fix-recipes の R1 どおり「中央で一括置換→段階検証」する。

## 3. 依存順（deps と roadmap）
- `f.fix.deps[]` の finding は f より前の wave で完了していること。
- roadmap(now→next→later)はおおよその優先度。**deps が真の制約**、roadmap は同順位内の並べ替え目安。
- テスト不在で挙動を変える finding(recipe R2/R4/R6 かつ特性化テスト無し)は、その finding の
  **前段 wave に R8(特性化テスト導入)を差し込む**。テストの finding が無ければ「テスト先行」を
  適用 subagent の指示に含める(変更前に現状の入出力を固定 → 変更 → 緑を確認)。

## 4. 貪欲な wave パッキング（PARALLEL 候補）
SERIAL を除いた候補集合に対して:
```
remaining = PARALLEL候補 を (deps深さ, severity) で安定ソート
waves = []
while remaining:
    wave = []
    usedFiles = set()
    for f in remaining (順に):
        if f.deps がすべて (既存wave or 完了) で満たされ
           and touched(f) ∩ usedFiles == ∅:
            wave.append(f); usedFiles |= touched(f)
    waves.append(wave); remaining -= wave
```
- 同 wave 内はファイルが重ならない → 並列で作業ツリーを直接編集して安全(worktree 不要)。
- どうしても重なるが並列したい稀なケースのみ、その finding を別 wave にするか worktree 隔離にする。

## 5. 実行順の組み立て
最終的な実行列は次のように混在してよい:
```
[ SERIAL(F_rename1) ] → [ wave0: F02,F07,F11 (並列) ] → [ wave1: F03,F09 (並列) ] → [ SERIAL(F_move2) ] → …
```
- 一般に **リネーム/移動(SERIAL)を先に**済ませると、後続の touched 解決が安定する(パスが確定するため)。
  ただし SERIAL finding 自体に deps があればそれを優先。
- 各ステップ(SERIAL 1件 or 1 wave)の後に必ず全体検証 → PASS でグループコミット → 次へ。

## 6. 自己点検（Phase 1 終了時）
- すべての finding がちょうど1つの wave か SERIAL に割り当てられている(取りこぼし無し)。
- どの wave 内も touched が互いに素。
- すべての `deps` が先行ステップで満たされる順序になっている。
- 全域波及系がすべて SERIAL 側にある(PARALLEL wave に混入していない)。
