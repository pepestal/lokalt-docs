Denna fil är en direkt kopia av syntes/docs/INTEGRATION.md och bör hållas synkroniserad med denna för att undvika dokumentationsdrift.

# Ansluta en app till Syntes — integrationsguide

> **Denna fil är skriven för att klistras in till en AI-agent (eller läsas av en
> utvecklare) som arbetar i en ANNAN app — Signal, Todos, Zens, Stronk eller en
> framtida app — och ska få den att kommunicera med Syntes.** Den är självständig:
> du behöver inte se Syntes källkod för att följa den.

**Vad är Syntes?** Navet som alla appar i ekosystemet pratar med. Ingen app känner till
någon annan app — de känner bara till Syntes och dess regler. En app *berättar* att saker
hänt (producent) och/eller *lyssnar* efter att saker hänt (konsument). Syntes tar emot,
validerar, lagrar (oföränderlig logg) och serverar dessa händelser.

**Grundregel:** All kommunikation går via HTTP-anrop till Syntes API. Ingen app läser
någonsin direkt i en annan apps databas. Varje app äger sin egen databas.

---

## Innehåll

1. Anslutningsdetaljer (URL, auth)
2. De tre endpointsen
3. Vad som är implementerat i dagsläget (2026-07-22)
4. Händelsekontrakten (scheman) — reglerna
5. Kommunikationsmönster — och vad som krävs på BÅDA håll
   - 5a. Bli producent
   - 5b. Bli processande konsument (polling)
   - 5c. Bli display-konsument (snapshot)
   - 5d. Commands: be en annan app göra något
6. Referenskod (klistra in i din app)
7. Lägga till en ny händelsetyp (schema) — exakta steg
8. Checklista för att ansluta en app
9. Vanliga misstag

---

## 1. Anslutningsdetaljer

- **Bas-URL (maskin-API):** `https://api.syntes.dev`
- **Autentisering:** varje anrop skickar headern `X-API-Key: <din nyckel>`.
- **Nyckeln identifierar dig.** Syntes sätter `source_app` i eventet från nyckeln —
  en app kan aldrig utge sig för att vara någon annan.
- **Scopes** (rättighetsnivåer på en nyckel):
  - `read` — får bara läsa (`GET /events`, `GET /snapshot`).
  - `write` — får bara publicera (`POST /events`).
  - `readwrite` — båda. (Backend-appar som Signal/Todos får denna.)
  - Allt som distribueras till en klient (Android-APK, widget) ska ha `read` — en läckt
    read-nyckel kan aldrig injicera falska events.
- **Felkoder:** `401` = ogiltig/saknad nyckel · `403` = nyckeln saknar rätt scope ·
  `422` = payload bröt mot schemat (avvisas i dörren, inget sparas).
- **Du får din nyckel av den som har VPS-åtkomst** (kör `scripts/create_key.py <app> <scope>`
  i syntes-containern + INSERT i databasen). Be om en nyckel med rätt scope och lägg den i
  din apps `.env` — **aldrig i git**.

> Not: `https://syntes.dev` (utan `api.`) är en människo-dashboard bakom Authelia-inloggning
> och är INTE till för maskiner. Maskiner använder alltid `https://api.syntes.dev`.

---

## 2. De tre endpointsen

| Metod | URL | Scope | Vad |
|---|---|---|---|
| `POST` | `/events` | write | Publicera en ny händelse |
| `GET` | `/events` | read | Hämta händelser (för processande konsumenter) |
| `GET` | `/snapshot` | read | Nulägesbild (för display-konsumenter) |
| `GET` | `/health` | ingen | Livskontroll → `{"status":"ok"}` |

Interaktiv dokumentation finns live på `https://api.syntes.dev/docs`.

### POST /events
Body:
```json
{ "event_type": "signal.recommendation.v1", "payload": { ... } }
```
Svar `201`:
```json
{ "id": 4711, "created_at": "2026-07-22T12:53:02Z" }
```
Syntes validerar `payload` mot schemat för `event_type`. Okänd typ eller fel fält → `422`.

### GET /events
Query-parametrar: `type` (valfri, filtrera på event_type), `after_id` (bokmärke, default 0),
`limit` (default 100, max 500).
```
GET /events?type=signal.recommendation.v1&after_id=4700
```
Svar:
```json
{
  "events": [
    { "id": 4711, "event_type": "...", "source_app": "signal",
      "payload": { ... }, "created_at": "..." }
  ],
  "last_id": 4711
}
```

