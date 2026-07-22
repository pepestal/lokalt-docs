# Dokumentationsstandard

Den gemensamma standarden för hur **alla appar i ekosystemet** dokumenterar sig
själva. Målet är att en app ska kännas igen direkt: samma filnamn, samma platser,
samma roller — oavsett om du öppnar `syntes`, `stronk` eller ett nystartat projekt.

> Detta dokument är **normen** (single source of truth för struktur). Nuläget per
> app — och hur långt varje app kommit mot normen — beskrivs i
> [`documentation_map.md`](documentation_map.md). En kopieringsfärdig startuppsättning
> ligger i [`templates/`](templates/).

---

## 1. Principer

1. **Rot-`README.md` är ingången.** Verktyg, GitHub och nya läsare börjar där.
   All fördjupning bor i `docs/`.
2. **Fasta dokument har fasta namn.** Samma roll → samma filnamn i varje repo.
3. **Engelska VERSALER för filnamn, svenskt innehåll.** `README.md`,
   `CHANGELOG.md`, `ROADMAP.md`, `STATUS.md`, `INTEGRATION.md`, `ARCHITECTURE.md`.
   Texten inuti skrivs på svenska.
4. **Ett dokument, en roll.** Dela inte samma information mellan två filer — länka
   istället. Vid krock gäller den utpekade sanningskällan (t.ex. Syntes för
   integration).
5. **Dokumentation är en del av ändringen.** En kodändring är inte klar förrän
   `README`/`CHANGELOG` speglar den.

---

## 2. Kanonisk filuppsättning

### Repo-roten

| Fil | Roll | Krav |
|-----|------|------|
| `README.md` | Teknisk ingång ("bibeln"): syfte, arkitektur i korthet, mappstruktur, teknikstack, snabbstart, konventioner. | **Obligatorisk** |
| `CLAUDE.md` | Hårda regler och operativa instruktioner för AI-agenter. Kort och operativ; djup finns i länkade dokument. Ligger i **roten** (auto-läses av Claude Code). | Om AI-agenter används |

### `docs/`

| Fil | Roll | Krav |
|-----|------|------|
| `docs/README.md` | **Index** över `docs/` + "var finns vad"-tabell för hela projektet. Kort — inte en andra bibel. | **Obligatorisk** |
| `docs/CHANGELOG.md` | Kronologisk ändringslogg. Keep-a-Changelog-stil med `## [Ej släppt]` överst. | **Obligatorisk** |
| `docs/STATUS.md` | Nuläge: var vi står, nästa steg, öppna beslut, infra-förutsättningar. Ingången för den som ska *ta vid*. (Ersätter tidigare `NULAGE.md`.) | Rekommenderad |
| `docs/ROADMAP.md` | Versionsplan, vad som medvetet ligger efter v1, när/om det kommer. **Öppnar alltid med en `## Idéer / att utreda`-sektion** (se §6). | Rekommenderad |
| `docs/ARCHITECTURE.md` | Djupare arkitektur: datamodell/DDL, diagram, komponenter, dataflöden. | Vid behov |
| `docs/INTEGRATION.md` | Syntes-integration. **För appar som pratar med Syntes:** en kort pekare till sanningskällan `syntes/docs/INTEGRATION.md` + appens eget kontrakt (vilka events den producerar/konsumerar). **För Syntes själv:** detta *är* källan. | Om appen rör Syntes |
| `docs/DEPLOY.md` | Driftsättning: Docker Compose, Caddy, backup-rutiner. | Vid behov |
| Övriga fördjupningar | T.ex. `docs/engines/`, UI-dokument, komponent-djupdykningar. Se §4 om namn. | Vid behov |

---

## 3. Namnregler

- **Fasta dokument** (tabellerna ovan): exakt de namnen, VERSALER, `.md`.
- **Case är låst** — `CHANGELOG.md`, aldrig `Changelog.md`/`changelog.md`. (På
  skiftlägesokänsliga filsystem som Windows är detta extra viktigt: två filer som
  bara skiljer sig i skiftläge kollapsar.)
