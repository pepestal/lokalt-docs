# Arkitektur — {{APP}}

Djupare beskrivning av hur {{APP}} är byggd. Översikten i korthet finns i
[`../README.md`](../README.md); detta dokument går på djupet.

## Komponenter

<!-- Mappstruktur, moduler och deras ansvar. -->

## Datamodell

<!-- Tabeller/DDL med motivering. Varje app äger sin egen databas. -->

## Dataflöden

<!-- Hur data rör sig genom appen; ev. diagram (mermaid/ascii). -->

## Roll i navet

{{APP}} är **{{ROLL}}** mot Syntes. Kontraktet (vilka events som produceras/
konsumeras) beskrivs i [`INTEGRATION.md`](INTEGRATION.md). Ingen app läser direkt
i en annans databas — all app-till-app-kommunikation går via Syntes API.
