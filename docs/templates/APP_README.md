# {{APP}}

> {{ETT_ORD}}

Kort om vad appen är och dess roll i ekosystemet **syntes.dev**: {{APP}} är
**{{ROLL}}** mot händelsenavet **Syntes** (se [`docs/INTEGRATION.md`](docs/INTEGRATION.md)).

**Status:** se [`docs/STATUS.md`](docs/STATUS.md) — börja där om du ska ta vid.

---

## Innehåll

- [Vad appen gör](#vad-appen-gör)
- [Arkitektur i korthet](#arkitektur-i-korthet)
- [Teknikstack](#teknikstack)
- [Snabbstart](#snabbstart)
- [Konventioner](#konventioner)
- [Dokumentation](#dokumentation)

## Vad appen gör

<!-- 3–6 meningar: kärnvärdet, avgränsningen, vem den är till för. -->

## Arkitektur i korthet

<!-- Kort översikt/diagram. Djup i docs/ARCHITECTURE.md. -->
Varje app äger sin egen databas. Ingen app läser direkt i en annans — all
app-till-app-kommunikation går via Syntes API (`https://api.syntes.dev`).

## Teknikstack

| Lager | Val |
|-------|-----|
| Frontend | {{STACK}} |
| Backend | Python 3.12 + FastAPI |
| Databas | PostgreSQL 16 (egen instans) |
| Drift | Docker Compose + Caddy på VPS |

## Snabbstart

```bash
git clone https://github.com/pepestal/{{app}}.git
cd {{app}}
cp .env.example .env      # checkas ALDRIG in
docker compose up --build
```

## Konventioner

- Hemligheter i `.env`, aldrig i git.
- Dokumentation uppdateras i samma commit som koden (se nedan).

## Dokumentation

All fördjupning ligger i [`docs/`](docs/) — se [`docs/README.md`](docs/README.md)
för index. Följer den gemensamma
[dokumentationsstandarden](https://github.com/pepestal/lokalt-docs/blob/main/docs/documentation_standard.md).

**Vid varje kodändring, i samma commit:**
1. Uppdatera denna `README.md` om struktur/API/stack/uppstart påverkas.
2. Lägg en rad i [`docs/CHANGELOG.md`](docs/CHANGELOG.md) under `## [Ej släppt]`.
