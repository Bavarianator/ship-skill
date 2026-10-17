---
name: ship
description: "Plan → Build → Review → Ship mit vielen parallelen Subagents: Aufgabe in Wellen schneiden, Tasks parallel von implementer-Agents bauen lassen (gemeinsamer Checkout oder eigene Worktrees), jeden Task sofort reviewen, Befunde prüfen, verifizieren und auf dem aktuellen Branch committen – eigener Branch und PR nur bei Bedarf. Nutzen bei /ship <aufgabe> oder wenn ein Feature/Fix komplett von Plan bis Commit/PR durchgezogen werden soll."
argument-hint: "<aufgabe> [--branch] [--worktrees] [--workflow]"
allowed-tools: Bash(git rev-parse *) Bash(git status *) Bash(git log *) Bash(git remote *) Bash(git worktree list *) Bash(head *)
---

# /ship – Plan → Build → Review → Ship

Aufgabe: $ARGUMENTS
(Leer → die zuletzt im Chat beschriebene Aufgabe.) Flags: `--branch` eigener Branch + PR · `--worktrees` jeder parallele Task im eigenen Worktree · `--workflow` große Läufe über das Workflow-Tool ([workflow.md](workflow.md)).

Eigenständig bis zum Ende durchziehen. Projekt-CLAUDE.md und Projekt-Memories haben Vorrang (z. B. „keine Subagents“ → alle Phasen seriell selbst).

## Ausgangslage
```!
git rev-parse --abbrev-ref HEAD 2>&1 || true
git status --short 2>&1 | head -25 || true
git remote -v 2>&1 | head -2 || true
git log --oneline -6 2>&1 || true
git worktree list 2>&1 || true
```
Der Block läuft im Session-Verzeichnis. Nennt die Aufgabe ein anderes Repo, dieselben Befehle dort ausführen.

## Rollen
- **Du orchestrierst:** planen, verteilen, integrieren, entscheiden. Code schreiben die implementer (nur bei Größe S du selbst). Lies nur, was du zum Entscheiden brauchst (Reports, `git diff --stat`, Stellen, auf die Befunde zeigen), damit dein Kontext für die Integration frei bleibt.
- Agents: `Explore` sucht, `Plan` plant (nur L), `implementer` baut, `reviewer` prüft read-only, `general-purpose` recherchiert (z. B. aktuelle API-Doku).
- Alles Unabhängige in **einer** Nachricht parallel starten. Breite nach Bedarf: 2–5 Explore, bis ~8 implementer je Welle, je Task ein reviewer, am Ende Linsen-Reviewer.
- Mechanische Tasks (1–2 Dateien, klares Muster) mit `model: "sonnet"`, Architektur und knifflige Integration mit dem Standardmodell.
- Prompts sind selbsttragend, der Agent sieht den Chat nicht (Vorlagen unten).
- SHAs als Text notieren, nicht in Shell-Variablen: Die überleben keinen Bash-Aufruf.

## Phase 0 – Ausgangslage
1. Kein Repo oder noch kein Commit → `git init` + Initial-Commit, nur im Projektordner (nie im Home-Verzeichnis; im Zweifel fragen).
2. **Branch:** Standard ist der aktuelle Branch. Einen neuen Branch `<typ>/<kurzname>` gibt es nur in drei Fällen: `--branch`, die Projektregeln verlangen PRs, oder HEAD ist detached.
3. **BASE** = `git rev-parse HEAD` notieren: Sicherungspunkt und Diff-Basis.
4. Uncommittete Änderungen nie verwerfen oder stashen:
   - gehören zur Aufgabe (aus diesem Chat) → Teil der Arbeit;
   - fremde WIP → bleibt uncommittet. Stand sichern, damit Reviewer Fremdes erkennen: `git diff HEAD > "$(git rev-parse --git-dir)/ship-wip.patch"` (untracked Dateien zusätzlich nennen). Überschneidet sie sich mit Dateien der Aufgabe → der ganze Lauf im gemeinsamen Checkout (Worktrees sehen nur Commits).
5. Build-, Test- und Lint-Befehle ermitteln (package.json, Makefile, gradle, pyproject, CLAUDE.md …). Die Baseline mit `run_in_background` starten, während du planst. Sie muss fertig sein, bevor Welle 1 Dateien ändert. Vorher rote Tests sind nicht unsere.
6. Commit-Stil aus `git log` übernehmen (Sprache, Präfixe, Granularität).

## Phase 1 – Plan
| Größe | Merkmal | Vorgehen |
|---|---|---|
| S | Diff in einem Satz beschreibbar, ≤ 3 Dateien | selbst bauen, 1 reviewer, weiter mit Phase 4 |
| M | mehrere unabhängige Stränge | Explore-Fan-out → Plan → Wellen → Reviews |
| L | > ~8 Tasks oder mehrere Subsysteme | wie M + `Plan`-Agent + Plan-Review + Ledger; mit `--workflow` per Workflow-Tool |