### GET /snapshot
Statslös nulägesbild (läser färdigberäknade projektioner, inte hela loggen):
```json
{
  "generated_at": "2026-07-22T12:58:02Z",
  "open_buys": [ { "ticker": "VOLV-B", "signal_score": 88, "held": false, "run_date": "2026-07-22" } ],
  "today": { "workout_done": false, "tasks_completed": 0, "routines_checked": 0 },
  "portfolio": null
}
```

---

## 3. Vad som är implementerat i dagsläget (2026-07-22)

**Live och driftsatt på VPS:en:**
- Alla tre endpoints + `/health` + `/docs`, bakom X-API-Key.
- Nyckel-auth med scopes `read` / `write` / `readwrite`.
- Schema-validering (strikt) vid `POST /events`.
- Projektioner + `/snapshot` (Fas 8).
- En människo-dashboard på `https://syntes.dev` (bakom Authelia) som läser `/snapshot`.

**Registrerade händelsekontrakt (scheman) som finns just nu:**

| event_type | Producent (tänkt) | Uppdaterar projektion → syns i /snapshot som |
|---|---|---|
| `signal.recommendation.v1` | Signal | `proj_current_signals` → `open_buys` (action=BUY) |
| `task.created.v1` | Todos | *(ingen projektion ännu)* |
| `task.completed.v1` | Todos | `proj_daily_status.tasks_completed` → `today` |

**Projektionslogik finns redan förberedd** för `workout.completed.v1` (→ `today.workout_done`)
och `portfolio.daily_performance.v1` (→ `portfolio`), **men deras scheman finns inte ännu** —
så de eventtyperna avvisas tills schemat läggs till (se avsnitt 7).

**Inte byggt ännu (planerat):** `user`-scope med vitlistade eventtyper (för telefonen),
realtid/push (allt är polling), samt de flesta appars faktiska producent-/konsumentkod.

---

## 4. Händelsekontrakten (scheman) — reglerna

Ett schema är ett JSON Schema som exakt definierar en händelsetyps struktur. Det bor i
Syntes (`app/schemas/<event_type>.json`) och ÄR dokumentationen — följer du schemat kan du
lita blint på datan.

**Namnkonvention:** `objekt.verb.version`, alltid dåtid på verbet (händelsen har hänt):
`workout.completed.v1`, `task.created.v1`, `signal.recommendation.v1`.

**Tre hårda regler:**
1. **`additionalProperties: false`** — inga påhittade extrafält. Skickar du ett fält som inte
   finns i schemat → hela eventet avvisas (`422`). Detta är med flit.
2. **Ett publicerat schema ändras ALDRIG.** Behöver strukturen brytas → skapa `v2` bredvid
   `v1`. Att LÄGGA TILL ett *valfritt* (ej required) fält är ofarligt och kräver ingen ny version.
3. **Publicera bara fakta som hänt.** En klient får aldrig publicera `task.created.v1` innan
   ägande appen (Todos) faktiskt skapat uppgiften — annars ljuger loggen. (Se avsnitt 5d om commands.)

Exempel — det nuvarande `signal.recommendation.v1`:
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "signal.recommendation.v1",
  "type": "object",
  "required": ["ticker", "action", "signal_score", "run_date"],
  "properties": {
    "ticker":       { "type": "string" },
    "action":       { "type": "string", "enum": ["BUY", "NEUTRAL", "SELL"] },
    "signal_score": { "type": "number", "minimum": 0, "maximum": 100 },
    "held":         { "type": "boolean" },
    "run_date":     { "type": "string", "format": "date" },
    "model_version":{ "type": "string" }
  },
  "additionalProperties": false
}
```

---

## 5. Kommunikationsmönster — och vad som krävs på BÅDA håll

Det finns tre roller. En app kan ha flera samtidigt.

### 5a. Bli PRODUCENT (din app berättar att något hänt)

| På Syntes-sidan (en gång) | I din app |
|---|---|
| 1. Schemat för din eventtyp måste finnas i `app/schemas/`. Finns det inte — se avsnitt 7. | 1. Skaffa en nyckel med `write` (eller `readwrite`). |
| 2. Behöver händelsen synas i `/snapshot`? Lägg till projektionslogik (avsnitt 7). | 2. Lägg nyckeln + `SYNTES_URL=https://api.syntes.dev` i din `.env`. |
| | 3. Kopiera in `syntes_client.py` (avsnitt 6). |
| | 4. Anropa `publish("<event_type>", { ...payload })` när fakta inträffar. |

