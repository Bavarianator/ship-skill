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

## Installation in Claude Code

### Voraussetzungen

- [Claude Code](https://code.claude.com/docs/en/setup) im Terminal (`claude --version` prüfen)
- `git`
- Node/npm nur, wenn das **Zielprojekt** es braucht – der Skill selbst braucht nichts davon

### 1. Skill installieren (persönlich – in allen deinen Projekten)

```bash
git clone https://github.com/Bavarianator/ship-skill.git ~/.claude/skills/ship
```

Der Ordnername `ship` plus die Frontmatter `name: ship` in der `SKILL.md` ergeben den Befehl `/ship`.

### 2. Subagents installieren (nötig – der Skill ruft sie auf)

`/ship` arbeitet mit den Subagent-Typen `implementer` und `reviewer`. Ohne diese beiden Dateien schlägt jeder Lauf mit „agent type not found" fehl:

```bash
mkdir -p ~/.claude/agents
cp ~/.claude/skills/ship/agents/*.md ~/.claude/agents/
```

(Andere Agent-Typen des Skills – `Explore`, `Plan`, `general-purpose` – sind eingebaut.)

### 3. Installation prüfen

```bash
claude
```

- `/skills` eingeben → `ship` muss unter **personal** stehen
- `/ship` eintippen → Autocomplete zeigt den Hint `<aufgabe> [--branch] [--worktrees] [--workflow]`
- im `/`-Menü erscheinen `implementer` und `reviewer` als Subagent-Typen

Hast du die Ordner erst **während** einer laufenden Session angelegt, damit sie sofort sichtbar werden:

```
/reload-skills
```

Sonst reicht ein Neustart von `claude`. Danach einmal kurz testen, z. B. in einem Git-Repo:

```
/ship lese den Projektstand und schlage vor, was als Nächstes zu tun wäre
```

### Alternative A: nur für ein einzelnes Projekt

```bash
cd <pfad-zum-projekt>
git clone https://github.com/Bavarianator/ship-skill.git .claude/skills/ship
mkdir -p .claude/agents && cp .claude/skills/ship/agents/*.md .claude/agents/
git add .claude && git commit -m "Add ship skill"
```

Damit bekommt das ganze Team den Skill, sobald der Commit geteilt wird. Ohne `git add` bleibt es eine lokale Änderung.

### Alternative B: Repo woanders klonen und verlinken

Nützlich, wenn du die Dateien im Editor bearbeiten willst:

```bash
git clone https://github.com/Bavarianator/ship-skill.git ~/src/ship-skill
ln -s ~/src/ship-skill ~/.claude/skills/ship
cp ~/src/ship-skill/agents/*.md ~/.claude/agents/
```

Claude Code folgt Symlinks auf Skill-Ordner. Der Vorteil: `git pull` im geklonten Repo aktualisiert den Skill, und Änderungen sind sofort da (Live-Change-Detection, kein Neustart nötig).

### Priorität bei Namenskonflikten

Gibt es den Befehl `/ship` an mehreren Stellen, gewinnt: Enterprise vor personal vor project. Eigene Skills ersetzen eingebaute Befehle gleichen Namens.

### Aktualisieren

```bash
git -C ~/.claude/skills/ship pull          # persönliche Installation
git -C <projekt>/.claude/skills/ship pull  # Projekt-Installation
```

Änderungen an `SKILL.md` werden in laufenden Sessions automatisch übernommen.

### Deinstallieren

```bash
rm -rf ~/.claude/skills/ship
rm -f  ~/.claude/agents/implementer.md ~/.claude/agents/reviewer.md
```

Bei einer Projekt-Installation stattdessen `rm -rf <projekt>/.claude/skills/ship` und die beiden Dateien in `<projekt>/.claude/agents/`. Die Subagents nur löschen, wenn du sie nicht für andere Zwecke nutzt.

### Fehlerbehebung

| Symptom | Ursache / Fix |
|---|---|
| `/ship` fehlt im Menü | Ordner heißt nicht `ship` oder `SKILL.md` liegt nicht direkt darin → `ls ~/.claude/skills/ship` |
| „agent type not found" (implementer/reviewer) | Schritt 2 ausgeführt? `ls ~/.claude/agents/` |
| Skill läuft, findet aber kein Repo | `/ship` braucht ein gestartetes Git-Repo im Session-Verzeichnis; kein Repo → Phase 0 führt `git init` an |
| Altes Verhalten nach Update | Session neu starten bzw. `/reload-skills` |
| Zwei `/ship`-Befünde | Personal- und Projekt-Installation überschneiden sich → eine entfernen |

## Dateien

- **[SKILL.md](SKILL.md)** – der Skill selbst: Frontmatter, Phasen 0–4, Vorlagen für `implementer` und `reviewer`.
- **[workflow.md](workflow.md)** – Hybrid-Modus für große Läufe über das Workflow-Tool (Build + Review + Fix pro Welle als Workflow, Integration inline).
- **[agents/implementer.md](agents/implementer.md)** – Subagent, der die parallelen Bau-Tasks umsetzt.
- **[agents/reviewer.md](agents/reviewer.md)** – read-only Reviewer für Spec-Treue und Code-Qualität.

## Verhalten in Projekten mit eigenen Regeln

Projekt-CLAUDE.md und Projekt-Memories haben Vorrang: Steht dort „keine Subagents", führt `/ship` alle Phasen seriell selbst aus.
