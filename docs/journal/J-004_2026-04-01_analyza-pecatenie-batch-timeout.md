---
id: J-004
date: 2026-04-01
slug: analyza-pecatenie-batch-timeout
status: draft
author: roman
---

# J-004 — Analýza: Pečatenie, race condition, a prehľad bugov od Milady

## Vstup od Milady (testerka) — kompletný prehľad

### Hlavný problém
> "Dávky 50ks keď sme zapli pečatenie tak sa nafukuje čas a nám to padá."

### Stav testovania

**EESSI**: OK — dávky zbiehajú z oboch priečinkov, pridaná značka CB02 spracovaná OK.

**TEST — DP hromadné dávky**: Pretestované všetky variácie vrátane 50ks dávky. Po retestoch a dôkladnej kontrole pridané nové bugy.

**TEST — JVP hromadné dávky**: Pretestované variácie pre FO/PO, tlačivá `30807484.Rozhodnutie_o_poistnom.sk.jvp` a `30807484.Rozhodnutie_na_penale.sk.jvp`, pobočka BAM. Väčšie dávky 50+ pozastavené do opravy bugov.

**DEV**: Len retesty opravených bugov pre DP a JVP.

### Kompletný zoznam bugov

#### DEV — JVP (3.1.2)
| Bug | Popis | Assignee | Poznámka |
|-----|-------|----------|----------|
| SPISREG-2394 | Nedotiahnuté údaje o spracovateľovi na zázname | Roman | Nuaktiv (p. Pavlenda) |

#### DEV — DP (3.1.2)
| Bug | Popis | Assignee | Poznámka |
|-----|-------|----------|----------|
| SPISREG-1517 | Po spracovaní dávky spisy zostali "V riešení" namiesto "Vybavený", čísla záznamov sa neuložili do DB | Roman | |
| SPISREG-2428 | Nedotiahnuté údaje o spracovateľovi na zázname | Roman | Nuaktiv (p. Pavlenda) |

#### TEST — JVP (3.1.2)
| Bug | Popis | Assignee | Poznámka |
|-----|-------|----------|----------|
| SPISREG-2422 | Nedotiahnuté údaje o spracovateľovi na zázname | Roman | Nuaktiv (p. Pavlenda) |
| SPISREG-2695 | Nový bug | Roman | "Podľa mňa ide o tú istú chybu ako pri DP dávke bug 2688" |

#### TEST — DP (3.1.2)
| Bug | Popis | Assignee | Poznámka |
|-----|-------|----------|----------|
| SPISREG-2423 | Nedotiahnuté údaje o spracovateľovi na zázname | Roman | Nuaktiv (p. Pavlenda) |
| SPISREG-2688 | Nový bug | Roman | |
| SPISREG-2690 | Nový bug — MEGA BUG | Tibor | "Taký som ešte nezadala" |

### Kategorizácia bugov (naša analýza)

Bugy spadajú do 3 oddelených problémov:

1. **Pečatenie + timeout** (F-016): SPISREG-2688, SPISREG-2695, SPISREG-1517
2. **Race condition / duálne spracovanie** (F-017): SPISREG-2690 (MEGA BUG)
3. **Nedotiahnutý spracovateľ / Nuaktiv**: SPISREG-2394, SPISREG-2422, SPISREG-2423, SPISREG-2428 — Roman rieši s p. Pavlendom

## Vstup od Romana (developer)

### Potvrdenie race condition v S312
> "Duálne spracovanie toho istého záznamu výstupných dokumentov kde jeden uspeje a druhý neuspeje a vzájomne si prepíšu výsledok. Scenár 3.1.2. V niektorých prípadoch vznikne aj správny záznam."

### Čaká sa na podklad
- Mail od Nuaktivu — súvisí s problémom spracovateľa (SPISREG-2394 a súvisiace)

## Analýza 1 — F-016: Pečatenie pri 50ks dávkach

### Čo sme zistili

Prečítané a analyzované:
- DNR dokument (R1_1_DNR_DETAILNY_NAVRH_RIESENIA_Projekt_ISREG_2.2_FINAL.pdf) — sekcie 5.2.1.6 (Hromadné spracovanie), 5.2.2 (Prepojenie na agendové systémy)
- Zdrojový kód registratura-services-integration:
  - `FormConfiguration.java` — pole `signature`: 1=pečať, 2=faximile, 3=mandátny podpis
  - `Scenario36Sign.java` — SOAP volanie CEP služby ÚPVS pre digitálne podpísanie
  - `ObjectsRegistryRecordProcessingSendDto.java` — `signRegistryRecordData` boolean flag
  - `SignRequestMapper.java` — mapovanie typov podpisov (XAdES, CAdES, PAdES, ASIC)
  - `application.properties` — DB pool max=30, acquisition timeout=60s
  - `CommonRouteConfiguration.java` — retry pattern: 10s, 60s, 300s

