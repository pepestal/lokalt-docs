# Statistik — spec & genomförandeplan för Signals statistiksida (Update 1.4)

Samlad specifikation för menypunkten **Statistik**. Varje mått beskrivs med
definition, exakt beräkning, datakrav och fallgropar. Allt beräknas i den nattliga
batchen och skrivs till färdiga aggregat-tabeller — frontend ska bara läsa och rita.

> **Levande dokument — primär spec+plan för update 1.4.** Avsnitt 0 (nuläge, beslut,
> fasplan) hålls uppdaterat vid varje leverans. Panel-specarna (§1–4) är den stabila
> referensen; datamodell + implementationsavsnittet längst ned styr genomförandet.

**Genomgående principer**

- **Point-in-time:** all motorstatistik beräknas på vad motorn *faktiskt sa* vid
  signaltillfället. Signalhistoriken finns **redan** oföränderlig i `recommendations`
  (`UNIQUE(ticker, engine_name, date)` → en rad per motor/aktie/dag) — räkna aldrig
  om historiska signaler med dagens modellversion. (Stämpla gärna `features_version`
  i `metrics` för att kunna spåra modellbyten; se datamodellen.)
- **Minsta urval:** visa `–` eller gråa ut paneler när n < 30. Hellre tomt än lögn.
- **Forward return** definieras överallt som `r_h = P(t+h) / P(t) − 1` över
  handelsdagar, helst på totalavkastning (justerat för utdelning/split).

---

## 0. Nuläge, beslut & fasplan

### 0.1 TL;DR — var vi står (uppdaterad 2026-07-22)

Update 1.4 är **påbörjad och till stor del i produktion**. Statistik-sidan + motor-Analys är live.

- ✅ **Fas 0 (fundament) — DEPLOYAD.** Egen route `/statistik` (4 flikar), sidebar-/mobilnav,
  `PortfolioPerformancePanel` flyttad hit, brygg-kort ersätter gamla `/portfolio`-Statistik-fliken.
- ✅ **Fas 1 (billiga vinster) — DEPLOYAD.** Affärsstatistik (§2.4 full), underwater+recovery (§2.2),
  fördelning per land (§2.6 interim), big numbers (§4.1). Allt klientberäknat, rika tooltips.
- ✅ **Fas 2a (motor-hit rate) — DEPLOYAD & verifierad live.** `GET /api/engines/hitrate` + flik
  **Analys** på `/engines`: hit rate + Wilson-CI + n per motor, horisont 20/40/60 (on-the-fly).
- ✅ **Fas 2b steg 1 (basrat) — BYGGD, EJ DEPLOYAD.** Nattbatch `scripts/compute_stats.py` +
  auto-skapade tabeller `signal_outcomes` + `stats_daily`; `/hitrate` returnerar nu **basrat + edge**
  (`hit_rate − basrat`), Analys-panelen visar edge + basrat-referenslinje. **Deploy:** backend-rebuild
  (tabeller auto-skapas via `create_all`) + **ny cron-rad** för `compute_stats.py` + engångskörning.
  INGEN manuell CREATE TABLE.
- ✅ **Fas 2b steg 2 (kalibrering/Brier/deciler) — BYGGD & DEPLOYAD (live-verifierad 2026-07-22).** §1.2 kalibreringskurva + ECE,
  §1.3 rullande Brier/BSS/AUC, §1.4 decilstaplar + Spearman — tre read-endpoints (`/calibration`,
  `/brier`, `/deciles` i `engines.py`) + tre ML-paneler i `EngineAnalys.tsx`, ovanpå `signal_outcomes`.
  `compute_stats.py` utökad att materialisera ML:s hela universum (probability-rader, ej bara BUY).
  **Deploystatus:** koden hängde med i steg 1:s backend-rebuild — alla tre endpoints svarar HTTP 200 live
  (agent-verifierat; icke-deployad = 404). ⏳ Visar tomt tills ML-facit mognar (~2026-08-06, h=20).
- ✅ **Fas 4 §3 (Marknaden — HELA fliken) — BYGGD, EJ DEPLOYAD.** Facit-OBEROENDE, riktig data direkt ur
  prishistoriken. Ny route `app/api/routes/statistics.py` (`/api/statistics`, monterad i `main.py`) med **fem**
  endpoints: `/breadth` (§3.1), `/nordic-race` (§3.3), `/highs-lows` (§3.4), `/seasonality` (§3.5, kuriosa-märkt),
  `/sector-treemap` (§3.2). Frontend: fem paneler i `statistik/components/` (`NordicRacePanel`, `SectorTreemap`
  [squarified], `BreadthPanel`, `HighsLowsPanel`, `SeasonalityPanel`) + delad `LineChartSvg` — ersätter Marknaden-
  placeholdern helt. **Deploy:** backend-rebuild + frontend (INGEN ny tabell/cron; on-the-fly SQL). Rök-test alla fem endpoints.
- ✅ **Fas 3 §1.5 (motorkorrelation + konsensus) — BYGGD, EJ DEPLOYAD.** `GET /api/engines/correlation`
  (Spearman-heatmap på dagliga signal_score-rankningar, **facit-oberoende**) + `/consensus?horizon=`
  (köpsignaler grupperade efter antal samstämmiga motorer; fördelning nu, hit rate/avkastning mognar med facit)
  → `CorrelationPanel`+`ConsensusPanel` på `/engines→Analys`. On-the-fly, ingen tabell/cron. **Deploy:** backend-rebuild + frontend.
- ✅ **Fas 3 §2.5 (beslutsalfa — KRONJUVELEN) — BYGGD, EJ DEPLOYAD.** Ny tabell `shadow_daily` (auto-skapas via
  `create_all`) fylls av `compute_stats.py::compute_shadow_portfolio` i den BEFINTLIGA nattbatchen (ingen ny cron):
  mekanisk `SIGNAL_SCORE_V1`-skugga (köp BUY / sälj SELL+60d-tidsstopp / likaviktat / 0,6 % rundtur, **regler spikade**).
  `GET /api/statistics/decision-alpha` + `DecisionAlphaPanel` på `/statistik→Affärer` (faktisk−skugga, rebasat till 100).
  Offline sim-testad (rising→TWR, tidsstopp=60d, kostnader). **Deploy:** backend-rebuild + engångskörning `compute_stats` + frontend.
- 🔜 **NÄSTA: hela 1.4-kärnan byggd.** Kvar: resten av §1 (basrat/kalibrering/Brier/deciler + konsensus-hit-rate)
  får liv allt eftersom facit mognar (fr.o.m. 2026-07-23 TA → ~08-06 ML) — bara VÄNTA, ingen kod. Valfri svans:
  §4-kuriosa (batchtider `job_runs`, streaks, coin-flip), lös-hängande §2.3 Sharpe/Sortino + §2.6 sektor/över-tid.

  > ⚠️ **Facit-mognad styr §1, inte kod.** Live `/hitrate` visar alla 4 motorer `maturing`, n=0, base_rate=null
  > (agent-verifierat 2026-07-22). Första facit 2026-07-23 (TA), FUND 07-29, ML ~08-06, INSIDER ~08-17. Därför
  > prioriteras facit-oberoende paneler (Fas 4 §3) medan facit mognar.
- ⛔ **Utanför 1.4:** regim-panelerna (§1.6 + regimvarianterna i §1.4/1.5) — kräver en regimmotor
  som medvetet är bortprioriterad.

### 0.2 Arkitekturbeslut (styr resten)

