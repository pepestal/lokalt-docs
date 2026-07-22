# Syntes-integration — {{APP}}

Hur {{APP}} kopplar mot händelsenavet **Syntes**.

> **Sanningskälla:** den fullständiga integrationsguiden ägs av Syntes-repot och
> underhålls där — [`syntes/docs/INTEGRATION.md`](https://github.com/pepestal/syntes/blob/main/docs/INTEGRATION.md).
> Vid minsta konflikt mellan detta dokument och källan **gäller källan** —
> uppdatera då detta dokument. Se även ekosystemets
> [arkitekturöversikt](https://github.com/pepestal/lokalt-docs/blob/main/docs/architecture.md).

## {{APP}}s roll

{{APP}} är **{{ROLL}}** mot Syntes.

## Kontrakt

**Producerar:**
- `<objekt.verb.v1>` — <!-- när, och vad payloaden betyder -->

**Konsumerar:**
- `<objekt.verb.v1>` — <!-- vad appen gör när eventet dyker upp -->

## Anslutning (sammanfattning)

- **Bas-URL:** `https://api.syntes.dev` · auth via `X-API-Key` (i `.env`, aldrig i git).
- **Scope:** `read` / `write` / `readwrite` — {{APP}} behöver `<scope>`.
- **Endpoints:** `POST /events` (publicera), `GET /events` (konsumera, bokmärkes­mönster), `GET /snapshot` (display).
- **Robusthet:** publicering sväljer alla fel (`try/except`, timeout) — får aldrig krascha appen.
- **Scheman:** `objekt.verb.version`, dåtidsverb, `additionalProperties: false`,
  publicerat schema ändras aldrig (→ `v2`).

Fullständiga detaljer, referenskod och checklista: se sanningskällan ovan.