- Verstehen: 2–5 `Explore`-Agents parallel, je ein Bereich, Ausgabe ≤ 15 Zeilen mit file:line. Entfällt, wenn die betroffenen Dateien schon bekannt sind. Bei externen APIs/Libs zusätzlich ein `general-purpose` für aktuelle Doku.
- Task (je 2–5 min) = Ziel, exakte Dateien, Schnittstellen (liefert an / nutzt von Task X), prüfbares Akzeptanzkriterium (Befehl → erwartetes Ergebnis).
- **Datei-Ownership:** Jede Datei gehört in einer Welle genau einem Task. Welle 1 = gemeinsame Grundlagen (Typen, Schema, Interfaces, Config). Danach breit schneiden: lieber 6 kleine als 2 große Tasks. Tests können eigene Tasks sein, sobald die Schnittstelle steht.
- L: Ein `reviewer` prüft den Plan vor dem Bau (fehlende Tasks, Dateikonflikte in einer Welle, Reihenfolge). Fortschritt in `$(git rev-parse --git-dir)/ship-ledger.md` führen (`T3: fertig, a1b2c3d, APPROVED`). Die Datei liegt in .git, wird nie committet und übersteht Kontext-Kompaktierung.
- Plan als kurze Tabelle (Welle · Task · Dateien) ausgeben, Annahmen nennen, ohne Rückfrage weiter.

## Phase 2 – Build (pro Welle)
**Modus wählen:**
- **Gemeinsamer Checkout** (Standard bis 3 parallele Tasks; bei überlappender fremder WIP immer, auch bei mehr Tasks): implementer arbeiten direkt im Repo an disjunkten Dateien, committen nicht und laufen nur gezielte Checks für ihre Dateien. Voll-Builds macht nur du. Keine Branches, nichts aufzuräumen.
- **Worktrees** (ab 4 parallelen Tasks, bei langen Builds/Tests pro Task oder mit `--worktrees`): selbst vom aktuellen Stand anlegen. **Nicht** `isolation: "worktree"` nutzen, denn das zweigt ohne `worktree.baseRef: "head"` in den Settings vom Remote-Default-Branch ab. Vorher committen, worauf die Welle aufbaut, und **WELLEN-BASIS** = `git rev-parse HEAD` notieren. Pro Task:
  ```bash
  git check-ignore -q .claude/worktrees/ || echo '.claude/worktrees/' >> "$(git rev-parse --git-common-dir)/info/exclude"
  git worktree add -b ship/<task> .claude/worktrees/ship-<task> HEAD
  ```
  Existiert `ship/<task>` schon (abgebrochener Lauf), erst ansehen, nicht überschreiben. Ein Worktree ist ein frischer Checkout: Abhängigkeiten installieren (`npm ci` …), gitignorierte Dateien wie `.env` bei Bedarf kopieren.

**Starten:** alle Tasks der Welle in einer Nachricht (Vorlage A).

**Je Task, sobald er fertig ist** (nicht auf die ganze Welle warten):
1. Status `NEEDS_CONTEXT` → Kontext per `SendMessage` an denselben Agent. `BLOCKED` → Task neu schneiden oder selbst übernehmen. `DONE` oder `DONE_WITH_CONCERNS` → weiter, Concerns an den reviewer geben.
2. reviewer starten (Phase 3), **vor** der Übernahme. Diff:
   - gemeinsamer Checkout: `git diff -- <dateien>` plus die neuen Dateien;
   - Worktree: `git -C <pfad> diff <WELLEN-BASIS>..HEAD`.
3. Nach `APPROVED` übernehmen, als **ein Commit je Task** inklusive Fix-Runden:
   - gemeinsamer Checkout: `git add <dateien>` + commit. Dateien mit fremder WIP nicht committen, sie kommen in den Bericht;
   - Worktree: `git cherry-pick -n <WELLEN-BASIS>..ship/<task>` + commit, Konflikte selbst lösen. Danach `git worktree remove --force .claude/worktrees/ship-<task>` und `git branch -D ship/<task>`.

**Welle abschließen:** Build + betroffene Tests selbst laufen lassen. Rot → erst fixen, dann die nächste Welle.

## Phase 3 – Review
- **Pro Task** ein `reviewer` (Vorlage B), sobald der implementer fertig meldet: erst Spec-Treue, dann Korrektheit.
- **Am Ende (M/L)** 2–4 Linsen-Reviewer parallel über `git diff <BASE>..HEAD` (plus `git diff -- <uncommittete Task-Dateien>`), nur die passenden:
  1. Integration & Korrektheit: Passen die Teile zusammen? Lücken zwischen Tasks, Edge Cases?
  2. Sicherheit: nur wenn Eingaben, Auth, Netzwerk, Dateipfade oder Secrets berührt sind.
  3. Einfachheit: doppelte Logik, Unverlangtes, toter Code.
  4. Tests: Decken sie die Akzeptanzkriterien? Schlagen sie ohne die Änderung fehl?