1. **Motor- vs portföljstatistik separeras efter *vem panelen tjänar*, inte datakälla.**
   - **Motor-/research-statistik (§1)** → bor på **`/engines`** (flik **Analys**, bredvid
     Översikt/Shadow/Detaljer). Håller research skilt från pengar. ✅ Analys-fliken finns (Fas 2a).
   - **Portfölj-/beslutsstatistik (§2)** → sidomeny-fliken **Statistik**. ✅ Finns (Fas 0).
   - **Marknad (§3) + system/kuriosa (§4)** → egna undermenyflikar i Statistik.
2. **Regim-beroende paneler skjuts upp.** Det finns **ingen** regim-/HMM-tabell och ingen
   regimmotor (HMM avfördes som egen motor 2026-07-13 — regimproxies testas som features
   först). Paneler som kräver den (§1.6 regimband, regim-uppdelningen i §1.4, 4×4-heatmapen
   i §1.5) är märkta **⛔ BLOCKERAD** och ligger utanför 1.4.
3. **Ingen ny `signals_history`-tabell.** `recommendations` ÄR redan den immutabla
   point-in-time-signalhistoriken. Statistiken byggs ovanpå den.
4. ✅ **Namnkollisionen löst (Fas 0).** Det fanns tre "Statistik"-ytor. Slutläge: **en** "Statistik"
   (sidomeny, pengar) + **Shadow/Analys** (motorer). Gamla `/portfolio`-Statistik-fliken är nu ett
   brygg-kort → `/statistik?tab=portfolj`; `/engines→Shadow` länkar tillbaka dit.

### 0.3 Statusöversikt per panel (🟢 klar · 🟡 delvis · 🔴 saknas · ⛔ blockerad)

| Panel | Status | Var / kommentar |
|---|---|---|
| 1.1 Hit rate + basrat | 🟢 | ✅ Hit rate + **Wilson-CI** + n + per-horisont (Fas 2a) + **basrat + edge** (`hit_rate − basrat`, Fas 2b steg 1 via `compute_stats.py`→`stats_daily`), `/engines→Analys`. Kvar: dedup av överlappande fönster (mindre). |
| 1.2 Kalibrering / ECE | 🟢 | ✅ **Byggt (Fas 2b steg 2), EJ DEPLOYAD.** `GET /api/engines/calibration` + `CalibrationPanel.tsx`: reliability-scatter mot y=x + ECE, kvantilhinkar. Gäller ML. ⏳ facit mognar. |
| 1.3 Brier / AUC rullande | 🟢 | ✅ **Byggt (Fas 2b steg 2), EJ DEPLOYAD.** `GET /api/engines/brier` + `BrierPanel.tsx`: Brier/BSS/AUC nuvärden + rullande tidsserie. AUC/Spearman utan sklearn. Gäller ML. |
| 1.4 Forward return per decil | 🟢 / ⛔ | ✅ **Grundvariant byggd (Fas 2b steg 2), EJ DEPLOYAD.** `GET /api/engines/deciles` + `DecilesPanel.tsx`: cross-sectional decilstaplar + Spearman rho. Regim-split ⛔ (utanför 1.4). |
| 1.5 Motorkorrelation/konsensus | 🟢 / ⛔ | ✅ **Byggt (Fas 3), EJ DEPLOYAD.** `GET /api/engines/correlation` (Spearman-heatmap, **facit-oberoende**) + `/consensus?horizon=` (konsensus-buckets, hit rate mognar med facit) → `CorrelationPanel`/`ConsensusPanel` på `/engines→Analys`. 4×4 med regim ⛔ (utanför 1.4). |
| 1.6 Regimband | ⛔ | Ingen regim-tabell. Utanför 1.4. |
| 2.1 Equity vs benchmark (TWR) | 🟢 | `PortfolioPerformancePanel`, nu på `/statistik→Portfölj`. Benchmark = OMX SPI + likaviktat universum. |
| 2.2 Drawdown / underwater | 🟢 | ✅ **Byggt (Fas 1)** — `DrawdownPanel.tsx`: underwater-SVG + max DD + längsta recovery + under-vatten-nu. |
| 2.3 Rullande Sharpe/Sortino | 🔴 | Klientberäkningsbart ur TWR-serien (lös-hängande, kan tas när som — ej i någon bunden fas). |
| 2.4 Affärsstatistik | 🟢 | ✅ **Byggt (Fas 1)** — `AffarsStatistik.tsx`: win rate/PF/expectancy/payoff/snittvinst-förlust/innehavstid + R-histogram + bästa/sämsta. |
| 2.5 Beslutsalfa | 🟢 | ✅ **Byggt (Fas 3), EJ DEPLOYAD.** `shadow_daily` (ny tabell, auto-skapas) fylls av `compute_stats.py::compute_shadow_portfolio` (mekanisk `SIGNAL_SCORE_V1`-skugga: köp BUY / sälj SELL+60d-stopp / likaviktat / 0,6 % rundtur, regler spikade). `GET /api/statistics/decision-alpha` + `DecisionAlphaPanel.tsx` (`/statistik→Affärer`, faktisk−skugga rebasat). **Kronjuvelen.** |
| 2.6 Exponering över tid | 🟡 | ✅ **Snapshot per land + kassa byggt (Fas 1)**, klientsidan. Saknas: **över tid** (stackad area) + **per sektor** — kräver backend-sektorjoin + per-position-historik. |
| 3.1 Breadth | 🟢 | ✅ **Byggt (Fas 4 §3-A), EJ DEPLOYAD.** `GET /api/statistics/breadth` + `BreadthPanel.tsx`: andel > MA20/50/200 över tid, nivåer 20/50/80 %. On-the-fly window-SQL. |
| 3.2 Sektor-treemap | 🟢 | ✅ **Byggt (Fas 4 §3-C), EJ DEPLOYAD.** `GET /api/statistics/sector-treemap?window=` + `SectorTreemap.tsx` (squarified, storlek=börsvärde, färg=kapitalviktad avk. idag/vecka). |
| 3.3 Nordenkampen | 🟢 | ✅ **Byggt (Fas 4 §3-A), EJ DEPLOYAD.** `GET /api/statistics/nordic-race` + `NordicRacePanel.tsx`: likaviktat landindex→100 vid årsskiftet. Land ur ticker-suffix. |
| 3.4 52v high/low | 🟢 | ✅ **Byggt (Fas 4 §3-B), EJ DEPLOYAD.** `GET /api/statistics/highs-lows` + `HighsLowsPanel.tsx`: highs/lows/net + kumulativ net + dagliga net-staplar. |
| 3.5 Säsongsmönster | 🟢 | ✅ **Byggt (Fas 4 §3-B), EJ DEPLOYAD.** `GET /api/statistics/seasonality` + `SeasonalityPanel.tsx`: medel per månad/veckodag + felstaplar, **märkt kuriosa**. |
| 4.1 Big numbers | 🟢 | ✅ **Byggt (Fas 1)** — `BigNumbers.tsx` ur `/api/admin/status` (datapunkter/bolag/valutakurser/synk). Saknas: signalräkning + äldsta datapunkt (nya COUNT-queries, senare). |
| 4.2 Batchkörningstider | 🔴 | Kräver `job_runs` (finns ej; idag bara `cron.log`) (Fas 4). |
| 4.3 Streaks/rekord | 🔴 | Kräver `outcomes` (korrekt-signal-def) → Fas 4 efter 2b. |
| 4.4 Coin flip-bootstrap | 🔴 | Ny, självständig, billig (Fas 4). |

### 0.4 Roadmap (detaljerad, fas för fas)

Varje fas listar **status · vad · var · deploy-behov**. "Deploy-behov" är nyckeln: bara Fas 2b (och
delar av 3/4) rör servern med SQL/cron — resten är frontend eller ren backend-rebuild.

