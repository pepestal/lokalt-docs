# Dokumentationsmall (framework för nya appar)

En kopieringsfärdig startuppsättning som gör att ett nytt repo följer
[dokumentationsstandarden](../documentation_standard.md) från första commit.

Använd detta för **portal** och alla framtida projekt som saknar dokumentation.

## Så använder du mallen

1. Kopiera innehållet i denna mapp in i det nya repots rot:

   ```bash
   # från LOKALT/, med det nya repot klonat bredvid (t.ex. portal/)
   cp docs/templates/APP_README.md   portal/README.md
   cp docs/templates/APP_CLAUDE.md   portal/CLAUDE.md
   cp -r docs/templates/docs         portal/docs
   ```

   > Filerna heter `APP_README.md` / `APP_CLAUDE.md` här bara för att de annars
   > skulle krocka med denna mapps egen `README.md`. Vid kopiering döps de till
   > `README.md` respektive `CLAUDE.md` i målrepot (kommandona ovan gör det).

2. Sök/ersätt platshållarna i alla kopierade filer:

   | Platshållare | Betyder | Exempel |
   |---|---|---|
   | `{{APP}}` | Appens namn, versal | `Portal` |
   | `{{app}}` | Repo-/mappnamn, gemener | `portal` |
   | `{{ETT_ORD}}` | Appens ledstjärna i en mening | `En snabb landningssida.` |
   | `{{ROLL}}` | Roll i navet | `producent` / `konsument` / `—` |
   | `{{STACK}}` | Teknikstack i en rad | `Vite + React + TS` |

3. Ta bort de valfria dokument du inte behöver ännu (`ROADMAP.md`,
   `ARCHITECTURE.md`, `INTEGRATION.md`), men behåll `README.md`, `docs/README.md`
   och `docs/CHANGELOG.md` — de är obligatoriska.

4. Lägg in det nya repot i [`../documentation_map.md`](../documentation_map.md).

## Vad som ingår

```
APP_README.md    → README.md    (obligatorisk ingång)
APP_CLAUDE.md    → CLAUDE.md     (om AI-agenter används)
docs/
  README.md      (obligatorisk – index)
  CHANGELOG.md   (obligatorisk)
  STATUS.md      (rekommenderad)
  ROADMAP.md     (rekommenderad)
  ARCHITECTURE.md(vid behov)
  INTEGRATION.md (om appen rör Syntes)
```
