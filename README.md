# ship – Plan → Build → Review → Ship für Claude Code

Ein Claude-Code-Skill, der eine Aufgabe komplett von der Planung bis zum Commit/PR durchzieht: Wellenweise parallel bauen, jeden Task sofort reviewen, Befunde prüfen, fixen, verifizieren und committen.

## Was der Skill macht

```
/ship <aufgabe>
```

1. **Ausgangslage** – Repo-Status, Branch, letzte Commits, Worktrees, Baseline-Build/Tests.
2. **Plan** – Aufgabe nach Größe in S/M/L schneiden, Tasks je 2–5 Min mit exakten Dateien, Schnittstellen und prüfbaren Akzeptanzkriterien.
3. **Build** – Tasks parallel von `implementer`-Agents bauen: gemeinsamer Checkout (≤ 3 Tasks) oder eigene Worktrees (ab 4 Tasks / `--worktrees`).
4. **Review** – pro Task ein `reviewer`, am Ende 2–4 Linsen-Reviewer (Integration, Sicherheit, Einfachheit, Tests), Befunde erst prüfen, dann fixen (max. 3 Runden).
5. **Ship** – Build/Lint/Tests selbst verifizieren, je Task ein Commit, Push/PR nur wenn vorgesehen – nie Merge, Deploy, Release oder bezahlte Schritte ohne Nachfrage.

## Flags

| Flag | Wirkung |
|---|---|
| `--branch` | eigenen Branch `<typ>/<kurzname>` + PR/MR |
| `--worktrees` | jeder parallele Task im eigenen Worktree |
| `--workflow` | große Läufe über das Workflow-Tool ([workflow.md](workflow.md)) |

Ohne Argumente wird die zuletzt im Chat beschriebene Aufgabe übernommen.

## Installation

Repo klonen und den Skill-Ordner nach `~/.claude/skills/` kopieren:

```bash
git clone https://github.com/Bavarianator/ship-skill.git
mkdir -p ~/.claude/skills/ship
cp ship-skill/SKILL.md ship-skill/workflow.md ~/.claude/skills/ship/
```

Danach in Claude Code `/ship <aufgabe>` aufrufen. Projekt-CLAUDE.md und Projekt-Memories haben Vorrang (z. B. „keine Subagents“ → alle Phasen seriell).

## Dateien

- **[SKILL.md](SKILL.md)** – der Skill selbst: Frontmatter, Phasen 0–4, Vorlagen für `implementer` und `reviewer`.
- **[workflow.md](workflow.md)** – Hybrid-Modus für große Läufe über das Workflow-Tool (Build + Review + Fix pro Welle als Workflow, Integration inline).

## Voraussetzungen

- Claude Code mit Subagent-Support (`implementer`, `reviewer`, `Explore`, `Plan`)
- Git im Projekt; für PRs zusätzlich `gh` (GitHub) oder `glab` (GitLab)