| Fas | Status | Innehåll | Deploy-behov |
|---|---|---|---|
| **0 Fundament** | ✅ KLAR + DEPLOYAD | `/statistik`-route (4 flikar) + nav + brygg-kort. | Frontend. |
| **1 Billiga vinster** | ✅ KLAR + DEPLOYAD | §2.4 affärsstat, §2.2 underwater, §2.6 land-snapshot, §4.1 big numbers. Klientberäknat, rika tooltips. | Frontend. |
| **2a Motor-hit rate** | ✅ BYGGD, EJ DEPLOYAD | §1.1 hit rate + Wilson-CI + n (on-the-fly) → `/engines→Analys`. | **Backend-rebuild** (ingen SQL/cron). Rök-test `/api/engines/hitrate?horizon=20` efter deploy. |
| **2b steg 1 — basrat** | ✅ DEPLOYAD + validerad 2026-07-22 | Nattbatch `compute_stats.py` fyller `signal_outcomes`(rec_id,horizon,forward_return) + basrat per horisont → `stats_daily`. `/hitrate` ger **basrat + edge** (`hit_rate−basrat`). ⏳ Data mognar; basrat/edge syns fr.o.m. 2026-07-23 (TA först). | Gjort: backend-rebuild (tabeller auto-skapade) + cron-rad 00:30 + validerad. |
| **2b steg 2 — kalibrering/Brier/deciler** | ✅ BYGGD + DEPLOYAD (live-verifierad 2026-07-22) | §1.2 kalibreringskurva + ECE, §1.3 Brier/BSS/AUC (rullande), §1.4 decilstaplar + Spearman — 3 read-endpoints (`/calibration`, `/brier`, `/deciles`) + 3 ML-paneler (`Calibration/Brier/Deciles Panel.tsx`) på `/engines→Analys`. `compute_stats.py` materialiserar nu ML:s hela universum (probability-rader). | Gjort: hängde med i steg 1:s backend-rebuild → alla 3 endpoints HTTP 200 live. ⏳ Tomt tills ML-facit mognar (~08-06). |
| **4 §3 Marknaden (hela)** | ✅ BYGGD, EJ DEPLOYAD | §3.1 breadth, §3.3 Nordenkampen, §3.4 52v high/low, §3.5 säsong (kuriosa), §3.2 sektor-treemap. Ny route `statistics.py` (5 endpoints) + 5 paneler + `LineChartSvg`. Facit-oberoende. | **Backend-rebuild + frontend** (ingen tabell/cron). Rök-test alla 5 `/api/statistics/*`. |
| **3 §1.5 motorkorrelation/konsensus** | ✅ BYGGD, EJ DEPLOYAD | Spearman-heatmap (`/correlation`, facit-oberoende) + konsensus-buckets (`/consensus`, hit rate mognar med facit) → `CorrelationPanel`/`ConsensusPanel` på `/engines→Analys`. | **Backend-rebuild + frontend** (ingen tabell/cron). Rök-test `/api/engines/correlation` + `/consensus?horizon=20`. |
| **3 §2.5 beslutsalfa** | ✅ BYGGD, EJ DEPLOYAD | **Kronjuvelen:** `shadow_daily` (auto-skapad tabell) + `compute_shadow_portfolio` i BEFINTLIGA nattbatchen (ingen ny cron) + `/decision-alpha` + `DecisionAlphaPanel` på `/statistik→Affärer`. Regler spikade. | **Backend-rebuild** (tabell auto-skapas via `create_all`) + engångskörning `compute_stats` + frontend. INGEN ny cron/manuell tabell. |
| **4 kuriosa (rest)** | 🔜 | §4.2 batchtider (`job_runs`), §4.3 streaks, §4.4 coin-flip (`/statistik→System`). | Nya read-endpoints (backend-rebuild); §4.2 kräver `job_runs`-tabell + loggning. |
| **Lös-hängande** | 🔜 | §2.3 rullande Sharpe/Sortino (klientberäkningsbart ur TWR — kan tas när som). | Frontend. |
| **⛔ Utanför 1.4** | ⛔ | §1.6 regimband + regimvarianterna i §1.4/1.5. | Kräver egen regimmotor (eget projekt). |

**Beroendekedja:** 2b är grinden till nästan hela §1 (basrat/kalibrering/Brier/deciler) OCH §4.3 streaks —
allt som behöver materialiserat facit. §3 (marknad) och §2.3/§2.5 hänger inte på 2b och kan tas oberoende.

### 0.5 Leveranslogg