- **Befunde prüfen, bevor gefixt wird:** Kritisch/Wichtig an der zitierten Stelle selbst nachlesen. Bei Zweifel soll ein zweiter `reviewer` den Befund widerlegen. Stil-Nitpicks und schon bestehende Probleme verwerfen.
- **Fix-Eskalation:**
  1. Befunde wörtlich per `SendMessage` an denselben implementer (er behält seinen Kontext).
  2. Danach ein frischer implementer mit dem Standardmodell und allen Befunden.
  3. Danach selbst entscheiden und im Bericht begründen.

  Unabhängige Fixes parallel, danach nur die betroffenen Reviews wiederholen. Höchstens 3 Reviews je Task. Fixes aus dem Abschluss-Review bekommen einen eigenen Commit.

## Phase 4 – Verifizieren & Ship
- Volle Verifikation selbst: Build, Lint/Typecheck, alle Tests; Apps einmal starten (Skill `run`). Mit der Baseline vergleichen: Test oder getesteter Code in diesem Lauf geändert → unser Fehler, fixen. Sonst war er vorher schon rot → melden. Im Zweifel unser Fehler.
- Rot und Ursache nach einem Blick unklar → 2–4 read-only Agents parallel, je eine Hypothese belegen oder widerlegen. Ursache fixen, nie Tests abschwächen. Nach 3 gescheiterten Fixes den Ansatz hinterfragen statt weiter zu flicken.
- `git diff --stat <BASE>..HEAD`, `git status` und `git worktree list` sichten: nichts Unerwartetes, keine Secrets, keine Debug-Reste, keine `ship/*`-Branches oder Worktrees übrig. `ship-wip.patch` und `ship-ledger.md` aus dem git-Verzeichnis entfernen, sonst verwirren sie den nächsten Lauf.
- Commits: jeder lauffähig und im Projektstil, einer je Task (bei S einer), dazu ggf. ein Commit mit den Fixes aus dem Abschluss-Review. Fremde WIP bleibt uncommittet. Dateien mit eigenen **und** fremden Änderungen nicht committen, sondern im Bericht nennen.
- Push/PR (nie force):
  - neuer Branch aus Phase 0 → pushen und PR/MR öffnen (`gh pr create` / `glab mr create`, gibt es schon einen → aktualisieren). Inhalt: Was/Warum, Verifikation (Befehle + Ergebnis), offene Punkte. Kein Remote → nur lokal.
  - anderer Branch mit Upstream (nicht Default) → pushen. Ohne Upstream → nur lokal.
  - Default-Branch (main/master) → nur lokal committen. Pushen nur, wenn die Projektregeln es vorsehen, denn ein Push kann CI-Minuten oder Deploys auslösen.
- **Nie ohne Nachfrage:** Merge nach main, Deploy, Release/Publish, alles was Geld kostet.

## Abschlussbericht (kurz)
- **Gebaut:** Wellen/Tasks, Commits (SHAs), Branch/PR-Link, Sicherungspunkt BASE
- **Geprüft:** Befehle + Ergebnis vs. Baseline, Review-Verdicts, verworfene Befunde (eine Zeile)
- **Offen:** ungelöste Befunde, uncommittete Dateien (warum), was nur lokal liegt, nötige Freigaben (Push, Deploy)

## Vorlage A – implementer
```
Gesamtziel: <1–2 Sätze>
Task <Tn>: <Text>
Arbeitsverzeichnis: <Repo-Pfad | absoluter Worktree-Pfad – nur dort arbeiten, Pfade absolut>
Dateien (nur diese ändern/anlegen): <Liste>
Schnittstellen: <Signaturen/Typen aus früheren Tasks mit file:line>
Akzeptanz: <Befehl> → <erwartetes Ergebnis>
Checks: <gezielte Befehle für deine Dateien>
Parallel laufen: <andere Tasks + deren Dateien>. Fehler in fremden Dateien nicht anfassen, nur melden.
Commit: <nein | ja, im Worktree: "<Nachricht im Projektstil>">
Bericht (max. 15 Zeilen): Status DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED,
geänderte Dateien, Checks mit Ergebnis, Commit-SHA, offene Punkte.
```

## Vorlage B – reviewer
```
Spec: <Task-Text inkl. Akzeptanz> | Linse: <eine der vier Linsen>
Änderung: <Diff-Befehl aus Phase 2 bzw. git diff <BASE>..HEAD>  (in <verzeichnis>)
Fremde WIP vor dem Lauf (falls vorhanden, nicht bewerten): <git-dir>/ship-wip.patch
Kontext: <Gesamtziel, Nachbar-Tasks, Projektkonventionen>
Nur stichhaltige Befunde, jeder mit file:line und zitiertem Code.
Kein Stil, keine Linter-Funde, nichts, was vorher schon so war.
```
