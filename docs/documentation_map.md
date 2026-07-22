# Dokumentationskarta

**Var finns dokumentationen för varje app?** Denna karta pekar ut var varje apps
projekt-specifika dokumentation ligger idag, och hur långt appen kommit mot den
gemensamma [dokumentationsstandarden](documentation_standard.md).

> Varje app versionshanteras i sitt eget repo (se [`../README.md`](../README.md)).
> Länkarna nedan är dels lokala sökvägar (om du klonat apparna bredvid detta repo),
> dels GitHub-länkar till sanningskällan.

---

## Snabböversikt

| App | Repo | Doc-plats | Ingång idag | Mot standard? |
|-----|------|-----------|-------------|---------------|
| **Syntes** | [syntes](https://github.com/pepestal/syntes) | `syntes/docs/` | `docs/README.md` | 🟡 nära |
| **Todos** | [todos](https://github.com/pepestal/todos) | `todos/docs/` + rot | rot-`README.md` | 🟡 nära |
| **Stronk** | [stronk](https://github.com/pepestal/stronk) | `stronk/docs/` + rot | rot-`README.md` | ✅ migrerad |
| **Signal · backend** | [signal_backend](https://github.com/pepestal/signal_backend) | `Signal/backend/docs/` | rot-`README.md` | ✅ migrerad |
| **Signal · frontend** | [signal_frontend](https://github.com/pepestal/signal_frontend) | `Signal/signal_frontend/docs/` | rot-`README.md` | ✅ migrerad |
| **Signal** (paraply) | *(detta repo)* | `Signal/` (rot) | `.agents/AGENTS.md` (redirect) | ✅ hub klar |
| **Portal** | [portal](https://github.com/pepestal/portal) | — | — | 🔴 saknar helt |

✅ migrerad till standarden · 🟢 nästan i mål · 🟡 mindre justeringar · 🟠 tydliga avvikelser · 🔴 saknar struktur

---

## Per app — nuläge och avvikelser

### Syntes — `syntes/docs/`
Sanningskällan för **Syntes-integrationen** (`docs/INTEGRATION.md`) — alla andra
appar pekar hit.

| Fil | Roll | Standardnamn |
|-----|------|--------------|
| `docs/README.md` ("Den Stora Bibeln") | Ingång/arkitektur | flytta kärnan till rot-`README.md`; gör `docs/README.md` till index |
| `docs/INTEGRATION.md` | Integrationskälla | ✅ redan rätt |
| `docs/DEPLOY.md` | Drift | ✅ `DEPLOY.md` |
| `docs/Changelog.md` | Ändringslogg | → `CHANGELOG.md` (case) |
| `docs/NULAGE.md` | Nuläge | → `STATUS.md` |
| `docs/syntes-implementeringsplan.md` | Plan | → `ROADMAP.md` (eller `IMPLEMENTATION.md` om den lever kvar som fasplan) |
| `README.md` (rot) | Ingång | behåll som ingång |

**Avvikelser:** ingången ligger i `docs/README.md` snarare än rot; `Changelog.md`
fel case; `NULAGE`/implementeringsplan följer inte fasta namn.

### Todos — `todos/docs/` + rot
| Fil | Roll | Standardnamn |
|-----|------|--------------|
| `README.md` (rot) | Teknisk bibel | ✅ ingång |
| `projektplan.md` (rot) | Vision, principer, datamodell, roadmap, beslutslogg | → dela upp: `docs/ARCHITECTURE.md` + `docs/ROADMAP.md`, eller flytta till `docs/` som `PROJEKTPLAN.md` |
| `docs/README.md` | Index | ✅ index (tunt — bra) |
| `docs/changelog.md` | Ändringslogg | → `CHANGELOG.md` (case) |
| `CLAUDE.md` (rot) | AI-regler | ✅ rätt plats |

**Avvikelser:** `changelog.md` fel case; ingen `STATUS.md`; `projektplan.md` bär
flera roller (roadmap + arkitektur + beslutslogg) i en fil i roten.

### Stronk — `stronk/docs/` + rot ✅ migrerad
Döpt/flyttad till standarden; alla korsreferenser (README, docs, `CLAUDE.md`)
uppdaterade i samma pass, verifierat 0 kvarvarande gamla namn.

| Nu | Var (gammalt namn/plats) |
|-----|--------------------------|
| `README.md` (rot, bibeln) | ← `docs/README.md` |
| `docs/README.md` (index) | **ny** |
| `docs/STATUS.md` | ← `docs/NULAGE.md` |
| `docs/CHANGELOG.md` | ← `docs/Changelog.md` (case) |
| `docs/INTEGRATION.md` | ← `docs/integration_syntes.md` |
| `CLAUDE.md` (rot) | ← `.claude/CLAUDE.md` |
| `docs/IMPLEMENTATION.md`, `docs/ROADMAP.md` | oförändrade (redan rätt) |

`INTEGRATION.md` bantades samtidigt från en inklistrad kopia av hela Syntes-guiden
till en pekare mot sanningskällan (`syntes/docs/INTEGRATION.md`) + Stronks eget
kontrakt — en innehållslig städning utöver den mekaniska namnmigreringen.

### Signal · backend — `Signal/backend/docs/` ✅ migrerad
Döpt till standarden; alla referenser (docs, `.claude/`+`.agents/`-skills,
`.py`-kommentarer) uppdaterade, verifierat 0 kvarvarande gamla namn.

| Nu | Var (gammalt namn) |
|-----|--------------------|
| `docs/ARCHITECTURE.md` | ← `1_Arkitektur.md` |
| `docs/API.md` | ← `2_API_Endpoints.md` |
| `docs/DATABASE.md` | ← `3_Database_Schema.md` |
| `docs/OVERVIEW.md` | ← `4_Översikt.md` |
| `docs/ENGINES.md` | ← `README_Motor.md` |
| `docs/engines/MOTOR_ROADMAP.md` | ← `Motor_Dev&Roadmap.md` |
| `docs/{README,STATUS,CHANGELOG}.md` | **nya** (index/nuläge/logg) |

Rättade även brutna länkar i rot-`README.md` (`./backend/docs/` → `./docs/`).
`REVIEW_BACKEND.md` + `docs/*.sql` (operativ DDL) lämnades där de låg.

### Signal · frontend — `Signal/signal_frontend/docs/` ✅ migrerad
`src/docs/` → `docs/` (repo-rot); numrerade filer omdöpta; alla referenser
(docs, `.claude/`+`.agents/`-skills, `.tsx`-kommentarer) uppdaterade, 0 kvar.

| Nu | Var (gammalt) |
|-----|---------------|
| `docs/OVERVIEW.md` | ← `src/docs/1_Översikt.md` |
| `docs/STATE_AND_DATA.md` | ← `src/docs/3_State_and_Data.md` |
| `docs/ROUTING_AND_COMPONENTS.md` | ← `src/docs/4_Routing_and_Components.md` |
| `docs/ENGINES_AND_SIGNALS.md` | ← `src/docs/5_Engines_and_Signals.md` |
| `docs/CHANGELOG.md` | ← `src/docs/Changelog.md` |
| `docs/UI_*.md` (5 st) | ← `src/docs/UI_*.md` (flyttade + konsoliderade) |
| `docs/{README,STATUS}.md` | **nya** |

> **UI-doc-konsolidering (gjord):** det arkiverade `UI_RULES.md` (746 rader) togs
> bort; dess unika, giltiga innehåll — **responsiv/mobil-arkitekturen** +
> **premium-finesser-katalogen** — flyttades in i `UI_PATTERNS.md §6–7`. De 5
> avsiktliga UI-dokumenten (LAW/FUSION/PATTERNS/OVERVIEW/GUIDE) behölls (en medveten
> designhierarki). Alla `UI_RULES`-referenser (docs, `.agents/.claude`-skills,
> `.tsx`/`.ts`-kommentarer) + `md §`-ankare repekade; daterad historik lämnad orörd.

### Signal (paraply) — `Signal/` i detta repo ✅ hub klar
Redirect-hubben ligger i `.agents/AGENTS.md` (uppdaterad: pekar in i båda repons
`docs/` enligt standarden). Per din egen konvention ska paraplymappen förbli
minimal — därför skapades **ingen** `Signal/docs/`.

**Städat (klart):**
- `push.md` → flyttad till `signal_backend/docs/STATISTICS.md` (var en backend-spec
  för Statistik-sidan, Update 1.4). Self-referenser uppdaterade.
- `REVIEW_BACKEND.md`, `REVIEW_FRONTEND.md` och `docs/CREATE_*.sql` — raderade
  (avsiktlig städning). Mina nya dok pekar inte längre på dem.
- 🔐 Inloggningsuppgifter flyttade ut ur `prompt.md`/`short_prompt.md` till
  `LOKALT/password.md` (**gitignorad**); prompt-filerna pekar dit istället.
  Lösenordet purgas ur git-historiken (commit `b470cad`) som sista steg.

### Portal — saknar dokumentation helt 🔴
Nystartat projekt (tidigare `landing-page`, Vite). **Ingen** `README.md`, inget
`docs/`. **Förstahandskandidat** för att scaffoldas ur
[`templates/`](templates/) — se dokumentationsstandarden §7.

---

## Nästa steg (migrering, per repo — separata godkända pass)

Migreringen sker **ett repo i taget**. För varje app: byt filnamn/plats enligt
tabellerna ovan, och jaga + rätta alla korsreferenser (se
[standard §5](documentation_standard.md#5-korsreferenser-viktigast-vid-migrering))
i samma commit. Föreslagen ordning — enklast först:

1. ~~**Signal backend**~~ — ✅ klar.
2. ~~**Signal frontend**~~ — ✅ klar.
3. ~~**Signal paraply**~~ — ✅ redirect-hub klar (loose-filer + credentials flaggade ovan).
4. **Portal** — scaffolda ur mallen (inget att jaga, ren nystart).
5. ~~**Stronk**~~ — ✅ klar (bibeln → rot, `docs/README.md`-index, `STATUS`/`CHANGELOG`/`INTEGRATION` omdöpta, `CLAUDE.md` → rot).
6. **Todos** — case + dela upp `projektplan.md`.
7. **Syntes** — flytta ingång till rot, döp om `NULAGE`/plan.