| Datum | Leverans | Filer (urval) | Verifiering |
|---|---|---|---|
| 2026-07-21 | **Planering + spec-städning.** 3 faktafel rättade (regim "finns" = falskt, `signals_history` dubblerar `recommendations`, "Statistik"-namnkrock). Fasplan + datamodell + implementationsavsnitt. | `push.md` (denna fil) | — |
| 2026-07-21 | **Fas 0 — fundament.** Route `/statistik` (GlobalSubmenu Portfölj/Affärer/Marknaden/System, `?tab=`), sidebar + mobilnav, brygg-kort båda hållen. | `src/app/statistik/page.tsx`, `Sidebar.tsx`, `MobileMoreMenu.tsx`, `portfolio/components/Statistics.tsx`, `EngineStats.tsx` | `tsc` grönt. **DEPLOYAD.** |
| 2026-07-21 | **Fas 1 — billiga vinster.** §2.4 affärsstatistik + R-histogram, §2.2 underwater+recovery, §2.6 land-snapshot, §4.1 big numbers. Rika tooltips. Ny lib `tradeStats.ts` (per-affär GAV, ej dubblerad FIFO), `drawdownStats` i `portfolioRisk.ts`. | `statistik/components/{AffarsStatistik,RHistogram,DrawdownPanel,AllocationSnapshot,BigNumbers,StatTip}.tsx`, `lib/tradeStats.ts`, `lib/portfolioRisk.ts` | `tsc` grönt. **DEPLOYAD.** |
| 2026-07-21 | **Fas 2a — motor-hit rate (lätt interim).** Endpoint `/api/engines/hitrate` (Wilson-CI, on-the-fly, speglar beprövad `/{engine}/stats`-SQL). Flik **Analys** på `/engines` (CI-band, horisont 20/40/60, "bygger facit"). `StatTip` flyttad till delad `ui/`. | `backend/app/api/routes/engines.py`, `engines/components/EngineAnalys.tsx`, `engines/page.tsx`, `components/ui/StatTip.tsx` | `py_compile` + Wilson-mattest + `sqlglot`-parse + `tsc`. ✅ **DEPLOYAD & verifierad live 2026-07-22** (efter backend-restart; 404 dessförinnan = ej omstartad). |
| 2026-07-22 | **Fix — undermeny-glassmorphism** (`/statistik` + `/signaler`): submeny flyttad in i `PageHeader` (den sticky blur:en täcker bara sina barn). | `statistik/page.tsx`, `signaler/page.tsx` | `tsc`. |
| 2026-07-22 | **Fas 2b steg 1 — basrat + edge.** Nattbatch `compute_stats.py` fyller `signal_outcomes` + basrat→`stats_daily`; `/hitrate` returnerar `base_rate`+`edge`; Analys visar edge + basrat-referenslinje. Tabeller auto-skapas via `create_all`. | `scripts/compute_stats.py`, `models.py`, `engines.py`, `admin.py`, `EngineAnalys.tsx`, `docs/CREATE_stats_tables.sql` | `py_compile` + `sqlglot`-parse (4 block) + `tsc`. ✅ **DEPLOYAD + CRON + VALIDERAD live 2026-07-22** (SQL körde rent mot Postgres, tabeller skapade, `/hitrate` 200; 0 mogna signaler ännu → basrat fr.o.m. 2026-07-23). |
| 2026-07-22 | **DEPLOY-VERIFIERING (allt Fas 3+4 live).** Agent SSH + API-svep: alla 8 nya endpoints HTTP 200; facit-oberoende data ÄKTA & rimlig (breadth ~44 %, Norden NO +4,4 %/SE −2,3 %, säsong 28 år, 20 sektorer, motorkorrelation nära 0 = oberoende). **Körde saknad engångskörning `compute_stats`** → `shadow_daily` fylld (330 skugg-positioner → 7 dgr, TWR 0.9864), `/decision-alpha` state=ready. `signal_outcomes`/basrat fortsatt 0 (facit mognar 2026-07-23, bekräftat). **1 bugg hittad+fixad:** `/consensus` `hit_rate` gav `0.0` istf `null` för omogna baskets (LEFT JOIN + `CASE WHEN NULL>0 → ELSE 0`) → la till `FILTER (WHERE o.fwd IS NOT NULL)`. **Kräver backend-rebuild.** INSIDER saknas i korrelationsmatrisen = korrekt (bara ~2 dgr historik, < 5-dagarströskeln). | `engines.py` (consensus-fix) | Live-curl + SSH psql + `py_compile`/`sqlglot`. |
| 2026-07-22 | **Fas 3 §2.5 — beslutsalfa (kronjuvelen).** Ny ORM `ShadowDaily`(date UNIQUE, twr bas 1.0) — **auto-skapas** av `create_all` (DDL-backup i `CREATE_stats_tables.sql`, → 19 tabeller). `compute_stats.py::compute_shadow_portfolio` (egen transaktion i befintliga nattbatchen, **ingen ny cron**): mekanisk `SIGNAL_SCORE_V1`-skugga — köp vid BUY, sälj vid SELL/60d-tidsstopp, en position per BUY-svit, likaviktat medel av öppna positioners dagsavkastning, 0,6 % rundtur (halva entry/exit), TWR Π(1+r). **Regler SPIKADE** (push.md §2.5-fallgropen). `GET /api/statistics/decision-alpha` läser `shadow_daily`; `DecisionAlphaPanel.tsx` (`/statistik→Affärer`) hämtar även `/api/portfolio/history`, rebasar båda till 100 vid första gemensamma datum → `beslutsalfa = faktisk − skugga` (två linjer + kumulativ alfa, horisont YTD/1Y/3Y). | `models.py`, `CREATE_stats_tables.sql`, `scripts/compute_stats.py`, `statistics.py`, `statistik/components/DecisionAlphaPanel.tsx`, `statistik/page.tsx` | `py_compile` + `sqlglot` (4 nya block) + **offline sim-test** (rising→TWR 1.034, tidsstopp=60d, kostnader) + `tsc` grönt. ⏳ **EJ DEPLOYAD** — backend-rebuild (tabell auto-skapas) + engångskörning `compute_stats` + frontend; skuggan mognar med consensus-historik+facit. |
| 2026-07-22 | **Fas 3 §1.5 — motorkorrelation + konsensus.** Två nya endpoints i `engines.py`: `/correlation` (parvis Spearman-rho mellan motorernas dagliga cross-sectionella signal_score-rankningar, snittad över ~120 dgr; ≥20 gemensamma tickers/dag, ≥5 dagar/cell; återanvänder `_spearman`; **facit-oberoende**) + `/consensus?horizon=` (köp grupperade efter antal samstämmiga motorer per ticker/datum; `n_baskets` nu, hit rate/avkastning via `signal_outcomes`-join när facit mognat). Frontend: `CorrelationPanel` (CSS-grid-heatmap, färg ∝ \|rho\|) + `ConsensusPanel` (staplar, delar horisontväljaren), ny sektion "Samspel mellan motorer" i `EngineAnalys.tsx`. | `engines.py`, `engines/components/{CorrelationPanel,ConsensusPanel,EngineAnalys}.tsx` | `py_compile` + `sqlglot`-parse (2 block, Postgres) + `tsc` grönt. ⏳ **EJ DEPLOYAD** — backend-rebuild + frontend; konsensus-hit rate mognar med facit. |
| 2026-07-22 | **Fas 4 §3-B/C — Marknaden komplett.** Tre nya endpoints i `statistics.py`: `/highs-lows` (§3.4 52v highs/lows via rullande 252d MAX/MIN, net + kumulativ net, ≥200d-historik-krav), `/seasonality` (§3.5 equal-weight-index medel per månad/veckodag + std/√n; daglig avk. i SQL, månads-/veckodagsgruppering i Python), `/sector-treemap` (§3.2 Σmcap per sektor + kapitalviktad avk. över `window` handelsdagar). Frontend: `HighsLowsPanel` (linje+staplar), `SeasonalityPanel` (divergerande staplar + felstaplar, kuriosa-varning), `SectorTreemap` (squarified-algoritm i frontend, Idag/Vecka-toggle). Marknaden-fliken nu 5 paneler; `StatistikComingSoon` ej längre använd. | `statistics.py`, `statistik/components/{HighsLowsPanel,SeasonalityPanel,SectorTreemap}.tsx`, `statistik/page.tsx` | `py_compile` + `sqlglot`-parse (5 block totalt, Postgres) + `tsc` grönt. ⏳ **EJ DEPLOYAD** — backend-rebuild + frontend; rök-test alla endpoints på VPS. |
| 2026-07-22 | **Fas 4 §3-A — Marknaden: Nordenkampen + breadth.** Ny route `app/api/routes/statistics.py` (`/api/statistics`, monterad i `main.py` bakom x-api-key): `GET /breadth` (§3.1 andel > MA20/50/200 över tid via named-window-SQL, MA-buffert +320d) + `GET /nordic-race` (§3.3 likaviktat landindex SE/NO/DK/FI→100 vid årsskiftet, dagsavk. clip ±50 %, kumuleras i Python). Frontend: `NordicRacePanel`+`BreadthPanel`+delad `LineChartSvg` (egen SVG) ersätter Marknaden-`StatistikComingSoon`. Land ur ticker-suffix, universum `is_active`. **Facit-oberoende** (ren prishistorik). | `backend/app/api/routes/statistics.py`, `main.py`, `statistik/components/{NordicRacePanel,BreadthPanel,LineChartSvg}.tsx`, `statistik/page.tsx` | `py_compile` + `sqlglot`-parse (2 block, Postgres) + `tsc` grönt. ⏳ **EJ DEPLOYAD** — backend-rebuild + frontend; rök-test båda endpoints på VPS (ingen lokal Postgres). |
| 2026-07-22 | **Rättelse — Fas 2b steg 2 var redan DEPLOYAD.** Agent-verifierade mot prod: `/api/engines/{calibration,brier,deciles}` svarar HTTP 200 live (koden hängde med i steg 1:s backend-rebuild). §0.1/§0.3/§0.4/leveransloggen sa felaktigt "EJ DEPLOYAD" → rättat. Flaskhals för §1 = facit-mognad, inte kod. | `push.md` | Live-curl mot prod (agent-konto). |
| 2026-07-22 | **Fas 2b steg 2 — kalibrering/Brier/deciler.** 3 read-endpoints (`/calibration` §1.2 reliability+ECE, `/brier` §1.3 rullande Brier/BSS/AUC, `/deciles` §1.4 cross-sectional decilstaplar+Spearman) ovanpå `signal_outcomes`. `compute_stats.py::FILL_OUTCOMES_SQL` utökad: materialiserar nu ML:s hela universum (rader med `probability`, ej bara BUY) så kalibreringen inte skevar på topp-signaler. 3 ML-paneler i Analys (egna SVG). AUC/Spearman i ren Python (ingen sklearn/scipy i web-processen). | `engines.py`, `scripts/compute_stats.py`, `engines/components/{CalibrationPanel,BrierPanel,DecilesPanel,EngineAnalys}.tsx` | `py_compile` + `sqlglot`-parse (2 block) + offline-test (AUC/Spearman/decil) + `tsc`. ⏳ **EJ DEPLOYAD** — backend-rebuild + engångskörning av `compute_stats.py`; facit mognar (ML ~2026-08-06 h=20). |

