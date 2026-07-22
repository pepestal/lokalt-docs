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
| Signal (backend) | `Signal/backend/` | [pepestal/…](https://github.com/pepestal) | _(fyll i)_ |
| Signal (frontend) | `Signal/signal_frontend/` | [pepestal/…](https://github.com/pepestal) | _(fyll i)_ |
| Portal | `portal/` | [pepestal/portal](https://github.com/pepestal/portal) | _(fyll i)_ |
| Stronk | `stronk/` | [pepestal/stronk](https://github.com/pepestal/stronk) | _(fyll i)_ |
| Syntes | `syntes/` | [pepestal/syntes](https://github.com/pepestal/syntes) | _(fyll i)_ |
| Todos | `todos/` | [pepestal/todos](https://github.com/pepestal/todos) | _(fyll i)_ |

> **Signal** är en paraplymapp: dess övergripande dokumentation ligger i detta
> repo, men de två underrepona `backend/` och `signal_frontend/` versionshanteras
> var för sig och ignoreras här.

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
