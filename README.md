# Claude Code Self-Learning System

**Svenska** | [English](README.en.md)

Ett självlärande system som transformerar Claude Code från "Execute & Forget" till "Execute & Learn".

## Översikt

Detta system gör att Claude Code kan extrahera, lagra och återanvända kunskaper från tidigare sessioner. När du löser problem tillsammans med Claude kan dessa lärdomar sparas som skills som automatiskt laddas i framtida sessioner.

**Filosofi: Compounding Knowledge** - Varje session lägger till patterns, tekniker och insikter som gör Claude smartare över tid. Kunskap förstärker sig själv - dagens learnings bygger på gårdagens, vilket skapar en ständigt växande expertis.

**Nuvarande implementation:** En enkel, fungerande lösning baserad på Claude Code's inbyggda skill-system.

## Hur det fungerar

```
Session: Du och Claude löser ett problem
              ↓
Du: /learn
              ↓
Claude sammanfattar sessionen + analyserar kodändringar
              ↓
Extraherar patterns, tekniker och lärdomar
              ↓
Sparar som skills (global eller projekt-specifik)
              ↓
Nästa session: Skills laddas automatiskt
              ↓
Claude applicerar lärdomar proaktivt
```

## /learn-kommandot

### Vad det gör

1. **Frågar om lagringsplats** (Global eller Projekt) - påverkar vad som är värt att spara
2. **Sammanfattar sessionen** - diskussioner, vad som fungerade/inte fungerade
3. **Analyserar kodändringar** - uncommitted, commits, eller båda
4. **Extraherar patterns** - tekniker och kunskap som är värdefull att återanvända
5. **Kategoriserar med taggar** - security, performance, php, etc.
6. **Skapar/uppdaterar skills** - laddas automatiskt i framtida sessioner

**OBS:** Om inga värdefulla learnings hittas sparas inget. Detta är normalt för rutinuppgifter - inte varje session producerar återanvändbar kunskap.

### Scope-alternativ

Tre källor kan analyseras:
- **`changes`** - Uncommitted ändringar (git diff)
- **`commit [hash]`** - Specifik commit (default: HEAD)
- **`session`** - Session-konversation och lärdomar

**Default:** `changes session` (om inga argument ges)

### Användning

```bash
/learn                          # Default: changes + session
/learn session                  # Endast session-lärdomar
/learn changes                  # Endast uncommitted ändringar
/learn commit                   # Endast senaste commit (HEAD)
/learn commit abc123            # Endast specifik commit
/learn changes session          # Uncommitted + session
/learn commit session           # Senaste commit + session
/learn changes commit           # Uncommitted + senaste commit
/learn changes commit session   # Alla tre källor
```

### Lagringsplats

Varje gång du kör `/learn` får du välja var lärdomar ska sparas:

- **Global** - `~/.claude/skills/learned-{tag}/` - Tillgängligt i alla projekt
- **Project** - `{projekt}/.claude/skills/learned-{tag}/` - Projektspecifikt

**Viktigt om precedens:** Om samma skill finns på båda ställen har projekt-versionen företräde - den globala ignoreras helt (ingen merge). Undvik därför att ha samma skill-namn på båda platser. Välj en plats per kategori:
- Universella patterns (security, performance) → Global
- Projektspecifik arkitektur → Project

## Installation

### Steg 1: Skapa slash command och instruktioner

Kopiera filerna till din Claude-konfiguration:

```bash
# Skapa mappar
mkdir -p ~/.claude/commands ~/.claude/docs

# Kopiera filer
cp commands/learn.md ~/.claude/commands/learn.md
cp docs/learn-instructions.md ~/.claude/docs/learn-instructions.md
```

**Varför två filer?** Slash commands laddas alltid i context. Genom att ha instruktionerna i en separat fil (`docs/`) sparar vi ~2.5k tokens som bara laddas när `/learn` körs.

### Steg 2: Konfigurera permissions

Permissions kan konfigureras globalt eller per projekt.

#### Globalt (rekommenderat)

Lägg till i `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(mkdir:~/.claude/skills/learned-*)",
      "Bash(mkdir:.claude/skills/learned-*)",
      "Read(~/.claude/docs/learn-instructions.md)",
      "Read(~/.claude/skills/learned-*)",
      "Edit(~/.claude/skills/learned-*)",
      "Write(~/.claude/skills/learned-*)",
      "Read(.claude/skills/learned-*)",
      "Edit(.claude/skills/learned-*)",
      "Write(.claude/skills/learned-*)"
    ]
  }
}
```

#### Per projekt

Lägg till i projektets `.claude/settings.local.json` med samma innehåll som ovan.

**Klart!** Nu kan du använda `/learn` i dina Claude Code-sessioner.

### Permissions förklaring

| Permission | Syfte |
|------------|-------|
| `Bash(mkdir:~/.claude/skills/learned-*)` | Skapa globala skill-mappar |
| `Bash(mkdir:.claude/skills/learned-*)` | Skapa projektspecifika skill-mappar |
| `Read(~/.claude/docs/learn-instructions.md)` | Läsa instruktionsfilen |
| `Read/Edit/Write(~/.claude/skills/learned-*)` | Hantera globala skills |
| `Read/Edit/Write(.claude/skills/learned-*)` | Hantera projektspecifika skills |

**Notera:** Wildcards (`learned-*`) matchar alla learned-skills. Tilde (`~`) expanderas till home-katalogen.

## Nyckelfeatures

### Kvalitetsfilter och selectivity

Systemet balanserar att bygga kunskap över tid med att undvika informationsbrus:

**Sparar värdefulla learnings:**
- Patterns/tekniker upptäckta genom trial-and-error
- Anti-patterns från verkliga misslyckanden (förhindrar upprepade misstag)
- Icke-självklara lösningar på vanliga problem
- Domänspecifika best practices från erfarenhet
- Insikter om "varför X fungerar bättre än Y i kontext Z"

