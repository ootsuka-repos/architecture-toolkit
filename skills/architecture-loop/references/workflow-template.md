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

// 以下はすべて擬似シグネチャ＝骨格。実体は実装者が用意する(本テンプレートには定義を置かない)。
// (A) skill 委譲関数(プロジェクトの起動方法に合わせて実装。実体化は下記「委譲の実装メモ」参照):
//   runReview(n, ledger, scope) -> { reviewPath, findings, counts, roadmap }  // architecture-review を実行
//   runFix(n, targetIds)        -> { results }                         // architecture-fix の wave 実行に委譲
//   runVerify()                 -> { green: boolean, output }          // 全体検証(委譲先 verify.md)
//   commitIter(n, msg)          -> { commit }                          // 緑確認後に周/wave 単位でコミット
// (B) 局所ヘルパ(台帳・サマリの I/O と振動検知。実装者が下記契約どおりに実装する):
//   loadLedger()                -> ledger(object)                     // arch-loop/ledger.json を読む(無ければ {})。loop-control.md §4
//   updateLedger(ledger, results, n) -> void                         // applied/failed(+attempts)/skipped を反映、上限超で wontfix 昇格。loop-control.md §4
//   saveLedger(ledger)          -> void                              // ledger を arch-loop/ledger.json に永続化
//   saveSummary(n, summary)     -> void                              // arch-loop/iter-<n>/summary.json に件数/progress/results を書く
//   detectOscillation(ledger, history) -> boolean                    // 出戻り/純増ゼロ往復を検知。loop-control.md §5
// counts  = {critical, high, medium, low}
// roadmap = {now:[{finding_id}], next:[...], later:[...]}  // output-schema.md のトップレベル roadmap(正本=SSOT)

const ledger = loadLedger()      // arch-loop/ledger.json(無ければ空)
const history = []               // 各周の counts/progress を蓄積(収束カーブ)
let prevHigh = Infinity          // 前周の critical+high
let stopReason = 'budget'        // 既定: 予算消尽

for (let n = 1; n <= CFG.maxIterations; n++) {
  // ── Review: 調査を委譲。差分スコープ + 周期的な全体スイープ ─────────────
  phase('Review')
  const scope = (n === 1 || n % CFG.fullSweepEvery === 0) ? 'full' : 'diff'
  const { findings, counts, roadmap } = await runReview(n, ledger, scope)
  log(`iter${n} review: c=${counts.critical} h=${counts.high} m=${counts.medium} l=${counts.low} (${scope})`)

  // roadmap は finding 直下ではなくトップレベルの mapping(SSOT)。finding.id → bucket を解決して選定に使う。
  const roadmapOf = {}
  for (const bucket of ['now', 'next', 'later'])
    for (const it of (roadmap?.[bucket] ?? [])) roadmapOf[it.finding_id] = bucket

  // ── 収束(成功)判定はレビュー直後にも見る: 既にゲート達成なら何もせず終了 ──
  if (CFG.qualityGate(counts)) { stopReason = 'converged'; history.push({ n, counts, progress: 0 }); break }

  // ── Select: 台帳で絞った今周の対象集合 ───────────────────────────────
  const targets = findings.filter(f =>
    CFG.applyPolicy.severities.includes(f.severity) &&
    CFG.applyPolicy.roadmap.includes(roadmapOf[f.id]) &&
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
- **Workflow スクリプト内ではスキルを直接「起動」できない**(使えるのは `agent()`/`parallel()`/`pipeline()`/
  `workflow()`)。委譲関数は次のどちらかで実体化する:
  - `runReview` → review スキルの手順を渡した `agent()`(大規模なら `parallel()` でサブシステム並列)。
    出力 JSON を書かせ、そのパスから `{findings, counts, roadmap}` を読む。
  - `runFix` → fix スキルの Workflow を **`workflow('apply-arch-fixes', { steps })` で子として呼ぶ**
    (`architecture-fix/references/workflow-template.md` の雛形)。中身の wave 分割・並列適用は fix 側に委ねる。**ただし名前指定 `workflow('apply-arch-fixes', …)` は、その Workflow が保存/登録済み(例: `.claude/workflows/` に実体がある)のときだけ解決される**(未知の名前は throw)。雛形が .md に埋め込まれているだけなら、(a) 先に名前付き Workflow として登録するか、(b) 雛形の JS を実体 `.js` に書き出して `workflow({ scriptPath: '<書き出した.js>' }, { steps })` で呼ぶ。**`.md` をそのまま `scriptPath` に渡しても JS として実行できない**点に注意。
- **入れ子は1段まで**: 本ループ(トップ)から `workflow()`/`agent()` を呼ぶのは可。ただし**子の中でさらに
  `workflow()` を呼ぶと throw する**。fix の Workflow は内部で `agent()`/`parallel()` のみを使う(さらに nest しない)
  ので、ループ→fix の1段で成立する。review/fix を別々の子 workflow にしても、どちらも深さ1なら同居できる。
- 周内の並列(wave)は fix スキルの Workflow に内包される。本テンプレートの barrier は**周境界**のみ。

## コミットと証跡
- **緑を確認した周だけ**コミットを積む(`refactor(arch): iter<n> <要約>`)。回帰した周は切り戻し、コミットしない。
- 各周の `review.json` / `summary.json` と `ledger.json` を残す。これが Phase 4 の収束カーブの一次データ。
- 最終的に `stopReason`(converged / diminishing / budget / regression / oscillation)を必ずレポートに出す。

## 規模に応じた調整
- 1〜2周で収束する見込みなら Workflow 化せず、手番で「review → fix → 再 review で確認」でよい(判定基準は同じ)。
- 数周にわたり決定論的に回し証跡を残したいときに、このループ Workflow が活きる。
