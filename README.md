# LOKALT — översikt & arkitektur

Det här repot innehåller **enbart dokumentation** för hur mina appar hänger ihop.
De individuella apparna/repona ligger i egna mappar bredvid denna fil men är
**inte** en del av detta repo (de ignoreras via [.gitignore](.gitignore) och
versionshanteras var för sig).

Syftet är att kunna synka en gemensam beskrivning av systemet mellan mina olika
datorer.

## Appar

| App | Mapp | Repo | Beskrivning |
|-----|------|------|-------------|
| Signal | `Signal/` | — | _(fyll i)_ |
| Landing page | `landing-page/` | — | _(fyll i)_ |
| Stronk | `stronk/` | _(länk)_ | _(fyll i)_ |
| Syntes | `syntes/` | _(länk)_ | _(fyll i)_ |
| Todos | `todos/` | _(länk)_ | _(fyll i)_ |

## Dokumentation

- [Arkitektur & kommunikation](docs/architecture.md) — hur apparna pratar med varandra
- Lägg till fler dokument i [docs/](docs/) efter behov.

## Så här jobbar du med detta repo på en ny dator

```bash
# Klona in i LOKALT-mappen (eller klona och döp om till LOKALT)
git clone <repo-url> LOKALT
cd LOKALT

# Klona sedan de individuella apparna bredvid dokumentationen som vanligt
git clone <app-repo-url>
```

Dagligt flöde: `git pull` när du sätter dig, `git add`/`commit`/`push` när du
uppdaterat dokumentationen.