### Datakvalitet — kritisk förutsättning (åtgärdad 2026-07-21)

**Hela EOD-pris­historiken var daterad −1 dag — nu om-hämtad och korrekt.** Detta är en
**förutsättning för att i stort sett HELA denna spec ska ge sanna siffror**: §1.1 hit rate,
§1.4 decil-forward-returns, §1.3 Brier/AUC, §2.1 TWR-equity, §3.1 breadth, §3.4 52v high/low,
§3.5 säsong och §4.3 streaks räknar alla forward returns / MA / extrempunkter direkt på
`historical_prices` (och point-in-time-principen högst upp förutsätter rätt datum). Med skiftet
hade varje `r_h = P(t+h)/P(t)−1`, varje MA200-korsning och varje säsongssnitt legat en handelsdag
fel, plus ~1M falska helg-rader i serien.

- **Vad:** `historical_prices` (+ 7 EU-tickers i `global_historical_prices`) hade varje rad
  stämplad en kalenderdag för tidigt (verklig tis→mån, mån→sön) → veckodagshistogram Sön–Tor
  ≈ 1M/dag. Orsak: tidszons-naiv UTC-index i en äldre yfinance/pandas vid backfill-tillfället.
- **Åtgärd:** om-backfillad på VPS med nuvarande (korrekta) version — `historical_prices`
  ombyggd (`fetch_history.py` TRUNCATE+max, 1566/1569 aktiva; 3 utgångna BTA/BTU utan yfinance-data
  utelämnade) + riktad om-hämtning av de 7 skiftade EU-tickerna i global. Verifierat: 0 helg-rader
  i båda tabellerna, datum matchar verkliga kurser, portföljgrafens 1M utan falskt hopp. Backup:
  `historical_prices_bak_20260721` + `global_historical_prices_bak_20260721` (droppa efter UI-verifiering).
- **Framåt skyddat:** HELG-VAKT (`d.weekday()<5`) i `fetch_daily/history/global_daily/indices.py`.
- **Kvarstår att tänka på för 1.4:** `recommendations`-radernas historiska `date` sattes av motorer
  som läste den skiftade datan → gamla signaldatum kan vara ±1 dag. Point-in-time-panelerna (§1)
  bör antingen (a) räkna forward returns från den nu KORREKTA prisserien oavsett, eller (b) köra om
  motorernas historik om exakt signaldatum är kritiskt. Nya signaler (nattkörning) är redan korrekta.

### 0.6 Startguide — Fas 2b steg 2 (för ny utvecklare, kall start)

> **Målgrupp:** en utvecklare som *aldrig sett projektet*. Läs detta + panel-specarna §1.2–1.4 +
> de refererade nyckelfilerna, så kan du börja direkt. Steg 1 (basrat) är klart och deployat —
> du bygger **ovanpå** den redan materialiserade `signal_outcomes`-tabellen.

**Vad ska byggas:** tre motor-analys-vyer på `/engines→Analys` + backend-endpoints:
§1.2 **kalibreringskurva** (+ ECE), §1.3 **Brier/AUC** (rullande tidsserie), §1.4 **decilstaplar**.

**Datagrunden (redan klar — bygg PÅ den):**
- **`signal_outcomes`** (`backend/app/db/models.py`; tabell 17 i `backend/docs/3_Database_Schema.md`):
  en rad per (signal, horisont) med `forward_return` (rå `r_h`). Fylls nattligen av
  `scripts/compute_stats.py` (00:30, Tis–Lör).
- Join `signal_outcomes.rec_id = recommendations.id` ger signalens `date`, `engine_name`, `metrics` (JSONB).
- **ML:s kalibrerade sannolikhet:** `recommendations.metrics->>'probability'` (0–1). **ENDAST
  `ML_HGB_V3_ABOVE_MEDIAN` har en äkta kalibrerad p** — TA/F-score/insider har bara
  `metrics->>'signal_score'` (0–100, riktad, ej sannolikhet). → §1.2 kalibrering + §1.3 Brier gäller **ML**.
  §1.4 deciler kan rankas på `signal_score` (alla motorer) eller `probability` (ML).
- **Utfall y:** spika `y = 1 om forward_return > 0` (fast tröskel — bestäm EN gång, ändra aldrig, jfr
  §1.1-principen). Applicera split-spärren `forward_return BETWEEN -0.95 AND 3.0` vid läsning (samma som `/hitrate`).

**⚠️ FÖRSTA UPPGIFTEN — utöka outcomes-materialiseringen:** `compute_stats.py::FILL_OUTCOMES_SQL` fyller
i dag `signal_outcomes` **ENDAST för `signal_type='BUY'`**. Kalibrering (§1.2) och deciler (§1.4) behöver
p/utfall för **hela ML:s dagliga output** (även NEUTRAL — ML skriver `probability` på hela universumet varje
dag, mestadels NEUTRAL). Utöka `FILL_OUTCOMES_SQL` att materialisera outcomes även för icke-BUY-rader med en
`probability` (eller för alla `recommendations`-rader), annars beräknas kalibreringen skevt på bara topp-signalerna.

**Var koden bor (följ befintliga mönster — kopiera, uppfinn inte):**
- **Backend:** nya read-endpoints i `backend/app/api/routes/engines.py`, bredvid `get_engines_hitrate`
  (kopiera dess struktur: `horizon`-param, läs `signal_outcomes` JOIN `recommendations`, aggregera i Python).
  T.ex. `GET /api/engines/calibration?horizon=`, `/brier?horizon=`, `/deciles?horizon=`. Börja **on-the-fly**;
  materialisera i `stats_daily` via `compute_stats.py` bara om det blir för långsamt.
- **Frontend:** nya paneler i `EngineAnalys.tsx` (eller utbrutna i `src/app/engines/components/`). §1.2 =
  reliability-scatter mot y=x, §1.3 = linjediagram över tid, §1.4 = stapeldiagram decil 1–10 (egen SVG som
  `RHistogram.tsx`/`DrawdownPanel.tsx`, eller `lightweight-charts`). **Rika tooltips via
  `src/components/ui/StatTip.tsx`** (delad `StatTip`/`TooltipCard`). Följ UI_RULES (CSS-variabler,
  `.text-*`-klasser, `data-signal-*`-taggar).

**Verifiera (som resten av projektet):** backend `python -m py_compile <fil>` + `sqlglot`-parse av ny SQL
(**går ej att köra mot Postgres lokalt** — ingen lokal DB; rök-testa på VPS efter deploy, se insider-lärdomen).
Frontend `npx tsc --noEmit`. Uppdatera docs (`2_API_Endpoints.md`, `3_Database_Schema.md` om `compute_stats`
utökas, `src/docs/{1_Översikt,Changelog}.md`, samt avsnitt 0 här).

