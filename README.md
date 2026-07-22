# LOKALT — översikt & arkitektur

Det här repot ([pepestal/lokalt-docs](https://github.com/pepestal/lokalt-docs))
innehåller **enbart dokumentation** för hur mina appar hänger ihop. De individuella
apparna ligger i egna mappar bredvid denna fil men är **inte** en del av detta repo —
de ignoreras via [.gitignore](.gitignore) och versionshanteras var för sig.

Syftet är att kunna synka en gemensam karta över systemet mellan mina olika datorer.

## Arkitektur i korthet

**Syntes är navet.** Ingen app pratar direkt med en annan app — alla pratar bara med
Syntes över HTTP. En app *berättar* att saker hänt (**producent**) och/eller *lyssnar*
efter att saker hänt (**konsument**). Syntes tar emot, validerar mot strikta scheman,
sparar i en oföränderlig logg och serverar händelserna.

> Varje app äger sin egen databas. Ingen app läser någonsin direkt i en annans databas.
> All kommunikation går via Syntes API (`https://api.syntes.dev`).

Detaljerna — endpoints, scheman, hur man ansluter en app — finns i
[docs/syntes_integration.md](docs/syntes_integration.md). Se även
[docs/architecture.md](docs/architecture.md) för diagram och händelseflöden.

## Appar

| App | Mapp | Repo | Roll i navet | Beskrivning |
|-----|------|------|--------------|-------------|
| **Syntes** | `syntes/` | [syntes](https://github.com/pepestal/syntes) | **navet** | Event-buss: FastAPI + PostgreSQL. Tar emot, validerar och serverar händelser. |
| **Signal** · backend | `Signal/backend/` | [signal_backend](https://github.com/pepestal/signal_backend) | producent | FastAPI + PostgreSQL. Hämtar & processar finansdata, genererar köp-/säljsignaler. Publicerar `signal.recommendation.v1`. |
| **Signal** · frontend | `Signal/signal_frontend/` | [signal_frontend](https://github.com/pepestal/signal_frontend) | — | Next.js + TailwindCSS + Lightweight Charts. UI för Signal, pratar med Signal-backend. |
| **Todos** ("Listan") | `todos/` | [todos](https://github.com/pepestal/todos) | producent + konsument | FastAPI + React/TS + PostgreSQL. Uppgiftshanterare. Publicerar `task.created.v1` / `task.completed.v1`; konsumerar signaler → skapar granskningsuppgifter. |
| **Stronk** | `stronk/` | [stronk](https://github.com/pepestal/stronk) | producent _(planerad)_ | Mobile-first gympass-tracker (PWA, en användare). Ska publicera `workout.completed.v1`. |
| **Portal** | `portal/` | [portal](https://github.com/pepestal/portal) | — | Landing page (Vite). Tidigare `landing-page`. |

> **Signal** är en paraplymapp: dess övergripande dokumentation ligger i detta repo,
> men de två underrepona `backend/` och `signal_frontend/` versionshanteras var för sig
> och ignoreras här.

## Dokumentation

- [docs/architecture.md](docs/architecture.md) — hur apparna kommunicerar (diagram + flöden)
- [docs/syntes_integration.md](docs/syntes_integration.md) — komplett integrationsguide (kopia av `syntes/docs/INTEGRATION.md`)

---

## Arbeta på en ny dator

Så här återskapar du hela `LOKALT`-uppsättningen från grunden.

### 1. Klona dokumentationen (detta repo) till en mapp som heter `LOKALT`

```bash
git clone https://github.com/pepestal/lokalt-docs.git LOKALT
cd LOKALT
```

### 2. Klona apparna bredvid dokumentationen

Kör detta **inifrån `LOKALT`**:

```bash
git clone https://github.com/pepestal/syntes.git
git clone https://github.com/pepestal/todos.git
git clone https://github.com/pepestal/stronk.git
git clone https://github.com/pepestal/portal.git
```

### 3. Klona Signals två underrepon i en `Signal/`-mapp

```bash
mkdir Signal && cd Signal
git clone https://github.com/pepestal/signal_backend.git backend
git clone https://github.com/pepestal/signal_frontend.git signal_frontend
cd ..
```

> Signals övergripande dokumentation (`REVIEW_FRONTEND.md`, `push.md`, m.fl.) följer
> med `lokalt-docs` automatiskt — bara de två kod-underrepona behöver klonas separat.

### 4. Efter kloning: hemligheter (`.env`) per app

`.env`-filer ligger **aldrig** i git och följer alltså inte med. Skapa dem per app
från respektive `.env.example`, t.ex.:

```bash
cp syntes/.env.example        syntes/.env
cp Signal/backend/.env.example Signal/backend/.env
# ...osv. Fyll i riktiga värden (DB-lösenord, API-nycklar).
```

För att prata med Syntes behöver varje app dessutom en **API-nyckel** (`SYNTES_API_KEY`)
med rätt scope. Nyckeln fås av den som har VPS-åtkomst — se
[docs/syntes_integration.md](docs/syntes_integration.md) avsnitt 1.

### Slutresultat — mappstrukturen ska se ut så här

```
LOKALT/                       ← lokalt-docs (detta repo)
├── README.md
├── docs/
├── Signal/                   ← blandat: docs (i detta repo) + två egna repon
│   ├── REVIEW_FRONTEND.md    ← följer med lokalt-docs
│   ├── push.md               ← följer med lokalt-docs
│   ├── backend/              ← eget repo (ignoreras här)
│   └── signal_frontend/      ← eget repo (ignoreras här)
├── portal/                   ← eget repo (ignoreras här)
├── stronk/                   ← eget repo (ignoreras här)
├── syntes/                   ← eget repo (ignoreras här)
└── todos/                    ← eget repo (ignoreras här)
```

## Dagligt flöde

- **`lokalt-docs`** (uppe i `LOKALT`): `git pull` när du sätter dig, `git push` när du
  uppdaterat dokumentationen.
- **Varje app**: hantera som ett vanligt eget repo — `pull`/`commit`/`push` i respektive mapp.
