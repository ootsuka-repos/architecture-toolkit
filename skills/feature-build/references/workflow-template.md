# workflow-template.md — Workflow スクリプト雛形（Phase 3）

planning の `waves` を Workflow で実行する。**wave 単位の barrier** が要る(次 wave は前 wave 統合＋検証 PASS 後に
開始)ので pipeline ではなく、wave ごとに `parallel()` → 検証 → コミット を順に回す。

実装本体 wave は並列、配線 wave は中央逐次(SERIAL)。各 subagent は**自分の files 以外を触らない**ことが前提。

```javascript
export const meta = {
  name: 'feature-build-apply',
  description: 'Build a feature by applying implementation tasks wave by wave with verification',
  phases: [
    { title: 'Foundation' },
    { title: 'Implement' },
    { title: 'Wire' },
  ],
}

// planning の出力をここに埋める（実際は args で渡す/インラインで構築）
const WAVES = args.waves   // [{ serial?: bool, phase: string, tasks: [...] }, ...]

const RESULT = {
  type: 'object',
  required: ['id', 'status', 'files_changed', 'verify_passed'],
  properties: {
    id: { type: 'string' },
    status: { enum: ['done', 'failed', 'blocked'] },
    files_changed: { type: 'array', items: { type: 'string' } },
    tests_added: { type: 'array', items: { type: 'string' } },
    verify_passed: { type: 'boolean' },
    verify_output: { type: 'string' },
    notes: { type: 'string' },
  },
}

function buildPrompt(t) {
  return [
    `新機能の実装タスク ${t.id}: ${t.title}`,
    `設計仕様(契約): ${JSON.stringify(t.spec)}`,
    `触れてよいファイル(これ以外は絶対に編集しない): ${t.files.join(', ')}`,
    `依存先で確定済みの型/IF: ${t.deps_context ?? '(なし)'}`,
    `受け入れ条件: ${t.acceptance ?? t.title}`,
    '',
    '指示:',
    '- 既存コードの同種パターンに倣って実装する（配置/命名/レイヤ/エラー処理/テストの作法を踏襲）。',
    '- 実装と一緒にテストを書く（重要パスはテスト先行）。',
    `- 完了したら検証を実行する: ${t.verify}`,
    '- 指定ファイル以外には触れない（他タスクと並列実行中のため）。',
    '- 結果を構造化(RESULT スキーマ)で返す。',
  ].join('\n')
}

const report = []
for (const wave of WAVES) {
  phase(wave.phase)
  let results
  if (wave.serial) {
    // 配線 wave: 中央で1つずつ
    results = []
    for (const t of wave.tasks) {
      results.push(await agent(buildPrompt(t), { label: `wire:${t.id}`, phase: wave.phase, schema: RESULT }))
    }
  } else {
    // 実装 wave: ファイル素集合なので並列
    results = await parallel(
      wave.tasks.map(t => () => agent(buildPrompt(t), { label: t.id, phase: wave.phase, schema: RESULT }))
    )
  }
  report.push(...results.filter(Boolean))

  // ここで main 側の全体検証（verify.md）を行い、PASS したらこの wave をグループコミットする。
  // 検証が赤なら停止して切り分ける（次 wave に進まない）。
  const failed = results.filter(Boolean).filter(r => r.status !== 'done' || !r.verify_passed)
  if (failed.length) {
    log(`wave "${wave.phase}" で失敗: ${failed.map(f => f.id).join(', ')} — 停止`)
    return { stoppedAt: wave.phase, report }
  }
  log(`wave "${wave.phase}" 完了 → 全体検証してグループコミット`)
}

return { report }
```

## 運用メモ
- **全体検証とコミットは main 側(このスクリプトの外/各 wave 後)で実行**する。スクリプトはファイル編集を
  subagent に任せ、検証・コミットの判断はオーケストレータが握る。
- subagent の `spec`(設計仕様/契約)より実コードが優先。乖離があれば subagent は整合させ、無理なら `blocked` で返す。
- 配線 wave で結合テストを通し、最後に Phase 0 の受け入れ条件を1つずつ確認する。
- ファイル衝突が設計上どうしても避けられない wave だけ `isolation: 'worktree'` にフォールバックする。
