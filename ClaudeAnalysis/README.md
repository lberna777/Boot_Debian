# Il Cervello di Claude — Architettura del Setup

Documentazione di come è configurato Claude Code su questa macchina.
Aggiornare questo file ogni volta che si aggiunge un plugin, una skill, un hook o un progetto con memoria persistente.

---

## Strati di Personalizzazione

```
┌─────────────────────────────────────────────┐
│              CLAUDE CODE CLI                │
├─────────────────────────────────────────────┤
│  Plugin: superpowers (ufficiale)            │  ← skill di sistema, auto-trigger
│  Plugin: lorenzo-skills (locale)           │  ← skill dominio-specifiche
├─────────────────────────────────────────────┤
│  Hook: PreCompact → auto-handoff            │  ← safety net su compressione ctx
│  Status line: user | dir | model | ctx%    │  ← monitoraggio contesto in tempo reale
├─────────────────────────────────────────────┤
│  CLAUDE.md per progetto                    │  ← istruzioni vincolanti per contesto
│  Memoria persistente per progetto          │  ← profilo, feedback, stato
└─────────────────────────────────────────────┘
```

---

## Plugin

### superpowers `@claude-plugins-official` — v5.1.0
Plugin ufficiale Anthropic. Aggiunge 14 skill di sistema che si **auto-triggerano** senza intervento esplicito:

| Gruppo | Skill |
|--------|-------|
| Sviluppo | `test-driven-development`, `systematic-debugging`, `finishing-a-development-branch` |
| Architettura | `writing-plans`, `executing-plans`, `brainstorming` |
| Agenti | `dispatching-parallel-agents`, `subagent-driven-development`, `using-git-worktrees` |
| Review | `requesting-code-review`, `receiving-code-review`, `verification-before-completion` |
| Meta | `using-superpowers`, `writing-skills` |

Non si invocano con `/comando` — scattano automaticamente quando Claude rileva il momento giusto (es. `verification-before-completion` prima di dichiarare un task finito).

### lorenzo-skills `@lorenzo-local` — marketplace locale privato
Skill custom in `~/.claude/local-marketplace/plugins/lorenzo-skills/skills/`. Si invocano esplicitamente con `/nome`.

| Skill | Quando si usa | Progetto |
|-------|--------------|---------|
| `unicode-output-gate` | Prima di finalizzare lezioni/appunti/simulazioni | UniCode |
| `unicode-session-close` | Chiusura sessione studio — verifica file `stato/` | UniCode |
| `audio-dsp-debug` | Debug click/distorsione/CPU su VST o DAW | Idee/StereoCompressor |
| `game-scope-guard` | Prima di aggiungere feature a un gioco — previene scope creep | Idee |

**Come aggiungere una skill locale:**
```bash
mkdir -p ~/.claude/local-marketplace/plugins/lorenzo-skills/skills/<nome-skill>
# creare SKILL.md con frontmatter name/description/version
```

---

## Hook

### `PreCompact` → `~/.claude/hooks/precompact-handoff.sh`
Si esegue automaticamente **prima che Claude comprima il contesto**. Salva uno snapshot del progetto corrente in `plans/handoffs/HANDOFF_auto-precompact_DATA_ORA.md` con:
- Ultimi 15 commit git
- Diff uncommitted
- Stato build/test npm

**Perché esiste**: se il contesto si comprime a sorpresa (sessione lunga), la sessione successiva ha comunque un punto di partenza documentato senza azione manuale.

---

## Status Line

Script in `~/.claude/statusline-command.sh`. Mostra nella barra inferiore:
```
lorenzo@hostname ~/UniCode | Claude Sonnet 4.6 | ctx:43%
```
Il `ctx%` permette di sapere quando lanciare `/handoff` prima che il contesto si esaurisca.

---

## Memoria Persistente

Ogni progetto con sessioni continuative ha memoria in `~/.claude/projects/<progetto>/memory/`. La memoria persiste tra sessioni — Claude la legge all'avvio del contesto.

| Progetto | Cosa memorizza |
|----------|---------------|
| `UniCode` | Profilo studio, feedback workflow, anti-pattern per materia, stato moduli |
| `Idee` | Profilo music production (FL Studio, Waves), sistema handoff |
| `Idee/GitUnicode` | Obiettivi repo UniCode pubblica |
| `LifeManager` | Stack Next.js, moduli implementati, profilo utente |

**Formato**: ogni memoria è un file `.md` con frontmatter `name/description/type`. Il file `MEMORY.md` è l'indice (max 200 righe — oltre viene troncato).

---

## CLAUDE.md

File di istruzioni vincolanti, presente nella root di ogni progetto. Ha precedenza su qualsiasi comportamento di default. Non è memoria — è **configurazione del comportamento**.

Progetti con CLAUDE.md attivo:
- `~/UniCode/CLAUDE.md` — il più elaborato: regole per Diritto, SysAdmin, Security; struttura file; standard qualità output; anti-pattern

---

## Permessi (settings.local.json)

Allowlist di comandi pre-approvati (nessun prompt di conferma):
- `npm install/uninstall/list` — gestione pacchetti
- `gh api` — GitHub API
- `repomix` — packaging codebase per context
- `claude plugin *` — gestione plugin
- `Read(//tmp/**)`, `Read(//usr/bin/**)` — lettura path di sistema

---

## Manutenzione

### Quando aggiornare questo file
- [ ] Nuovo plugin installato → aggiorna sezione Plugin
- [ ] Nuova skill locale creata → aggiorna tabella `lorenzo-skills`
- [ ] Nuovo hook aggiunto → aggiunta sezione Hook
- [ ] Nuovo progetto con memoria persistente → aggiorna tabella Memoria
- [ ] Permesso aggiunto a `settings.local.json` → aggiorna sezione Permessi

### Rigenera il dump completo
```bash
repomix /home/lorenzo/.claude/ \
  --include "settings.json,settings.local.json,statusline-command.sh,hooks/**,local-marketplace/**,plugins/cache/claude-plugins-official/superpowers/**/skills/**,projects/*/memory/**,plans/*.md" \
  --output /home/lorenzo/Boot_Debian/ClaudeAnalysis/claude-setup-context.md
```

### File chiave
| File | Scopo |
|------|-------|
| `~/.claude/settings.json` | Plugin attivi, hook, status line, tema |
| `~/.claude/settings.local.json` | Permessi allowlist (non committare su repo pubbliche) |
| `~/.claude/hooks/precompact-handoff.sh` | Hook auto-handoff |
| `~/.claude/statusline-command.sh` | Script status line |
| `~/.claude/local-marketplace/plugins/lorenzo-skills/` | Skill custom |
| `~/.claude/projects/*/memory/` | Memoria persistente per progetto |