**Deploy:** backend-rebuild (`git pull` + `docker compose up -d --build backend` på VPS `root@65.109.143.130`,
backend-mapp `/root/apps/signal/backend`). Om `compute_stats.py` utökas: kör en gång via
`POST /api/admin/sync/compute_stats`. **Ingen ny tabell/cron** (`signal_outcomes`/`stats_daily` + cron finns redan).

**Facit-mognad (påverkar vad du ser):** `signal_outcomes` fylls först när signaler mognat ≈ horisont × 1,4
kalenderdagar. Per 2026-07-22: 0 mogna; TA får facit 2026-07-23, ML ~2026-08-06 (h=20). Bygg mot gles/tom data
med graceful "bygger facit"-lägen (som `/hitrate` redan gör).

**Läs dessa filer först:** `backend/app/api/routes/engines.py` (`get_engines_hitrate` = mönstret) ·
`backend/scripts/compute_stats.py` (batchen) · `backend/app/db/models.py` (`SignalOutcome`/`StatsDaily`) ·
`signal_frontend/src/app/engines/components/EngineAnalys.tsx` (UI-mönstret) · `backend/docs/3_Database_Schema.md`
(tabell 17–18). Denna fils §1.2–1.4 = de statistiska formlerna.

---

## 1. Motorernas prestanda

### 1.1 Hit rate per motor

> ✅ **Byggt & deployat (Fas 2a + 2b steg 1, 2026-07-21/22).** `GET /api/engines/hitrate` + `/engines→Analys`:
> hit rate + **Wilson-CI** + n + **basrat + edge** (`hit_rate−basrat`, huvudsiffran) per motor.
> **Horisonter i produktion: 20/40/60** handelsdagar (systemets etablerade band, matchar Shadow +
> harnessen) — inte spec-textens {5,20,60}. Kvar: dedup av överlappande fönster (mindre, ej gjort).
> ⏳ Basrat/edge fylls av nattbatchen när facit mognat (~28 dgr; TA först 2026-07-23).

**Vad:** Andel köpsignaler med positiv forward return, per horisont h ∈ {5, 20, 60}.

**Beräkning:**

```
hit_rate(h) = antal signaler med r_h > 0 / totalt antal signaler
```

Konfidensintervall med **Wilson score** (bättre än normalapproximation vid små n):

```
centrum = (p̂ + z²/2n) / (1 + z²/n)
bredd   = z/(1 + z²/n) · sqrt( p̂(1−p̂)/n + z²/4n² )
CI      = centrum ± bredd,   z = 1.96 för 95 %
```

**Viktigt:** visa alltid bredvid **basraten** — andelen av *alla* aktier i universumet
som var plus samma period. En hit rate på 62 % i en marknad där 65 % av allt steg är
negativ edge. Rapportera gärna `hit_rate − basrat` som huvudsiffra.

**Fallgrop:** signaler med överlappande fönster (två signaler på samma aktie inom h
dagar) är korrelerade — effektivt n är lägre än nominellt n. Enklast: dedupa till max
en signal per aktie per fönster.

### 1.2 Kalibreringskurva (reliability diagram)

> ✅ **Byggt (Fas 2b steg 2, 2026-07-22), EJ DEPLOYAD.** `GET /api/engines/calibration?horizon=` +
> `CalibrationPanel.tsx` på `/engines→Analys`: reliability-scatter mot y=x, punktstorlek = n_b, ECE i headern.
> Kvantilhinkar (lika många obs/hink). Gäller ML (enda motorn med kalibrerad `probability`). ⏳ facit mognar.

**Vad:** När ML-motorn säger 70 % — händer det 70 % av gångerna?

**Beräkning:**

```
1. Ta alla (p_i, y_i) där p = predikterad sannolikhet, y = utfall (1 om r_h > tröskel)
2. Binna p i deciler (eller kvantiler så varje bin får lika många obs)
3. Per bin b: conf_b = medel(p_i), acc_b = medel(y_i)
4. Plotta acc_b mot conf_b, referenslinje y = x
```

Sammanfatta med **Expected Calibration Error**:

```
ECE = Σ_b (n_b / N) · |acc_b − conf_b|
```

**Visualisering:** scatter/linje mot 45-graderslinjen, punktstorlek = n_b.
Perfekt kalibrering = punkterna ligger på linjen.

### 1.3 Rullande Brier score och AUC

> ✅ **Byggt (Fas 2b steg 2, 2026-07-22), EJ DEPLOYAD.** `GET /api/engines/brier?horizon=&window=` +
> `BrierPanel.tsx`: Brier + BSS (mot klimatologin ȳ) + AUC, som nuvärden + rullande tidsserie (default
> 252 kalenderdagar). AUC (Mann–Whitney, rank-baserad) + Spearman i **ren Python** — ingen sklearn/scipy
> i web-processen. Fönster med < 40 obs hoppas över. Gäller ML.

**Vad:** Blir motorn bättre eller sämre över tid?

**Beräkning:**

```
Brier = (1/N) Σ (p_i − y_i)²          # lägre = bättre, 0.25 = värdelös vid p̄=0.5
```

Normalisera mot klimatologisk baseline (alltid gissa basraten):

```
BS_ref = p̄ · (1 − p̄)
BSS    = 1 − Brier / BS_ref            # > 0 betyder bättre än att gissa basraten
```

AUC via Mann–Whitney: sannolikheten att en slumpad positiv får högre p än en slumpad
negativ. `sklearn.metrics.roc_auc_score` räcker. Beräkna båda i **rullande
252-dagarsfönster** och plotta som tidsserie — det är trendlinjen som är intressant,
inte nivån en enskild dag.

### 1.4 Forward returns per sannolikhetsdecil

> ✅ **Grundvariant byggd (Fas 2b steg 2, 2026-07-22), EJ DEPLOYAD.** `GET /api/engines/deciles?horizon=` +
> `DecilesPanel.tsx`: cross-sectional decilrankning per dag på ML:s `probability` (D1 lägst → D10 högst),
> poolad medel-forward-return per decil, divergerande staplar + Spearman rho (+ approx p). Regim-split ⛔.

**Vad:** Ger högre signalstyrka faktiskt högre avkastning?

**Beräkning:**

```
1. Per dag: ranka universumets p-värden i deciler (cross-sectional, inte global rank)
2. Per decil: medel av r_20 över alla dagar
3. Stapeldiagram decil 1–10
```

Monotont stigande staplar = motorn rankar rätt. Testa formellt med Spearman-korrelation
mellan decil och avkastning; rapportera rho + p-värde. ⛔ **BLOCKERAD (utanför 1.4):** samma
diagram uppdelat per regim (calm/neutral/stress) — kräver en regimmotor som inte finns
(HMM avförd 2026-07-13). Grundvarianten (utan regim-split) byggs i Fas 2/3.

### 1.5 Motorkorrelation och konsensus

**Vad:** Hur ofta håller F-score, TA och ML med varandra — och vad är konsensus värd?

**Beräkning:**

```
1. Normalisera varje motors dagliga output till cross-sectional rank (0–1) per dag
2. Spearman-korrelation parvis mellan motorernas rankvektorer, medel över dagar
3. Heatmap 3×3 (⛔ 4×4 "med regimmotorn som gate" är BLOCKERAD — ingen regimmotor finns)
```
*(Motorerna är i praktiken 4 st idag: TA/F-Score/ML/Insider — heatmapen blir 4×4 av dessa,
inte p.g.a. regim.)*

Konsensusanalys:

```
Gruppera signaler efter antal motorer som samtidigt säger köp (1, 2, 3)
→ hit rate och medel-r_20 per grupp
```

Låg korrelation mellan motorer + stigande avkastning med konsensus är exakt vad man
vill se — det är beviset på att motorerna tillför oberoende information.

