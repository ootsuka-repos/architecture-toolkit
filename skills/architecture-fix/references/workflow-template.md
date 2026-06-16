# Workflow スクリプト雛形（wave barrier 適用）

Phase 1 で決めた実行列(SERIAL と PARALLEL wave の混在)を Workflow ツールで回す雛形。
**wave 間は barrier**(次 wave は前段の統合・検証後に開始)なので `pipeline` ではなく
`parallel` を wave 単位で使い、各 wave 後に検証・コミットを挟む。

雛形(`grouping.md` の結果を `STEPS` に流し込んで使う。値は実 finding で埋めること):

```javascript
export const meta = {
  name: 'apply-arch-fixes',
  description: 'architecture-review.json の findings を wave 単位で並列適用し、各 wave で検証・コミット',
  phases: [{ title: 'Apply' }, { title: 'Verify' }],
}

// STEPS は Phase1 の出力。kind: 'serial' は1件、'wave' は並列群。
// 各 finding f は {id, summary, recipe, before, after, refs, verify, files, deps} を持つ。
const STEPS = args.steps   // Workflow 呼び出しの args で渡す

const RESULT = {
  type: 'object',
  required: ['id', 'status', 'verify_passed'],
  properties: {
    id: { type: 'string' },
    status: { enum: ['applied', 'failed', 'skipped'] },
    files_changed: { type: 'array', items: { type: 'string' } },
    verify_passed: { type: 'boolean' },
    verify_output: { type: 'string' },
    notes: { type: 'string' },
  },
}

const applyPrompt = (f) => `あなたは architecture-review の finding を1件だけ適用する修正エージェント。
ID: ${f.id}
要約: ${f.summary}
レシピ: ${(f.recipe || []).join(', ')}
対象ファイル(これ以外は絶対に触らない): ${(f.files || []).join(', ')}
before:
${f.before || '(なし)'}
after:
${f.after || '(なし)'}
波及更新(refs — import/設定/docstring/動的参照も直す): ${(f.refs || []).join(' / ')}
検証(適用後に必ず実行し、成功条件を満たすか確認): ${(f.verify || []).join(' / ')}

手順:
1. 対象ファイルの現状を読み、before と差異があれば実コードに合わせて整合させる(乖離が大きければ status=failed)。
2. 変更を適用し、refs に挙がった波及箇所も更新する。挙動を変える整理なら現状の入出力を保つ(必要なら特性化テスト先行)。
3. 検証コマンドを実行する。旧名 grep は 0 件、import 解決・リンタが通ることを確認。
4. 結果を構造化で返す(verify_output には実行ログ要約を入れる)。
他 finding のファイルには触れないこと(並列実行の前提)。

注意(よくある回帰):
- print→logging 置換時、\`logger = getLogger(__name__)\` の代入は **必ず全 import 文の後ろ**に置く
  (import ブロックの途中・複数行 import の内側に挟むと構文/E402 エラーになる)。
- private→public 化したら旧名は alias で残す(スコープ外やテストが旧名を参照しうる)。削除は外部参照 0 件確認後。
- 適用後に自分が編集したファイルを必ず py_compile / 構文チェックし、新規の構文・import 位置エラーが無いことを確認する。`

const results = []
for (const step of STEPS) {
  phase('Apply')
  let stepResults
  if (step.kind === 'serial') {
    // リネーム/移動など全域波及: 1件ずつ中央で
    const r = await agent(applyPrompt(step.finding), { label: `serial:${step.finding.id}`, schema: RESULT })
    stepResults = [r].filter(Boolean)
  } else {
    // wave: ファイル素集合なので並列で作業ツリーを直接編集
    stepResults = (await parallel(
      step.findings.map(f => () => agent(applyPrompt(f), { label: `apply:${f.id}`, schema: RESULT }))
    )).filter(Boolean)
  }
  results.push(...stepResults)

  // wave 後の全体検証(verify.md のコマンドを実行)。失敗 finding があれば停止判断のため記録。
  phase('Verify')
  const failed = stepResults.filter(r => r.status !== 'applied' || !r.verify_passed)
  log(`step done: applied=${stepResults.filter(r => r.status === 'applied').length} failed=${failed.length}`)
  if (failed.length) {
    log(`STOP: ${failed.map(r => r.id).join(',')} で検証失敗。後続 wave を中止して切り分けが必要。`)
    break   // 壊れた状態で次 wave に進まない
  }
  // ここまで緑。main 側でグループコミットする(下記注記参照)。
}

return { results }
```

## コミットの扱い
- Workflow 内の subagent はファイルを編集するが、**コミットは main 側(本体)で wave ごとに行う**のが安全
  (検証 PASS を本体が確認してから `fix(arch): <要約> (F0x,F0y)` でコミット)。
- 実運用では「1 wave 分の Workflow を回す → 本体で全体検証 → コミット → 次 wave の Workflow」と
  ステップを刻むか、上記のように1回の Workflow で回しつつ各 wave 後の検証結果を `results` で受け取り、
  本体が wave 境界ごとに `git add -A && git commit` する。どちらでも「緑を確認してからコミット」を守る。
- 検証コマンドが Workflow 内から実行しづらい場合は、subagent の `verify_passed` を一次情報とし、
  本体が最終 wave 後にまとめて全体検証(verify.md)してから push 等を判断する。

## 規模に応じた調整
- finding 数が少なく wave も1〜2なら、Workflow を使わず素朴に並列エージェントを起動(or 手番で順に適用)でも可。
- 多数(目安 6件以上 or wave 3段以上)で順序・検証・コミットを決定論的に回したいときに Workflow が活きる。