**Sparar INTE:**
- Rå implementation details utan mönster
- Standard framework-användning som finns i dokumentationen
- Kronologiska loggar utan insikter om VARFÖR

Session-historik innehåller både lyckade och misslyckade ansatser. Systemet separerar vad som fungerade (i final kod) från vad som misslyckades (prövat men övergivet), och extraherar de lärdomar som gör Claude smartare nästa gång.

### Intelligent uppdatering

Skills hålls uppdaterade över tid:

- **Nya ämnen** skapas automatiskt som nya entries
- **Befintliga ämnen** uppdateras och berikas med nya insikter
- **Duplicering undviks** genom att merga liknande learnings
- **Motsägelser löses** genom att integrera och uppdatera befintlig kunskap

När du kör `/learn` analyseras om liknande kunskap redan finns. Istället för att skapa duplicerade entries uppdateras befintliga med ny information, vilket håller kunskapsbasen ren och aktuell.

### Lazy loading

Skills organiseras för optimal context-hantering:

- **SKILL.md** (Lightweight index) - ~50 tokens per learning, laddas vid session start
- **reference.md** (Full details) - ~1500 tokens per learning, läses on-demand

Detta skalerar till 100+ learnings utan context-explosion:
- 100 learnings × 50 tokens = ~5,000 tokens vid session start
- Full details läses endast när de behövs

### Proaktiv användning

```
Du: "Lägg till ett kommentarsformulär"

Claude: *Ser i learned-security skill*
        "XSS Protection - Use when: handling user input as HTML"
        *Läser reference.md för detaljer*

        "Jag implementerar med DOMPurify XSS-skydd baserat på
         tidigare lärdomar..."
```

## Exempel

### Efter buggfix

```bash
# Fixa XSS-sårbarhet, committa
git commit -m "Fix XSS in comment form"

# Extrahera learning från commit
/learn commit

# → Skapar learned-security skill med XSS-pattern
```

### Efter problemlösning

```bash
# Ha en session där ni:
# - Prövade JWT först (fungerade inte)
# - Bytte till OAuth2
# - Debuggade token refresh

# Extrahera från både kod och session
/learn

# → Sparar både OAuth2-pattern OCH varför JWT inte fungerade
```

### Retroaktiv learning

```bash
# Extrahera från äldre commit
/learn commit abc123

# Eller: kombinera gammal commit med nuvarande session
/learn commit abc123 session
```

## Filorganisation

```
~/.claude/
├── commands/
│   └── learn.md              # Minimal slash command (~50 tokens)
├── docs/
│   └── learn-instructions.md # Fulla instruktioner (~2.5k tokens, on-demand)
└── skills/
    ├── learned-security/
    │   ├── SKILL.md          # Lightweight index
    │   └── reference.md      # Full details
    ├── learned-php/
    │   ├── SKILL.md
    │   └── reference.md
    └── learned-performance/
        ├── SKILL.md
        └── reference.md
```

### Context-optimering

Slash commands i `~/.claude/commands/` laddas **alltid** i context vid sessionsstart. Genom att separera instruktioner till `docs/` sparar vi tokens:

| Fil | Storlek | Laddas |
|-----|---------|--------|
| `commands/learn.md` | ~50 tokens | Alltid |
| `docs/learn-instructions.md` | ~2.5k tokens | Vid `/learn` |

**Besparing:** ~2.5k tokens per session där `/learn` inte används.

## Fördelar

- **Minimal setup** - En slash command + instruktionsfil
- **Inga externa dependencies** - Använder enbart Claude Code
- **Rik kunskapsextraktion** - Session-kontext + kodändringar
- **Flexibel lagring** - Global eller projekt-specifik
- **Skalbar** - Lazy loading hanterar 100+ learnings
- **Proaktiv** - Claude applicerar lärdomar automatiskt

## Troubleshooting

### Slash command finns inte

```bash
# Kontrollera att båda filerna finns
ls ~/.claude/commands/learn.md
ls ~/.claude/docs/learn-instructions.md

# Om inte, kopiera från detta repo
```

### Agent skapar inte skills

Vanligaste orsaken: Saknade permissions.

```bash
# Kontrollera permissions (globalt eller per projekt)
cat ~/.claude/settings.json
cat .claude/settings.local.json

# Se till att följande finns i allow-listan:
# - Bash(mkdir:*)
# - Read(~/.claude/docs/learn-instructions.md)
# - Write(~/.claude/skills/learned-*)
```

### Skills laddas inte

```bash
# Kontrollera att skills finns
ls -la ~/.claude/skills/*/SKILL.md

# Kontrollera SKILL.md format (måste ha frontmatter)
cat ~/.claude/skills/learned-php/SKILL.md

# Starta om Claude Code
```

### Git diff är tom

```bash
git status  # Kontrollera att det finns ändringar

# Alternativ:
/learn commit        # Använd senaste commit istället
/learn session       # Analysera bara session
```

## Vidareutveckling

För användare med 50+ learnings som behöver mer avancerad funktionalitet finns möjlighet att bygga ut till ett fullständigt system med:

- **Semantic search** - Vector embeddings med FAISS för smart sökning
- **Automatisk trigger** - SessionEnd hook istället för manuell `/learn`
- **Usage tracking** - Spåra hur ofta patterns används
- **MCP Server** - Dedikerade verktyg för kunskapssökning

Se [`ADVANCED_IMPLEMENTATION.md`](ADVANCED_IMPLEMENTATION.md) för teknisk specifikation och implementationsguide.

## License

MIT

## Författare

Johan Sölve (idéer & synpunkter)
Claude Code (själva jobbet)