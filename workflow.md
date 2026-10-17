# /ship --workflow – große Läufe über das Workflow-Tool

Nur bei `--workflow` oder aktivem ultracode: Erst das zählt als ausdrückliche Zustimmung zu vielen Agents. Vorher den Skill `workflow-authoring` laden.

**Hybrid:** Phasen 0–1, die Integration und Phase 4 machst du inline. Pro Welle läuft ein Workflow für Bauen + Review + Fix. Zwischen den Wellen liest du die Ergebnisse und entscheidest. Überschneidet sich fremde WIP mit Dateien der Aufgabe, gibt es keinen Workflow-Modus: Der ganze Lauf bleibt im gemeinsamen Checkout (SKILL.md, Phase 0).

## Pro Welle
1. Inline: committen, worauf die Welle aufbaut. WELLEN-BASIS = `git rev-parse HEAD` notieren und pro Task einen Worktree anlegen (SKILL.md, Phase 2).
2. Workflow mit dem Skript unten starten, `args` als echtes JSON-Objekt:
   `{goal, base: "<WELLEN-BASIS>", tasks: [{id, text, files: [...], interfaces, accept, checks, path, commitMsg}]}` (`path` = absoluter Worktree-Pfad).
3. Inline: `APPROVED`-Tasks übernehmen (`git cherry-pick -n <WELLEN-BASIS>..ship/<id>` + commit), Worktrees und Branches entfernen, Build/Tests laufen lassen, Ledger aktualisieren. Für `REQUEST_CHANGES`, `NOT_BUILT` und `REVIEW_FAILED` gilt die Fix-Eskalation aus Phase 3 ab Schritt 2, inline (Workflow-Agents sind beendet, `SendMessage` geht nicht).

## Skript
```js
export const meta = {
  name: 'ship-wave',
  description: 'Eine /ship-Welle: Tasks parallel in vorbereiteten Worktrees bauen, je Task reviewen und fixen',
  phases: [{ title: 'Build' }, { title: 'Review' }, { title: 'Fix' }],
}
const IMPL = { type: 'object', properties: {
  status: { type: 'string', enum: ['DONE', 'DONE_WITH_CONCERNS', 'NEEDS_CONTEXT', 'BLOCKED'] },
  commit: { type: 'string', description: 'SHA des Commits im Worktree, leer wenn nicht committet' },
  checks: { type: 'string' }, notes: { type: 'string' } },
  required: ['status', 'commit', 'checks'] }
const REVIEW = { type: 'object', properties: {
  verdict: { type: 'string', enum: ['APPROVED', 'REQUEST_CHANGES'] },
  findings: { type: 'array', items: { type: 'string' } } },
  required: ['verdict', 'findings'] }

const built = r => r && r.commit && (r.status === 'DONE' || r.status === 'DONE_WITH_CONCERNS')
const others = t => args.tasks.filter(o => o.id !== t.id).map(o => `${o.id}: ${o.files.join(', ')}`).join('; ')
const brief = t => `Gesamtziel: ${args.goal}
Task ${t.id}: ${t.text}
Arbeitsverzeichnis: ${t.path} – nur dort arbeiten, Pfade absolut.
Dateien (nur diese ändern/anlegen): ${t.files.join(', ')}
Schnittstellen: ${t.interfaces || '–'}
Akzeptanz: ${t.accept}
Checks: ${t.checks}
Parallel laufen: ${others(t)}. Fehler in fremden Dateien nicht anfassen, nur melden.
Am Ende im Worktree committen: "${t.commitMsg}"`

const review = (t, round) => agent(`Spec: ${t.text}
Akzeptanz: ${t.accept}
Änderung: git -C ${t.path} diff ${args.base}..HEAD
Gesamtziel: ${args.goal}
Nur stichhaltige Befunde, jeder mit file:line und zitiertem Code. Kein Stil, nichts, was vorher schon so war.`,
  { agentType: 'reviewer', phase: 'Review', label: `review:${t.id}#${round}`, schema: REVIEW })

const results = await pipeline(args.tasks,
  t => agent(brief(t), { agentType: 'implementer', phase: 'Build', label: `build:${t.id}`, schema: IMPL }),
  async (impl, t) => {
    if (!built(impl)) return { id: t.id, verdict: 'NOT_BUILT', impl }
    let rev = await review(t, 1)
    const fixNotes = []
    for (let round = 2; rev && rev.verdict === 'REQUEST_CHANGES' && round <= 3; round++) {
      const fix = await agent(`${brief(t)}

Der Task ist im Worktree schon umgesetzt und committet. Behebe diese Review-Befunde. Prüfe jeden vorher an der Stelle und lehne Unbegründetes mit Begründung ab. Danach erneut committen:
- ${rev.findings.join('\n- ')}`,
        { agentType: 'implementer', phase: 'Fix', label: `fix:${t.id}#${round}`, schema: IMPL })
      if (!fix || fix.status === 'BLOCKED' || fix.status === 'NEEDS_CONTEXT') break
      fixNotes.push(fix.notes || '')
      rev = await review(t, round)
    }
    return { id: t.id, verdict: rev ? rev.verdict : 'REVIEW_FAILED', findings: rev ? rev.findings : [], fixNotes, impl }
  })
return args.tasks.map(t => results.find(r => r && r.id === t.id) || { id: t.id, verdict: 'NOT_BUILT' })
```
