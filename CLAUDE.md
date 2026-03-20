# CLAUDE.md — Toekomstscenariodatabase v3

Dit bestand geeft Claude Code de projectcontext, architectuurkeuzes en conventies.
Lees dit volledig voordat je iets bouwt of aanpast.

---

## Projectdoel

Een persoonlijk scenario-analyse instrument dat toekomstscenario's voor de periode
2026–2035 bijhoudt op basis van bellwether-data, drivers en periodieke AI-agent analyses.

### Dynamisch karakter — BELANGRIJK

Het systeem start met een beginset, maar is ontworpen om te groeien én te krimpen:

- **Scenario's**: beginset is 3, maar kunnen worden toegevoegd (ook wildcards/black swans)
  of gedeactiveerd als ze irrelevant worden. Geen hard maximum.
- **Drivers**: beginset is 15, bandbreedte **10–25 actief**. Voeg toe als nieuwe structurele
  krachten relevant worden; deactiveer als ze irrelevant worden.
- **Bellwethers**: beginset is 5, bandbreedte **5–15 actief**. Voeg toe als nieuwe meetbare
  indicatoren beschikbaar komen; deactiveer als datakwaliteit of relevantie wegvalt.

Gebruik altijd `is_active` flags — verwijder nooit historische data.

### De beginset scenario's

| Naam | Startkans | Type |
|------|-----------|------|
| Turbulente Transitie | 52% | main |
| Gespleten Economie | 31% | main |
| Digitale Overvloed, Fysieke Schaarste | 17% | main |

**Let op:** Kansen tellen bewust NIET per se op tot 100%. Er is altijd ruimte voor
wildcards en onvoorziene scenario's. Sla kansen op als decimaal (0.52) maar toon als
percentage (52%).

Elk scenario heeft een `confidence` veld naast `probability` — "52% kans, maar ik ben er
niet zeker van" is iets anders dan "52% kans, stevig onderbouwd."

### Wildcard scenario's

Naast de hoofdscenario's kunnen wildcard-scenario's worden toegevoegd voor black swans
(lage kans, hoge impact). Scenario type `wildcard` is daarvoor. Voorbeeld: "Onverwachte
AGI-doorbraak", "Grote geopolitieke crisis". Wildcards worden apart weergegeven op het
dashboard.

---

## Tech Stack

| Laag | Keuze | Reden |
|------|-------|-------|
| Frontend | React + Vite + TypeScript | Bewezen in v1/v2 |
| Styling | Tailwind CSS | Utility-first, snel |
| Data fetching | TanStack Query (React Query) | Server state management |
| Grafieken | Recharts | Lightweight, React-native |
| Backend | Node.js + Express | Simpel, bewezen |
| Validatie | Zod | Schema validatie op API-grens |
| ORM | Drizzle ORM | Type-safe, migratie-based |
| Database | Neon PostgreSQL (serverless) | Bewezen in v1/v2 |
| AI Agent | Anthropic SDK (claude-sonnet-4-20250514) | Voor periodieke analyses |
| MCP Server | @modelcontextprotocol/sdk | Voor agent-integratie |

**Gebruik altijd deze stack. Introduceer geen nieuwe frameworks zonder expliciete instructie.**

---

## Databaseschema — Kernentiteiten

Dit is een volledig nieuw schema (geen migratie van v1/v2).

### Primaire tabellen

```
scenarios                  — Scenario's (beginset 3 + wildcards, kan groeien)
scenario_snapshots         — Tijdreeks: kans + confidence per scenario per datum

drivers                    — Structurele krachten (beginset 15, kan groeien/krimpen)
driver_scenario_impacts    — Hoe elke driver elk scenario beïnvloedt
driver_snapshots           — Tijdreeks: assessment van elke driver per datum

bellwethers                — Meetbare indicatoren (beginset 5, kan groeien/krimpen)
bellwether_thresholds      — Drempelwaarden: bij welke waarde → welk scenario-signaal
measurements               — Tijdreeks: datapunten per bellwether

sources                    — Databronnen (BLS, Metaculus, Polymarket, etc.)
tags                       — Flexibel labelsysteem voor extensies
analysis_runs              — Audit trail voor agent-analyses
```

### Kritieke ontwerpregels

