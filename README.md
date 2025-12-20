# Claude Code Self-Learning System

**Svenska** | [English](README.en.md)

Ett självlärande system som transformerar Claude Code från "Execute & Forget" till "Execute & Learn".

## Översikt

Detta system gör att Claude Code kan extrahera, lagra och återanvända kunskaper från tidigare sessioner. När du löser problem tillsammans med Claude kan dessa lärdomar sparas som skills som automatiskt laddas i framtida sessioner.

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

1. Sammanfattar nuvarande session (diskussioner, vad som fungerade/inte fungerade)
2. Analyserar kodändringar (uncommitted, commits, eller båda)
3. Extraherar patterns, tekniker och kunskap
4. Kategoriserar med taggar (security, performance, php, etc.)
5. Frågar om lagringsplats (Global eller Projekt)
6. Skapar/uppdaterar skills som laddas automatiskt i framtida sessioner

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

### Steg 1: Skapa slash command

Kopiera `commands/learn.md` till `~/.claude/commands/learn.md`

```bash
mkdir -p ~/.claude/commands
cp commands/learn.md ~/.claude/commands/learn.md
```

### Steg 2: Konfigurera permissions (per projekt)

Lägg till i projektets `.claude/settings.local.json`:

```json
{
  "permissions": {
    "allow": [
      "Read(~/.claude/skills/**)",
      "Edit(~/.claude/skills/**)",
      "Write(~/.claude/skills/**)"
    ]
  }
}
```

**Klart!** Nu kan du använda `/learn` i dina Claude Code-sessioner.

## Nyckelfeatures

### Context poisoning prevention

Session-historik innehåller både lyckade och misslyckade ansatser. Systemet separerar:

- **What Worked** - I final kod, inga användarkorrektioner
- **What Didn't Work** - Prövat men övergivet, användaren korrigerade
- **Key Insight** - Varför en approach vann över andra

Misslyckade ansatser är värdefulla - de förhindrar att samma misstag görs igen.

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
~/.claude/skills/
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

## Fördelar

- **Ingen setup-komplexitet** - Bara en slash command-fil
- **Inga externa dependencies** - Använder enbart Claude Code
- **Rik kunskapsextraktion** - Session-kontext + kodändringar
- **Flexibel lagring** - Global eller projekt-specifik
- **Skalbar** - Lazy loading hanterar 100+ learnings
- **Proaktiv** - Claude applicerar lärdomar automatiskt

## Troubleshooting

### Slash command finns inte

```bash
ls ~/.claude/commands/learn.md
# Om inte, kopiera från commands/learn.md
```

### Agent skapar inte skills

Vanligaste orsaken: Missing permissions i projektets settings.

```bash
# Kontrollera projektets settings
cat .claude/settings.local.json

# Se till att Write(~/.claude/skills/**) finns i allow-listan
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

Se `ADVANCED_IMPLEMENTATION.md` för teknisk specifikation och implementationsguide.

## License

MIT

## Författare

Johan Sölve (idéer & synpunkter)
Claude Code (själva jobbet)