- **Inga numeriska prefix** (`1_Arkitektur.md`, `3_State.md`). De ger luckor och
  låst ordning. Vill du ha ordning: styr den via listan i `docs/README.md`, inte
  via filnamn. Undantag: en genuint sekventiell serie som läses i ordning får ha
  prefix, men då utan luckor.
- **Fördjupningsdokument** (icke-fasta) döps beskrivande i VERSALER eller
  `Title_Case`, utan mellanslag och specialtecken — använd `_` (t.ex.
  `MOTOR_ROADMAP.md`, inte `Motor_Dev&Roadmap.md`; `&` och mellanslag krånglar i
  URL:er och shell).
- **Mappen heter `docs/`** och ligger i **repo-roten** — aldrig `src/docs/`.

---

## 4. Placering

```
repo/
├── README.md              ← ingång (obligatorisk)
├── CLAUDE.md              ← AI-regler (om agenter används)
└── docs/
    ├── README.md          ← index (obligatorisk)
    ├── CHANGELOG.md       ← ändringslogg (obligatorisk)
    ├── STATUS.md          ← nuläge
    ├── ROADMAP.md         ← framtid
    ├── ARCHITECTURE.md    ← djup arkitektur
    ├── INTEGRATION.md     ← Syntes-koppling
    └── ...                ← fördjupningar vid behov
```

---

## 5. Korsreferenser (viktigast vid migrering)

När ett dokument byter namn eller plats måste **varje referens till det** följa med.
Referenser bor typiskt i:

- Andra `.md`-filer (relativa länkar, `[text](STATUS.md)`).
- `README.md` / `docs/README.md` (index-tabeller).
- `CLAUDE.md` och `.agents/AGENTS.md` (pekar ofta ut läs-först-dokument).
- Kod: `.py`, `.ts`, `.tsx`, kommentarer och docstrings som hänvisar till en fil.
- CI/skript och `mkdocs.yml`/liknande om sådant finns.

**Regel:** byt aldrig ett filnamn utan att först söka igenom repot efter det gamla
namnet (`grep -ri "gammalt_namn"`) och rätta varje träff i samma commit.

---

## 6. Innehållskonventioner

- **Språk:** svenska. Teknisk, koncis ton.
- **CHANGELOG:** rubriker `## [Ej släppt]` överst, sedan versioner. Under varje:
  `### Tillagt` / `### Ändrat` / `### Fixat` / `### Borttaget`. Rader i imperativ.
- **STATUS:** börjar med en enradig statusrad (t.ex. "🚧 planering klar, kod ej
  påbörjad") och pekar ut *nästa konkreta steg*.
- **ROADMAP:** öppnar alltid med en `## Idéer / att utreda`-sektion (först i
  innehållsförteckningen) — ett fritt, personligt snabb-capture-område för uppslag
  *innan* de är versionsplacerade eller beslutade. Ingen punkt där är ett åtagande;
  när en idé mognar flyttas den vidare (→ `STATUS.md` `Öppna beslut`, en versions­rubrik,
  eller arkitektur-/implementationsdokumentet). Syftet är en låg tröskel att skriva
  ner en tanke som senare kan tas upp med en agent för diskussion.
- **Syntes-integration:** appens egen `INTEGRATION.md` sammanfattar bara — vid minsta
  konflikt gäller `syntes/docs/INTEGRATION.md`. Skriv det uttryckligen i dokumentet.

---

## 7. Nytt projekt

Kopiera [`templates/`](templates/) rakt in i det nya repot och sök/ersätt
platshållarna (`{{APP}}`, `{{app}}`, `{{ROLL}}`, `{{ETT_ORD}}`). Se
[`templates/README.md`](templates/README.md) för exakt lista.
