---
id: J-004
date: 2026-04-01
slug: analyza-pecatenie-batch-timeout
status: draft
author: roman
---

# J-004 — Analýza: Pečatenie pri 50ks dávkach spôsobuje timeout a pád

## Čo sa riešilo

Testerka Milady nahlásila, že pri 50ks dávkach so zapnutým pečatením sa nafukuje čas a dávka padá. Zároveň dodala kompletný prehľad bugov naprieč DEV a TEST prostrediami pre JVP a DP agendy.

## Vstup od Milady — prehľad bugov

### DEV
- **JVP**: SPISREG-2394 (nedotiahnutý spracovateľ, Nuaktiv)
- **DP**: SPISREG-1517 (spisy V riešení, chýbajúce čísla), SPISREG-2428 (nedotiahnutý spracovateľ)

### TEST
- **JVP**: SPISREG-2422 (nedotiahnutý spracovateľ), SPISREG-2695 (nový, rovnaká chyba ako DP 2688)
- **DP**: SPISREG-2423 (nedotiahnutý spracovateľ), SPISREG-2688 (nový), SPISREG-2690 (MEGA BUG, Tibor)

### EESSI
- Dávky zbiehajú OK, značka CB02 OK

### Stav testovania
- DEV: len retesty opravených bugov
- TEST DP: pretestované vrátane 50ks dávky, nájdené nové bugy
- TEST JVP: pretestované variácie pre FO/PO, 50+ dávky pozastavené do opravy bugov

## Analýza

Analyzované zdrojové kódy (registratura-services-integration + batch-services-integration), DNR dokument a existujúce findingy.

**Kľúčový nález**: Pečatenie (FormConfiguration.signature = 1) pridáva pre každý dokument externý SOAP call na CEP službu ÚPVS (Scenario36Sign). Pri 50 dokumentoch to môže nafúknuť čas nad JTA transaction timeout → RollbackException → symptómy pozorované v SPISREG-2688 a SPISREG-1517.

## Výstupy

- **F-016** — nový finding: batch sealing timeout crash (critical)
- Identifikované 4 root causes: transaction timeout, CEP latencia, connection pool, memory pressure
- Navrhnuté 8 opatrení (3 okamžité, 3 strednodobé, 2 dlhodobé)
- Korelácia 5 bugov s finding F-016

## Druhá analýza — F-017: Race condition v S312

Roman potvrdil symptóm: duálne spracovanie toho istého záznamu výstupných dokumentov v S312. Jeden uspeje, druhý neuspeje, vzájomne si prepíšu výsledok.

Analýza kódu odhalila **úplnú absenciu concurrency control**:
- Žiadny `@Version` (optimistický locking) na entitách
- Žiadny `SELECT FOR UPDATE` (pesimistický locking) v batch queries
- Žiadna deduplikácia pri `split()` + paralelnom spracovaní

Navrhnuté 4 varianty riešenia, odporúčaný: **Variant A (optimistický locking)** — `@Version` na EntityBase + ALTER TABLE + ošetrenie OptimisticLockException.

Pravdepodobne zodpovedá za SPISREG-2690 (MEGA BUG).

## Poznámky

Tri oddelené problémy:
1. **Pečatenie + timeout** (F-016) — SPISREG-2688, 2695, 1517
2. **Race condition / duálne spracovanie** (F-017) — SPISREG-2690 (MEGA BUG)
3. **Nedotiahnutý spracovateľ / Nuaktiv** — SPISREG-2394, 2422, 2423, 2428 — Roman rieši s p. Pavlendom, oddelené
