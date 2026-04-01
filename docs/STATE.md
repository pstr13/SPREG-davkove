# STAV

> Posledná aktualizácia: 2026-04-01

## Jednou vetou
F-016/F-017/F-018: pečatenie timeout + race condition + návrh riešenia na 100K VD/deň (async pečatenie).

## Čo je hotové
- GitHub repo vytvorené (pstr13/SPREG-davkove)
- AIGrip štruktúra scaffoldnutá (CLAUDE.md, docs/, .aigrip/)
- INTENT schválený
- Zdrojový repo JVP-nedoplatky pripojený ako zdroj (Fáza 1)
- Kompletná analýza zdrojových kódov IS REG integrácie (Fáza 1 — JVP repo)
- BREAKDOWN vytvorený s 3-fázovým prístupom
- **batch-services-integration repo stiahnutý a analyzovaný**
- **6 nových findingov identifikovaných (F-009 až F-014)** — 3 kritické, 3 vysoké
- docs/findings/ adresár vytvorený s detailnými nálezmi
- **F-010 opravený** v commit eabc3f5 (2026-03-31) — nested Batch filter fixed
- F-015 identifikovaný (regres z eabc3f5 — maximumRedeliveries hardcoded)
- **F-016 identifikovaný** (2026-04-01) — pečatenie pri 50ks dávkach: transaction timeout + CEP latencia

## Čo je rozpracované
- Verifikácia F-009 až F-014 Romanom
- F-001, F-002, F-003 odstránené (D-001) — boli z JVP repo, netýkajú sa S311/S312/S38
- S312CloseBatchWhenReadyStep @ApplicationScoped — investigate
- **F-016 opatrenia**: čaká na review Romanom a implementáciu hotfixov (O-1 až O-3)
- **F-017 identifikovaný** (2026-04-01) — race condition v S312: duálne spracovanie záznamu, chýba concurrency control
- **F-017 opatrenia**: hotfix Variant D (1 thread), produkcia Variant A (optimistický locking)
- **F-018 návrh**: async pečatenie oddelené od Insertu — stratégia na 100K VD/deň
- **F-019 identifikovaný** (2026-04-01) — chybný spracovateľ na zázname: `getNpUser()` hardcoded namiesto agenda-specific, `batch_registrar` prázdny pre JVP

## Čo blokuje
- **F-016**: 50ks dávky s pečatením nepoužiteľné kým sa neimplementuje aspoň O-1 (transaction timeout)
- **F-017**: Race condition v S312 — duálne spracovanie záznamov, MEGA BUG (SPISREG-2690)
- Milady pozastavila JVP 50+ dávky do opravy bugov

## Čo ďalej
1. **Roman (URGENTNÉ)**: F-017 hotfix — znížiť thready na 1 (Variant D) pre S312, eliminuje race condition
2. **Roman (URGENTNÉ)**: F-016 hotfix — zvýšiť transaction timeout (O-1), connection pool (O-2)
3. **Roman**: F-017 produkčný fix — implementovať Variant A (optimistický locking: @Version + ALTER TABLE)
4. **Roman**: Skontrolovať logy z pádu 50ks dávky — potvrdiť TransactionTimeout/RollbackException
5. **Roman**: Overiť F-009 (onException scope)
6. **Roman**: F-019 fix — agenda-specific user resolution v S334 + batch_registrar pre JVP (SPISREG-2394/2422/2423/2428) — viď docs/findings/F-019

## Prehľad bugov od Milady (2026-04-01)

### DEV
| Agenda | Bug | Popis | Assignee |
|--------|-----|-------|----------|
| JVP | SPISREG-2394 | Nedotiahnutý spracovateľ | Roman (Nuaktiv) |
| DP | SPISREG-1517 | Spisy V riešení, chýbajúce čísla | Roman |
| DP | SPISREG-2428 | Nedotiahnutý spracovateľ | Roman (Nuaktiv) |

### TEST
| Agenda | Bug | Popis | Assignee |
|--------|-----|-------|----------|
| JVP | SPISREG-2422 | Nedotiahnutý spracovateľ | Roman (Nuaktiv) |
| JVP | SPISREG-2695 | Nový, rovnaká chyba ako DP 2688 | Roman |
| DP | SPISREG-2423 | Nedotiahnutý spracovateľ | Roman (Nuaktiv) |
| DP | SPISREG-2688 | Nový | Roman |
| DP | SPISREG-2690 | MEGA BUG | Tibor |

## Zdravie
🔴 kritické — F-016 blokuje testovanie 50ks dávok s pečatením

## Metadáta
id: spreg-davkove
type: technical
framework_version: "0.7.4"
health: red
owner: roman
reviewer: tibor
started: 2026-03-31
last_active: 2026-04-01
session_active: "roman"
needs_help: false
help_description: ""
tags: [is-reg, legacy, code-audit, uat, batch-processing, sealing, timeout]
