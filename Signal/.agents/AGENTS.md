# Observera! — vägledning för Signal (paraplymapp, inte ett repo)

Denna mapp (`Signal/`) är **inte** ett repo och inte en del av projektets kod. Den
används enbart för att organisera de **två separata repona** som ligger i
undermapparna, sida vid sida i lokal miljö:

- `backend/` — Signal backend (FastAPI + PostgreSQL). Eget repo.
- `signal_frontend/` — Signal frontend (Next.js). Eget repo.

Mappen fungerar som en **redirect för agenter**: den äger ingen egen dokumentation
utan pekar in i respektive repo. Båda repona följer den gemensamma
[dokumentationsstandarden](https://github.com/pepestal/lokalt-docs/blob/main/docs/documentation_standard.md).

## Var dokumentationen finns

**Frontend** (`signal_frontend/`):
- `README.md` — teknisk ingång (stack, snabbstart, struktur).
- `docs/` — fördjupning. **Börja i `docs/OVERVIEW.md`** (karta + kända avvikelser);
  `docs/README.md` är index. UI-regelverket: `docs/UI_LAW.md` m.fl.
- `.agents/` + `.claude/` — agentregler och skills.

**Backend** (`backend/`):
- `README.md` — teknisk ingång (snabbstart, hemligheter, deployment).
- `docs/` — fördjupning. **Börja i `docs/OVERVIEW.md`**; `docs/README.md` är index;
  `docs/ARCHITECTURE.md` / `docs/API.md` / `docs/DATABASE.md` / `docs/ENGINES.md`.
- `.agents/` + `.claude/` — agentregler och skills (spegel av varandra).

Projektet är live på **https://signal.syntes.dev/**.

## Regel

Ingen dokumentation, utöver denna vägledande fil, ska skapas eller finnas i denna
mapp (`Signal/`). Doc-ändringar hör hemma i respektive repo enligt standarden ovan.
