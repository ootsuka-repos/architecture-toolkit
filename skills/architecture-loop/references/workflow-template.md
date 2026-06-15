# Workflow スクリプト雛形（イテレーション barrier ループ）

review→fix の1周を**イテレーション単位の barrier**で繰り返す雛形。周内の wave 並列は fix スキルの
Workflow(`architecture-fix/references/workflow-template.md`)に委譲し、本テンプレートは
**周回・収束判定・周境界のコミット**だけを司る。次周は前周の検証 PASS とコミット後に開始する。

雛形(`loop-control.md` のパラメータと `orchestration.md` のフローを埋めて使う):

```javascript
export const meta = {
  name: 'arch-loop',
  description: 'architecture-review→fix を収束まで反復し、各周で検証・コミット・収束判定する',
  phases: [{ title: 'Review' }, { title: 'Fix' }, { title: 'Verify' }, { title: 'Decide' }],
}

// Phase 0 で確定した契約(loop-control.md の既定を上書き可)
const CFG = {
  qualityGate: (c) => c.critical === 0 && c.high === 0, // 収束の定義
  maxIterations: args.maxIterations ?? 5,               // 反復予算(安全弁)
  progressThreshold: args.progressThreshold ?? 1,       // 頭打ち判定
  applyPolicy: args.applyPolicy ?? { severities: ['critical', 'high'], roadmap: ['now', 'next'] },
  fullSweepEvery: args.fullSweepEvery ?? 3,
}

// 1周の各サブステップは既存スキルの実行に委譲する委譲関数(プロジェクトの起動方法に合わせて実装)。
//   runReview(n, ledger, scope) -> { reviewPath, findings, counts }   // architecture-review を実行
//   runFix(n, targetIds)        -> { results }                         // architecture-fix の wave 実行に委譲
//   runVerify()                 -> { green: boolean, output }          // 全体検証(委譲先 verify.md)
//   commitIter(n, msg)          -> { commit }                          // 緑確認後に周/wave 単位でコミット
// counts = {critical, high, medium, low}

const ledger = loadLedger()      // arch-loop/ledger.json(無ければ空)
const history = []               // 各周の counts/progress を蓄積(収束カーブ)
let prevHigh = Infinity          // 前周の critical+high
let stopReason = 'budget'        // 既定: 予算消尽

for (let n = 1; n <= CFG.maxIterations; n++) {
  // ── Review: 調査を委譲。差分スコープ + 周期的な全体スイープ ─────────────
  phase('Review')
  const scope = (n === 1 || n % CFG.fullSweepEvery === 0) ? 'full' : 'diff'
  const { findings, counts } = await runReview(n, ledger, scope)
  log(`iter${n} review: c=${counts.critical} h=${counts.high} m=${counts.medium} l=${counts.low} (${scope})`)

  // ── 収束(成功)判定はレビュー直後にも見る: 既にゲート達成なら何もせず終了 ──
  if (CFG.qualityGate(counts)) { stopReason = 'converged'; history.push({ n, counts, progress: 0 }); break }

  // ── Select: 台帳で絞った今周の対象集合 ───────────────────────────────
  const targets = findings.filter(f =>
    CFG.applyPolicy.severities.includes(f.severity) &&
    CFG.applyPolicy.roadmap.includes(f.roadmap) &&
    ledger[f.id]?.status !== 'wontfix' &&
    !(ledger[f.id]?.status === 'failed' && (ledger[f.id]?.attempts ?? 0) >= 2)
  )
  if (targets.length === 0) { stopReason = 'converged'; history.push({ n, counts, progress: 0 }); break }

  // ── Fix: 改善を委譲(wave 分割・並列適用・wave 検証・グループコミットは fix スキル) ──
  phase('Fix')
  const { results } = await runFix(n, targets.map(t => t.id))
  updateLedger(ledger, results, n)   // applied/failed(+attempts)/skipped を反映、上限超で wontfix 昇格

  // ── Verify: 周末の全体検証。赤=回帰 → 切り戻して停止 ──────────────────
  phase('Verify')
  const v = await runVerify()
  if (!v.green) {
    log(`iter${n} REGRESSION: 全体検証が赤。当該 wave を切り戻して停止。`)
    // ここで git restore/reset により当該周の変更を切り戻す(緑の周までを成果として残す)
    stopReason = 'regression'; history.push({ n, counts, progress: 0, regression: true }); break
  }
  commitIter(n, `refactor(arch): iter${n} 適用 (${results.filter(r=>r.status==='applied').map(r=>r.id).join(',')})`)

  // ── Decide: 進捗(デルタ)で頭打ち/振動を判定 ──────────────────────────
  phase('Decide')
  const high = counts.critical + counts.high
  const progress = Math.max(0, prevHigh - high)
  history.push({ n, counts, progress })
  saveLedger(ledger); saveSummary(n, { counts, progress, results })

  if (detectOscillation(ledger, history)) { stopReason = 'oscillation'; break }  // loop-control.md §5
  if (progress < CFG.progressThreshold)   { stopReason = 'diminishing'; break }  // 頭打ち(逓減)
  prevHigh = high
  // ここまで緑・進捗あり → 次周へ(barrier 通過)
}

return { stopReason, iterations: history, ledger }
```

## 委譲の実装メモ
- `runReview` / `runFix` は**既存スキルを起動するだけ**。本テンプレートに監査ロジックや wave 分割を
  書き込まない(再実装は DRY 違反で二重メンテになる)。実体は各スキルの SKILL/references に従う。
- 周内の並列(wave)は fix スキルの Workflow に内包される。本テンプレートの barrier は**周境界**のみ。
- review を別プロセス/別エージェントで回す場合、出力 JSON のパスを受け取って `findings`/`counts` を読む。

## コミットと証跡
- **緑を確認した周だけ**コミットを積む(`refactor(arch): iter<n> <要約>`)。回帰した周は切り戻し、コミットしない。
- 各周の `review.json` / `summary.json` と `ledger.json` を残す。これが Phase 4 の収束カーブの一次データ。
- 最終的に `stopReason`(converged / diminishing / budget / regression / oscillation)を必ずレポートに出す。

## 規模に応じた調整
- 1〜2周で収束する見込みなら Workflow 化せず、手番で「review → fix → 再 review で確認」でよい(判定基準は同じ)。
- 数周にわたり決定論的に回し証跡を残したいときに、このループ Workflow が活きる。