### Identifikované root causes

1. **Transaction timeout** (hlavný): JTA default ~300s, pečatenie 50 dokumentov prekročí limit → RollbackException
2. **CEP latencia**: Externý SOAP call pre každý dokument (2-10s per doc × 50 = 100-500s)
3. **Connection pool exhaustion**: Thready držia DB connections počas čakania na CEP
4. **Memory pressure**: F-013 (žiadna paginácia) + signing buffers

### Navrhnuté opatrenia

**Okamžité (hotfix):**
- O-1: Zvýšiť transaction timeout (300→600s)
- O-2: Zvýšiť DB connection pool (30→50)
- O-3: Znížiť veľkosť dávky na 20-25 pre formuláre s pečatením

**Strednodobé:**
- O-4: Oddeliť pečatenie od hlavnej transakcie (async)
- O-5: Paginácia pre S38 (F-013)
- O-6: Konfigurovateľný thread pool

**Dlhodobé:**
- O-7: Circuit breaker na CEP službu
- O-8: Monitoring a alerting

## Analýza 2 — F-017: Race condition v S312

### Čo sme zistili

Prečítané a analyzované:
- Všetky entity classes: `EntityBase.java`, `DocumentBase.java`, `BatchBase.java`, `Submission.java`
- Named queries pre batch spracovanie
- `JpaRouteConfiguration.java` — 3-level error handling
- DB schéma (`db.changelog-submission.xml`)
- `ReplayMessagesEndpointRoute.java` — split() pattern

### Kľúčový nález

**Úplná absencia concurrency control v celom batch pipeline:**
- Žiadny `@Version` (optimistický locking) na žiadnej entite
- Žiadny `SELECT FOR UPDATE` (pesimistický locking) v queries
- Žiadna deduplikácia pri paralelnom spracovaní cez `split()`
- `modified` trigger je čisto audit, nie conflict detection
- Dva thready môžu čítať, spracovať a zapísať ten istý záznam bez akejkoľvek detekcie konfliktu

### Navrhnuté riešenie

4 varianty, odporúčaný postup:

**Hotfix**: Variant D — `threads(1,1)` namiesto `threads(10,10)` pre S312. Eliminuje race condition okamžite, ale je pomalší.

**Produkcia**: Variant A — optimistický locking:
1. DB migrácia: `ALTER TABLE ... ADD version BIGINT NOT NULL DEFAULT 0`
2. Entity: `@Version private long version` na `EntityBase`
3. Camel: `onException(OptimisticLockException.class)` s retry + reload
4. Idempotency check pred IS AR volaním

Kompletný dizajn implementácie vrátane sekvenčného diagramu, testovacieho plánu a rollback plánu je v F-017.

## Analýza 3 — F-018: Návrh riešenia na 100K VD/deň

### Vstup: Mail Romana pre Nuaktiv (18.3.2026)

Tri emaily v `docs/podklady_davky/`:
1. Originál od Romana pre Nuaktiv (Jurík, Pavlenda, Klučiková)
2. Reply od Petra ("Dik. Super napísané")
3. Forward od Romana pre Petra (1.4.2026) — "Treba sa tomu venovať"

### Kľúčové dáta z mailu
- **Požiadavka SP**: 100K VD/deň
- **Aktuálny výkon**: ~1-2K VD/deň (jednovlákno)
- **Insert**: 3s, s pečatením +35s, celkom ~40s per VD
- **Test 50 vlákien**: 25 OK, zvyšok HTTP 403/500/503/504
- **Tri stratégie** od Romana: sekvenčné + optimalizácia, paralelné, async pečatenie

### Návrh riešenia

Hlavný princíp: **oddelenie pečatenia od Insertu** (Romanova Stratégia 3).

- Fáza 1 (synchrónna): Insert + Resolve bez pečatenia → ~5s per VD
- Fáza 2 (asynchrónna): Pečatenie na pozadí → ~35s per VD ale neblokuje

Cieľ: 5 vlákien × Insert 5s = ~57K VD/deň; po S3 migrácii ~96K VD/deň.

5 urgentných otázok pre Nuaktiv v dokumente.

## Výstupy session

- **F-016** — `docs/findings/F-016_2026-04-01_batch-sealing-timeout-crash.md`
- **F-017** — `docs/findings/F-017_2026-04-01_s312-race-condition-duplicate-processing.md`
- **F-018** — `docs/findings/F-018_2026-04-01_navrh-riesenia-vykonnost-davok.md`
- **STATE.md** aktualizovaný
- **BREAKDOWN.md** aktualizovaný (2B.9, 2B.10, 2B.11)
- Podklady: 3 emaily v `docs/podklady_davky/`
