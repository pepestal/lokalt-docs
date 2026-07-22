# Syntes — integrationsguide (pekare)

Den fullständiga integrationsguiden (endpoints, scheman, referenskod, checklista för att
ansluta en app) **ägs av Syntes-repot** och underhålls där — inte här — för att undvika
att två kopior glider isär.

**Källa (single source of truth):**
[`syntes/docs/INTEGRATION.md`](https://github.com/pepestal/syntes/blob/main/docs/INTEGRATION.md)

Lokalt, om du klonat apparna enligt [../README.md](../README.md):
`syntes/docs/INTEGRATION.md`

---

Snabb sammanfattning (för fullständiga detaljer, läs källan ovan):

- **Syntes är navet.** Appar pratar bara med Syntes, aldrig direkt med varandra.
- **Bas-URL (maskin-API):** `https://api.syntes.dev` · auth via `X-API-Key` · scopes `read` / `write` / `readwrite`.
- **Tre endpoints:** `POST /events` (publicera), `GET /events` (processande konsument, bokmärkesmönster), `GET /snapshot` (display-konsument).
- **Scheman:** `objekt.verb.version`, dåtidsverb, `additionalProperties: false`, publicerat schema ändras aldrig (→ `v2`).

Se [architecture.md](architecture.md) för diagram och nuvarande händelseflöden mellan apparna.