### 1.6 Regimband ⛔ BLOCKERAD (utanför 1.4)

> ⛔ **Kräver en regimmotor som inte finns.** Det finns ingen `regime`-tabell och ingen
> HMM-motor — HMM avfördes uttryckligen som egen motor 2026-07-13 (regimproxies som breadth/
> räntor/rabattaggregat ska testas som features först, se `Motor_Dev&Roadmap.md`). Hela denna
> panel ligger utanför update 1.4. Behåll specen som referens för när/om regimmotorn byggs.

**Vad:** Indexgraf med bakgrund färgad efter HMM-regim. Snyggaste panelen på sidan.

**Beräkning:** läser en `regime`-tabell (`date, p_calm, p_neutral, p_stress, state`) som
**inte existerar ännu** — måste byggas av en framtida regimmotor.

```
Per dag: färg = argmax(p_calm, p_neutral, p_stress), alpha = maxsannolikheten
Rita som bakgrundsband bakom likaviktade indexet
```

Komplettera med en tabell **per regim**: medelavkastning, annualiserad vol och varje
motors hit rate. Det svarar på "vilken motor ska jag lita på i vilken regim?" och blir
underlag för regimviktningen i motorerna.

---

## 2. Portfölj

### 2.1 Equity curve vs benchmark

**Vad:** Portföljvärde mot OMXS30GI och ditt eget likaviktade nordenindex.

**Beräkning:** använd **tidsviktad avkastning (TWR)** så insättningar/uttag inte
förvränger kurvan:

```
r_t  = (V_t − F_t) / V_{t−1} − 1     # F_t = nettoinsättning dag t
TWR  = Π (1 + r_t) − 1
```

Indexera alla serier till 100 vid gemensam startdag. Använd GI-varianten (inklusive
utdelningar) av benchmark — annars jämför du äpplen med päron.

### 2.2 Drawdown och underwater plot

> ✅ **Byggt (Fas 1, 2026-07-21).** `DrawdownPanel.tsx` på `/statistik→Portfölj`: underwater-SVG
> (YTD) + max drawdown + **längsta recovery-tid** + "under vatten nu". Klientberäknat ur TWR
> (`portfolioRisk.ts::drawdownStats`).

**Beräkning:**

```
peak_t = max(V_s för s ≤ t)
DD_t   = V_t / peak_t − 1            # alltid ≤ 0
MaxDD  = min(DD_t)
```

Underwater plot = DD_t som area-chart under nollinjen. Visa även **längsta
recovery-tid** (dagar från peak till ny peak) — den siffran känns mer i magen än
procenttalet.

### 2.3 Rullande Sharpe och Sortino

**Beräkning:** på dagliga TWR-avkastningar, fönster 252 dagar:

```
Sharpe  = (mean(r − rf) / std(r − rf)) · sqrt(252)
DownDev = sqrt( mean( min(r − rf, 0)² ) ) · sqrt(252)
Sortino = (mean(r − rf) · 252) / DownDev
```

rf = Riksbankens styrränta / 252. Visa ingenting förrän minst 126 dagars historik
finns; annualiserad Sharpe på sex veckors data är numerologi.

### 2.4 Affärsstatistik

> ✅ **Byggt (Fas 1, 2026-07-21).** `AffarsStatistik.tsx` på `/statistik→Affärer`: win rate,
> profit factor, expectancy, payoff-kvot, snittvinst/-förlust, realiserat netto, innehavstid,
> **R-histogram** (egen SVG) och topplista bästa/sämsta. Ren klientberäkning ur `/api/transactions/`
> via `lib/tradeStats.ts` (per-stängd-affär GAV-matchning, delar metod med `portfolioReturns.ts`).

**Vad:** Fördelning och nyckeltal per **stängd** affär.

**Beräkning:**

```
R_i           = (exit − entry) / entry − courtage_andel
win_rate      = andel R_i > 0
avg_win       = medel(R_i | R_i > 0);   avg_loss = |medel(R_i | R_i < 0)|
profit_factor = Σ vinster / |Σ förluster|
expectancy    = win_rate · avg_win − (1 − win_rate) · avg_loss
payoff_ratio  = avg_win / avg_loss
```

**Visualisering:** histogram över R_i (symlog-x om svansarna är tunga), nyckeltalen
som kort ovanför, snittinnehavstid som egen fördelning. Plus topplista: bästa/sämsta
affärer all-time med ticker, datum och R.

### 2.5 Beslutsalfa — jämför dig mot dina egna motorer

**Vad:** Tillför dina manuella beslut värde ovanpå signalerna, eller förstör de det?
Sidans ärligaste och viktigaste panel.

**Beräkning:**

```
1. Bygg en skuggportfölj som mekaniskt följer signalerna:
   - entry: köp vid köpsignal (likaviktad allokering, samma kapitalbas som din)
   - exit: säljsignal eller tidsstopp (t.ex. 60 dagar), samma courtagemodell
2. Beräkna TWR för skuggportföljen dagligen (samma formel som 2.1)
3. beslutsalfa_t = TWR_faktisk(0..t) − TWR_skugga(0..t)
```

Plotta kumulativt över tid. Över noll = dina avvikelser från motorerna tillför värde.
Kan förfinas med attribution: **selektion** (du tog vissa signaler men inte andra) vs
**timing** (du agerade senare/tidigare än signalen). Börja enkelt med totalsiffran.

**Fallgrop:** skuggportföljens regler måste spikas *innan* du tittar på utfallet,
annars kurvpassar du benchmarken tills du ser bra ut.

### 2.6 Exponering över tid

> 🟡 **Delvis byggt (Fas 1, 2026-07-21).** `AllocationSnapshot.tsx` på `/statistik→Portfölj` visar
> **nuvarande fördelning per land + kassa** (ögonblicksbild), klientsidan ur `/api/portfolio/`
> (land ur ticker-suffix). **Saknas:** "över tid" (stackad area) + **per sektor** — `/api/portfolio/`
> returnerar inte sektor, och över-tid kräver per-position-historik → backend-jobb i en senare fas.

**Beräkning:** per dag, andel av portföljvärdet per sektor, land och kassa.
Stackad area-chart. Kräver bara innehavshistorik × sektormappning du redan har.

---

## 3. Marknaden

### 3.1 Breadth

**Beräkning:**

```
breadth_MA200(t) = andel aktier med close_t > MA200_t     # samma för MA50, MA20
```

Linjediagram med nivåerna 20/50/80 % markerade. Dubblar som feature till regimmotorn.
Vill du vara fancy: McClellan-oscillator = EMA19 − EMA39 av `(adv − dec)/(adv + dec)`
där adv/dec = antal aktier upp/ner per dag.

### 3.2 Sektor-treemap

**Beräkning:** rektangelstorlek = sektorns totala börsvärde (eller antal bolag om du
saknar börsvärden), färg = kapitalviktad dagsavkastning på divergerande färgskala med
mittpunkt 0. En vy för idag, en för rullande vecka.

### 3.3 Nordenkampen

**Beräkning:** likaviktat landindex ur egen data:

```
index_land(t) = index_land(t−1) · (1 + medel(r_i,t för alla aktier i landet))
```

Alla fyra indexerade till 100 vid årsskiftet, ett linjediagram. Enkel, snygg, och
folk älskar den här typen av jämförelse.

### 3.4 52-veckors highs och lows

**Beräkning:**

```
high_t = antal aktier med close_t = max(close, 252 dagar)
low_t  = antal aktier med close_t = min(close, 252 dagar)
net_t  = high_t − low_t              # plotta även kumulativ Σ net_t
```