**Kritiskt:** publiceringen får ALDRIG krascha din app. Använd try/except + timeout (finns i
referensklienten). Om Syntes är nere ska din app fortsätta opåverkad — navet är ett tillägg,
aldrig ett beroende för kärnfunktionen.

### 5b. Bli PROCESSANDE KONSUMENT (din app reagerar på varje event, exakt en gång)

Exempel: Todos skapar en uppgift för varje BUY-signal från Signal.

| På Syntes-sidan | I din app |
|---|---|
| Inget nytt behövs — `GET /events` finns redan. Du behöver bara känna till vilken `event_type` du lyssnar på och dess schema. | 1. Skaffa en `read`- (eller `readwrite`-)nyckel. |
| | 2. En enradstabell i din egen db för bokmärket (`last_id`). |
| | 3. En poll-funktion (avsnitt 6), körd via cron var ~5:e minut. |

**Bokmärkesmönstret** (gör konsumtionen idempotent — inget tappas, inget dubblas):
1. Läs ditt sparade `last_id` (börjar på 0).
2. `GET /events?type=<typ>&after_id=<last_id>`.
3. Behandla varje event i svaret.
4. Spara `last_id` från svaret **EFTER** lyckad behandling.

Kraschar pollern mitt i körs samma events om nästa gång (eftersom bokmärket inte hann sparas)
— därför måste din behandling tåla att se samma event igen (t.ex. upsert på ett id ur payloaden).

### 5c. Bli DISPLAY-KONSUMENT (widget/dashboard som visar nuläget)

| På Syntes-sidan | I din app |
|---|---|
| Behöver nuläget du vill visa finnas som projektion? Om inte — lägg till en (avsnitt 7). | 1. Skaffa en `read`-nyckel. |
| | 2. Polla `GET /snapshot` med rimligt intervall (t.ex. var 15:e min för en widget). |
| | 3. Rita om skärmen ur svaret. **Ingen `after_id`-loop, inget bokmärke.** |

En display-konsument frågar "hur ser läget ut just nu?" och läser tillstånd, inte ström. Den
får aldrig ha en `after_id`-loop — det är bara för processande konsumenter.

### 5d. COMMANDS — be en annan app GÖRA något (inte bara berätta)

Ibland vill en klient (t.ex. telefonen) be en app skapa något som inte finns ännu. Då får den
INTE publicera faktum-eventet själv (objektet finns ju inte förrän ägaren skapat det).

Mönstret:
1. Klienten publicerar ett **command**: `objekt.verb.requested.v1` (t.ex. `task.create.requested.v1`).
2. Ägande appen (Todos) konsumerar commandet via sin poller, utför handlingen i sin egen db,
   och publicerar sedan **faktumet**: `task.created.v1`.

Namnkonvention: `*.requested.v1` för commands (imperativ), dåtidsverb för fakta. Latensen (upp
till en polling-cykel) hanteras i UI:t med en "skapas…"-status. *(Command-scheman och `user`-scope
är planerade men inte byggda ännu — bygg producent/faktum-flödet först.)*

---

## 6. Referenskod (klistra in i din app)

### `syntes_client.py` — publiceringsklient (för producenter)
```python
import os, requests

SYNTES_URL = os.environ["SYNTES_URL"]        # https://api.syntes.dev
SYNTES_KEY = os.environ["SYNTES_API_KEY"]

def publish(event_type: str, payload: dict) -> bool:
    """Publicerar ett event till Syntes. Kraschar ALDRIG anropande app."""
    try:
        r = requests.post(
            f"{SYNTES_URL}/events",
            json={"event_type": event_type, "payload": payload},
            headers={"X-API-Key": SYNTES_KEY},
            timeout=5,
        )
        r.raise_for_status()
        return True
    except requests.RequestException as e:
        print(f"[syntes] publicering misslyckades: {e}")
        return False
```

Användning (i slutet av det jobb som producerar fakta):
```python
publish("signal.recommendation.v1", {
    "ticker": rec.ticker,
    "action": rec.action,            # "BUY" | "NEUTRAL" | "SELL"
    "signal_score": rec.signal_score, # 0–100
    "held": rec.ticker in current_holdings,
    "run_date": str(rec.run_date),   # "YYYY-MM-DD"
})
```