1. **Tijdreeksen zijn immutable** — `scenario_snapshots`, `measurements` en
   `driver_snapshots` worden nooit geüpdatet, alleen nieuwe rijen toegevoegd.

2. **Alle PKs zijn UUIDs** — gebruik `gen_random_uuid()` als default.

3. **Altijd FK constraints op DB-niveau** — dit was een fout in v1/v2.

4. **`recorded_at` vs `created_at`** — `recorded_at` is de werkelijke meetdatum
   (kan in het verleden liggen voor historische seed data). `created_at` is insertdatum.

5. **`is_active` op scenarios, drivers en bellwethers** — nooit verwijderen,
   alleen deactiveren. Historische data blijft altijd intact.

6. **Confidence naast probability** — `scenario_snapshots` heeft beide velden.

7. **Kansen tellen niet per se tot 100%** — wildcards kunnen extra probabiliteitsmassa
   toevoegen. Nooit normaliseren naar 100%.

8. **Geen child developments** — v1/v2 hiërarchie vervangen door `driver_scenario_impacts`
   (~45 rijen: 15 drivers × 3 scenario's).

### Enums (gebruik consistent)

```typescript
scenario_type: 'main' | 'wildcard' | 'archived'

impact_direction: 'supports' | 'opposes' | 'neutral' | 'complex'
impact_strength: 'weak' | 'moderate' | 'strong' | 'dominant'

speed: 'slow' | 'moderate' | 'fast' | 'exponential'
strength: 'weak' | 'moderate' | 'strong' | 'dominant'
uncertainty: 'low' | 'medium' | 'high' | 'very_high'
confidence: 'low' | 'medium' | 'high' | 'very_high'

signal_direction: 'strongly_toward' | 'toward' | 'neutral' | 'away' | 'strongly_away'

update_frequency: 'daily' | 'weekly' | 'monthly' | 'quarterly' | 'annual' | 'event_driven'

run_type: 'bellwether_update' | 'scenario_review' | 'monte_carlo' | 'agent_analysis' | 'full_refresh'
run_status: 'running' | 'completed' | 'failed'
```

---

## Beginset: 15 Drivers (bandbreedte 10–25)

Drivers zijn structurele krachten — geen bellwethers (meetpunten).
**Bandbreedte: 10–25 actief.** Onder 10 verlies je dekking; boven 25 worden agent-analyses
token-onhoudbaar. Streef naar 12–18 als werkzaam optimum.
"AI Capabilities" en "Labor Share" zijn bellwethers, geen drivers.
"Narratieve macht" en "Systeemkwetsbaarheid" zijn kenmerken van het analysemodel,
geen losse drivers.

| # | Driver | Domein |
|---|--------|--------|
| 1 | Geopolitieke machtsdynamiek | Geopolitiek |
| 2 | AI Governance & Regulering | Governance |
| 3 | Power Concentration vs. Democratization | Economie |
| 4 | AI Labor Market Impact | Economie |
| 5 | Macro-economische dynamiek | Economie |
| 6 | Beleidscyclus-volatiliteit | Governance |
| 7 | Robotica & fysieke automatisering | Technologie |
| 8 | Energie & grondstoffendynamiek | Klimaat/Econ |
| 9 | AI in Healthcare & Biotech | Technologie |
| 10 | Public Trust & Social Cohesion | Sociaal |
| 11 | AI Education & Workforce Adaptation | Sociaal |
| 12 | Surveillance & Privacy | Sociaal |
| 13 | AI Misinformation & Deepfakes | Sociaal |
| 14 | Demografische druk | Sociaal |
| 15 | Klimaatschokken | Klimaat |

**Relatie drivers ↔ bellwethers:** Niet 1-op-1. Één driver kan meerdere bellwethers
triggeren. Één bellwether kan door meerdere drivers bewogen worden (de "double whammy").
Leg primaire driver vast in `bellwether.driver_id`, secundaire via tags.

---

## Beginset: 5 Bellwethers (bandbreedte 5–15)

**Bandbreedte: 5–15 actief.** Onder 5 onvoldoende signaalcoverage; boven 15 ruis en
onderhoudslast. Voeg toe als nieuwe meetbare indicatoren beschikbaar komen; deactiveer
als datakwaliteit of relevantie wegvalt.

| # | Naam | Bron | Frequentie |
|---|------|------|------------|
| 1 | ARC-AGI-2 Benchmark Score | ARC Prize Foundation | event_driven |
| 2 | Tesla Optimus Productieaantallen | Tesla earnings | quarterly |
| 3 | US U-6 Werkloosheid % | BLS | monthly |
| 4 | US Labor Share Nationaal Inkomen % | BLS/FRED | quarterly |
| 5 | Prediction Market AGI Mediaan Datum | Metaculus | monthly |

---

## Architectuur

```
/
├── client/                 # React frontend
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── hooks/          # TanStack Query hooks
│   │   └── lib/
├── server/
│   ├── routes/             # Thin — alleen request/response
│   ├── services/           # Dik — alle business logic hier
│   ├── db/
│   │   ├── schema.ts       # Drizzle schema (single source of truth)
│   │   └── migrations/
│   ├── agents/             # AI agent logica (4 taken)
│   └── mcp/                # MCP server definitie en tools
├── shared/
│   └── types.ts
└── CLAUDE.md
```

---

## API Endpoints

```
# Scenarios
GET    /api/scenarios
GET    /api/scenarios/:slug
GET    /api/scenarios/:slug/snapshots    → Tijdreeks voor grafieken
POST   /api/scenarios                    → Nieuw scenario (ook wildcards)
PATCH  /api/scenarios/:slug              → incl. is_active
POST   /api/scenarios/:slug/snapshots    → Nieuwe kans + confidence

# Drivers
GET    /api/drivers
GET    /api/drivers/:slug
POST   /api/drivers
PATCH  /api/drivers/:slug                → incl. is_active
POST   /api/drivers/:slug/snapshots

# Bellwethers
GET    /api/bellwethers
GET    /api/bellwethers/:slug
POST   /api/bellwethers
PATCH  /api/bellwethers/:slug            → incl. is_active
POST   /api/bellwethers/:slug/measurements

# Analysis
POST   /api/analysis/run
GET    /api/analysis/runs
GET    /api/analysis/runs/:id

# Dashboard
GET    /api/dashboard                    → Huidige stand + recente signalen
GET    /api/dashboard/timeline           → Tijdreeks data voor grafieken
```

---

## MCP Server Tools

```typescript
// Read
list_scenarios, get_scenario_detail
list_drivers
list_bellwethers, get_bellwether_history
get_dashboard

// Write
add_measurement
add_scenario_snapshot          // met confidence
add_driver_snapshot
create_analysis_run
add_scenario                   // ook wildcards
add_driver
add_bellwether
```

---

## AI Agent Architectuur (4 taken)

Token-efficiëntie is cruciaal — geef de agent alleen deltas, niet de volledige database.

1. **Bellwether Monitor** — zoekt nieuwe data per bellwether via web search, slaat op
   als `measurement`. Draait per bellwether onafhankelijk.

2. **Signal Assessor** — vergelijkt nieuwe metingen met thresholds, genereert signalen
   per scenario. Input: alleen nieuwe metingen (delta).

3. **Scenario Updater** — op basis van signalen: stelt nieuwe `scenario_snapshot` voor
   met bijgestelde kans + confidence + redenering. **Stelt voor, pusht niet automatisch.**

4. **Driver Reviewer** — periodieke herbeoordeling van driver speed/strength/uncertainty.
   Draait minder frequent (maandelijks vs. wekelijks voor bellwethers).

Model: `claude-sonnet-4-20250514` | Max tokens: 4000 per run

---

## Frontend Dashboard

### Styling — clean separation voor toekomstige restyling

De frontend is opgezet zodat een ander model (of persoon) later de styling kan overnemen
zonder logica aan te raken. Dit vereist strikte scheiding:

- **Alleen Tailwind utility classes** voor styling — geen inline `style={}` props,
  geen CSS-in-JS, geen styled-components
- **Logica en presentatie gescheiden** — elk component heeft een duidelijke grens:
  data fetching/state in hooks, rendering in het component, styling alleen via classnames
- **Geen hardcoded kleuren buiten de kleurconstanten** — definieer scenario-kleuren
  als constanten in `lib/constants.ts`, gebruik die overal

```typescript
// lib/constants.ts — alle visuele constanten op één plek
export const SCENARIO_COLORS = {
  'turbulente-transitie': '#3b82f6',
  'gespleten-economie': '#f59e0b',
  'digitale-overvloed': '#10b981',
  wildcard: '#8b5cf6',
} as const;
```

Een ander model kan later alle Tailwind classes vervangen of een design system
introduceren zonder de logica of datastructuren aan te raken.

**Scenario kleuren (gebruik consistent):**
- Turbulente Transitie: `#3b82f6` (blauw)
- Gespleten Economie: `#f59e0b` (amber)
- Digitale Overvloed: `#10b981` (groen)
- Wildcards: `#8b5cf6` (paars)

**Hoofdscherm:**
- Scenario-kaarten (main groot, wildcards apart klein)
- Kansverloop-grafiek: lijn per scenario, confidence als foutmarge of kleurintensiteit
- Kansen tellen NIET op tot 100% op het scherm — dit is correct gedrag

**Overige schermen:** Bellwethers (tijdreeks + thresholdzones), Drivers
(impacts + tijdreeks), Analyses (audit trail + trigger-knop).

---

## Seed Data (bij initialisatie)

### Scenario snapshots (3 historische meetpunten)

| Datum | Turbulente Transitie | Gespleten Economie | Digitale Overvloed |
|-------|---------------------|-------------------|-------------------|
| 2025-12-01 | 55% | 30% | 15% |
| 2026-02-07 | 42% | 40% | 18% |
| 2026-03-01 | 52% | 31% | 17% |

*(Namen in dec 2025: Gematigde Groei / Arbeidsmarkt Disruptie / Hyperabundance —
zelfde scenario's, hernoemd in feb/mrt 2026)*

### Bellwether meetwaarden (historisch)

| Bellwether | Waarde | Datum |
|-----------|--------|-------|
| ARC-AGI-2 | 54% | 2026-01-15 |
| ARC-AGI-2 | 95.1% @ $8.71/taak | 2026-02-01 |
| Optimus units | ~300 | 2025-07-01 |
| U-6 | 8.7% | 2025-11-01 |
| U-6 | 8.4% | 2025-12-01 |
| U-6 | 8.0% | 2026-01-31 |
| Labor share | 53.8% | 2025-09-30 |
| AGI mediaan | 2027-11-30 | 2026-01-15 |

---

## Extensiepaden (via tags, geen schema-wijziging)

- **Prediction markets**: tag `domain:prediction_market`, `platform:metaculus`
- **Stocks**: tag `domain:stock`, `ticker:NVDA`
- **Monte Carlo**: `analysis_runs` type `monte_carlo`, resultaten als snapshots
- **Acties**: optionele tabel `actions` later, gekoppeld aan scenario's/bellwethers

---

## Wat je NIET doet

- Geen migratie van v1/v2 data
- Geen hard-coded max op scenario's, drivers of bellwethers
- Geen auth/multi-user tenzij expliciet gevraagd
- Geen directe DB writes in route handlers
- Geen `.env` committen — gebruik `.env.example`
- Nooit kansen automatisch normaliseren naar 100%
- Nooit scenario-kansen aanpassen zonder gebruikersbevestiging

---

## Omgevingsvariabelen

```bash
DATABASE_URL=             # Neon PostgreSQL connection string
ANTHROPIC_API_KEY=        # Voor AI agent analyses
PORT=3000
```

### Replit-compatibiliteit

De Express server moet als volgt opstarten om op Replit (en andere cloud providers) te werken:

```typescript
const PORT = process.env.PORT || 3000;
app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
});
```

`'0.0.0.0'` is verplicht — zonder dit is de server niet bereikbaar buiten de container.
`PORT` komt altijd uit de omgevingsvariabele; gebruik nooit een hardcoded poort.
Neon PostgreSQL werkt serverless en heeft geen lokale Postgres-installatie nodig — dit
maakt het platform-onafhankelijk (Replit, Railway, Render, Vercel, etc.).

---

## Implementatievolgorde

1. Drizzle schema + migraties (`is_active`, `confidence`, `scenario_type` overal)
2. Seed script (15 drivers, 3 scenario's, 5 bellwethers, historische snapshots)
3. REST API endpoints
4. MCP server (read + write tools)
5. Dashboard frontend: scenario-kaarten + kansverloop grafiek
6. Bellwether en driver schermen
7. AI agent (4 taken)
8. Analyses/audit trail scherm
