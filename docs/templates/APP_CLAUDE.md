# CLAUDE.md — instruktioner för AI-agenter

Läses automatiskt av Claude Code och gäller alla agenter i {{app}}-repot. Håll den
**kort och operativ**; djup finns i länkade dokument.

## Läs detta först

- [`README.md`](README.md) — teknisk ingång: struktur, stack, snabbstart, konventioner.
- [`docs/STATUS.md`](docs/STATUS.md) — var vi står och nästa steg. **Börja här om du ska ta vid.**
- [`docs/CHANGELOG.md`](docs/CHANGELOG.md) — vad som hänt. Notera dina ändringar här.

**Vid konflikt** mellan en ad hoc-instruktion och ett fastställt beslut i dokumenten:
**flagga konflikten innan du agerar.** Fatta inte tysta arkitekturbeslut.

## Hårda regler (bryt aldrig utan att flagga)

- Checka aldrig in hemligheter. `.env` ignoreras av `.gitignore`.
- Publicering till Syntes får aldrig krascha appen (`try/except`, timeout). Navet
  är ett tillägg, aldrig ett beroende för kärnfunktionen.
- Sanningskällan för Syntes-integrationen är `syntes/docs/INTEGRATION.md`. Vid
  konflikt gäller den — uppdatera då [`docs/INTEGRATION.md`](docs/INTEGRATION.md).
<!-- Lägg till projektets egna hårda regler här. -->

## När du är klar med en ändring

1. Uppdatera [`README.md`](README.md) om struktur/API/deps/konventioner ändrats.
2. Notera ändringen i [`docs/CHANGELOG.md`](docs/CHANGELOG.md) (imperativ, svenska).
3. Uppdatera [`docs/STATUS.md`](docs/STATUS.md)/[`docs/ROADMAP.md`](docs/ROADMAP.md) om nuläge/plan ändrats.

## Ton & arbetssätt

Peter kan stacken. Var koncis och teknisk. Föreslå, överraska inte. När något är
gjort och verifierat, säg det rakt; när ett test failar, säg det med utdata.