### `poll_syntes()` — konsument-poller (för processande konsumenter)
```python
import os, requests

SYNTES_URL = os.environ["SYNTES_URL"]
SYNTES_KEY = os.environ["SYNTES_API_KEY"]

def poll_syntes():
    last_id = read_bookmark()   # läs sparat bokmärke ur din egen db (börjar på 0)
    r = requests.get(
        f"{SYNTES_URL}/events",
        params={"type": "signal.recommendation.v1", "after_id": last_id},
        headers={"X-API-Key": SYNTES_KEY},
        timeout=5,
    )
    r.raise_for_status()
    data = r.json()
    for event in data["events"]:
        p = event["payload"]
        if p["action"] == "BUY":
            create_task(f"Granska köpsignal: {p['ticker']} (score {p['signal_score']:.0f})")
        elif p["action"] == "SELL" and p.get("held"):
            create_task(f"Granska säljsignal: {p['ticker']} (score {p['signal_score']:.0f})")
    save_bookmark(data["last_id"])   # spara EFTER lyckad behandling
```
Kör `poll_syntes()` via cron var ~5:e minut. `read_bookmark`/`save_bookmark`/`create_task`
är din apps egna funktioner mot din egen databas.

---

## 7. Lägga till en NY händelsetyp (schema) — exakta steg (görs i Syntes)

Detta görs i **Syntes-repot** (`github.com/pepestal/syntes`), inte i din app. Om du är agent i
en annan app: leverera schemat + ev. projektionsönskemål som en tydlig spec till den som äger Syntes.

1. **Skapa schemafilen** `app/schemas/<objekt>.<verb>.v1.json` (följ exemplet i avsnitt 4;
   glöm inte `additionalProperties: false` och `title` = filnamnet utan `.json`).
2. **Behövs nuläge i `/snapshot`?** Lägg till en gren i `app/projections.py::apply_projections`
   för din `event_type` (upsert mot en projektionstabell). Ny tabell? Skapa en Alembic-migration
   (`migrations/versions/000X_*.py`) och en ORM-modell i `app/models.py`. Utöka `/snapshot`
   i `app/routers/snapshot.py` om fältet ska exponeras.
3. **Deploya Syntes:** `git pull && docker compose up -d --build` på VPS:en (migrationer körs
   automatiskt vid start).
4. **Fyll historik** (om du la till en projektion): `docker compose exec syntes-api python
   scripts/rebuild_projections.py` — spelar om hela loggen mot de nya projektionerna.
5. **Verifiera:** `POST` ett testevent och kontrollera `GET /events` / `GET /snapshot`.

---

## 8. Checklista för att ansluta en app (hela processen)

1. **Bestäm roll(er):** producent, processande konsument, display-konsument?
2. **Scheman:** vilka eventtyper producerar/konsumerar appen? Finns de i Syntes? Annars → avsnitt 7.
3. **Nyckel:** be om en nyckel med rätt scope (`read` för display, `readwrite` för backend).
   Lägg i `.env`, aldrig i git.
4. **Producent:** kopiera in `syntes_client.py`, anropa `publish()` vid relevanta fakta.
5. **Processande konsument:** poll-funktion + bokmärkestabell, cron var ~5:e min.
6. **Display-konsument:** polla `/snapshot`, rita om. Ingen bokmärkesloop.
7. **Verifiera med curl** att events flödar och att snapshoten speglar dem.

Det är allt. Ingen annan app rörs. Det är plug and play.

---

## 9. Vanliga misstag att undvika

1. **Låta publicering krascha appen** — alltid try/except + timeout. Navet är ett tillägg.
2. **Ge en display-klient en `after_id`-loop** — widgets/dashboards läser `/snapshot`, punkt.
3. **Skippa bokmärket i en processande konsument** — då behandlas samma events om och om igen.
4. **Ändra ett publicerat schema** — nej, skapa `v2`.
5. **Publicera faktum-events för saker som inte hänt än** — använd `*.requested.v1`-commands.
6. **Baka in en `readwrite`-nyckel i en APK** — allt som lämnar servern får `read`-nycklar.
7. **Skicka extrafält** — `additionalProperties: false` avvisar hela eventet.
8. **Läsa en annan apps databas direkt** — aldrig. All kommunikation går via Syntes API.