Kumulativa net highs-linjen är en klassisk breadth-indikator — divergens mot index
brukar föregå trendskiften.

### 3.5 Säsongsmönster

**Beräkning:** medelavkastning för likaviktade indexet per kalendermånad över alla år
i databasen, felstaplar = std/sqrt(antal år). Samma per veckodag.

**Ärlighetsvarning i UI:** med 12 månader × 5 veckodagar testar du 17 hypoteser —
något blir "signifikant" av ren slump. Märk panelen som kuriosa, inte signal.

---

## 4. System & kuriosa

### 4.1 Big numbers-panel

> 🟡 **Delvis byggt (Fas 1, 2026-07-21).** `BigNumbers.tsx` på `/statistik→System` läser
> `/api/admin/status`: datapunkter, bevakade bolag, valutakurser, senaste kurssynk. **Saknas:**
> genererade signaler (totalt/senaste veckan) + äldsta datapunkt — kräver nya `COUNT(*)`/`MIN(date)`-
> frågor serversidan (adderas när nattbatchen ändå rör backend).

Rena `COUNT(*)`/`MIN(date)`-frågor, cachade nattligen: antal datapunkter, tickers,
genererade signaler totalt och senaste veckan, äldsta datapunkt, beräkningar per natt.
Stora siffror, monospace, mycket luft. Ren dopamin men ger sidan liv.

### 4.2 Batchkörningstider

Logga start/slut per motor per natt till en `job_runs`-tabell → linjediagram med
7-dagars glidande medel. Smygande prestandaförsämring syns här långt innan den blir
ett problem.

### 4.3 Streaks och rekord

```
korrekt signal   = köpsignal med r_20 > 0 (spika definitionen en gång, ändra aldrig)
längsta streak   = max antal konsekutiva korrekta signaler (globalt och per motor)
mest älskade     = ticker med flest köpsignaler senaste 90 dagarna
bästa månad      = max månads-TWR;  personbästa-lista med datum
```

### 4.4 Coin flip-benchmark

**Vad:** Är din avkastning skicklighet eller tur? Bootstrap-svar:

```
1. Simulera N = 10 000 slumpportföljer: samma antal affärer, samma
   innehavstider, samma universum — men slumpade tickers och entrydagar
2. Beräkna varje simulerings totala TWR
3. Din percentil i den fördelningen = panelens siffra
```

Histogram över simuleringarna med din siffra som vertikal linje. "Du slog 94 % av
slumpaporna" är både roligt och statistiskt hederligt — det är ett riktigt
permutationstest.

---

## Prioriteringsförslag

> **Ersatt av fasplanen i avsnitt 0.** Originalförslaget nedan behålls som historik. Det var
> för optimistiskt i Fas 1: kalibreringskurva + hit rate kräver den nya `outcomes`-pipelinen
> OCH ~28 dagars moget facit — de hör till Fas 2/3, inte först. Regimband är helt struket
> (blockerad). Se avsnitt 0 för den gällande fasindelningen.

| Fas (ursprunglig, ej gällande) | Paneler | Motivering |
|-----|---------|------------|
| 1 | Beslutsalfa, kalibreringskurva, hit rate m. basrat | Gör dig till bättre beslutsfattare |
| 2 | Equity/drawdown, affärsstatistik, regimband | Kärnvyer, mestadels befintlig data |
| 3 | Breadth, decilstaplar, konsensus, Nordenkampen | Kontext + underlag för motorutveckling |
| 4 | Treemap, säsong, big numbers, streaks, coin flip | Garnityr och glädje |

## Datamodell (korrigerad)

```
recommendations  (FINNS: date, ticker, engine_name, signal_type, metrics{signal_score,
                  probability, …}) — ÄR redan den immutabla signalhistoriken (ersätter
                  push.md:s tidigare påhittade "signals_history"). Lägg features_version i metrics.
signal_outcomes  (FINNS sedan Fas 2b steg 1: rec_id → recommendations.id, horizon,
                  forward_return, computed_at). Fylls av scripts/compute_stats.py när facit
                  mognat. SEDAN Fas 2b steg 2: materialiserar BUY-rader OCH alla rader med en
                  `probability` (ML:s hela universum) → §1.2/1.4 kalibreras på hela outputen.
stats_daily      (FINNS sedan Fas 2b steg 1: date, scope, metric, value). Nu: basrat per horisont.
                  Framtida hemvist för tunga aggregat (kalibrering/Brier) om on-the-fly blir för långsamt.
job_runs         (NY vid §4.2: job, started_at, finished_at, status)
portfolio_daily  (VALFRI: date, value, net_flow, cash)   # materialisering av det
                  /api/portfolio/history redan räknar; bygg bara vid prestandabehov
shadow_daily     (FINNS sedan Fas 3: date UNIQUE, twr bas 1.0, computed_at). Auto-skapad
                  via create_all. Fylls av compute_stats.py::compute_shadow_portfolio (mekanisk
                  SIGNAL_SCORE_V1-skugga). Läst av /api/statistics/decision-alpha (§2.5 beslutsalfa).
# regime         BORTTAGEN — finns INTE, kräver en regimmotor (utanför 1.4)
```

Nyckelidén: `signal_outcomes` fylls på av nattbatchen när facit finns, och §1-statistiken blir
enkla aggregat över `recommendations × signal_outcomes` — inga tunga beräkningar i frontend.
*(Den byggda tabellen heter `signal_outcomes`, inte `outcomes` — matcha koden.)*

---

## Implementation — endpoints & filplacering (1.4)

**Backend** (respekterar `backend/app/api/routes/`-mönstret + fristående cron-skript):
- Ny route `app/api/routes/statistics.py` (prefix `/api/statistics`) — läser `stats_daily`/
  aggregat för §2–4. Montera i `main.py` (bakom `x-api-key`).
- §1-motorstatistik utökar befintliga `engines.py` (håll motor-data samlad där Shadow redan bor).
- Nattaggregat som skript `scripts/compute_stats.py` (egen manuell cron-rad på VPS, som `fetch_*`/
  `train_ml_engine.py` — datafetch/batch ligger inte i `ENGINES`). Fyller `outcomes` + `stats_daily`.
- Nya ORM-modeller (`outcomes`, `stats_daily`, `job_runs`) i `models.py`; migration-SQL i
  `backend/docs/` (som tidigare `CREATE_*`-filer). `job_runs` fylls av `run_engines.py`/skripten.
- Registrera skriptet i `admin.py` (`script_map`/`/sync`) enligt kritisk regel 2.

**Frontend** (respekterar `src/app/<route>/`-mönstret + `GlobalSubmenu`):
- Ny route `src/app/statistik/` (`page.tsx` tunn `PageHeader`-wrapper). Undermenyflikar via
  `GlobalSubmenu` (läser `?tab=`, som `/signaler`/`/notiser`): **Portfölj · Affärer · Marknaden · System**.
- Paneler i `src/app/statistik/components/`. Återanvänd `PortfolioPerformancePanel` (redan utbruten
  för detta), `BentoBox`/`KpiCard`, `signalColor`/`tint`. Diagram via befintliga chart-wrappers.
- Motor-**Analys**-flik i `src/app/engines/components/` (§1-panelerna).
- Sidebar-rad i `src/components/Sidebar.tsx`. Följ UI_RULES (12 text-klasser, `data-signal-*`-taggar).

**Doc-synk vid bygge:** `2_API_Endpoints.md`, `3_Database_Schema.md`, `1_Arkitektur.md` (backend);
`src/docs/{1_Översikt,4_Routing_and_Components,Changelog}.md` (frontend). Uppdatera avsnitt 0 här per fas.