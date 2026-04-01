---
id: J-004
date: 2026-04-01
slug: analyza-pecatenie-batch-timeout
status: draft
author: roman
---

# J-004 — Analýza výkonnosti dávok s pečatením: bugy, race condition, E2E návrh riešenia

## Vstupy do session

### Vstup 1: Prehľad bugov od Milady (testerka)

Kompletný report zo stavu testovania na DEV a TEST prostrediach.

**Hlavný problém**: "Dávky 50ks keď sme zapli pečatenie tak sa nafukuje čas a nám to padá."

**EESSI**: OK — dávky zbiehajú, značka CB02 OK.

**TEST — DP**: Pretestované vrátane 50ks dávky. Nájdené nové bugy.
**TEST — JVP**: Pretestované variácie pre FO/PO, tlačivá `30807484.Rozhodnutie_o_poistnom.sk.jvp` a `30807484.Rozhodnutie_na_penale.sk.jvp`, pobočka BAM. Väčšie dávky 50+ pozastavené.
**DEV**: Len retesty.

#### Kompletný zoznam bugov

| Prostredie | Agenda | Bug | Popis | Assignee |
|-----------|--------|-----|-------|----------|
| DEV | JVP | SPISREG-2394 | Nedotiahnutý spracovateľ | Roman (Nuaktiv/Pavlenda) |
| DEV | DP | SPISREG-1517 | Spisy "V riešení", chýbajúce čísla záznamov | Roman |
| DEV | DP | SPISREG-2428 | Nedotiahnutý spracovateľ | Roman (Nuaktiv/Pavlenda) |
| TEST | JVP | SPISREG-2422 | Nedotiahnutý spracovateľ | Roman (Nuaktiv/Pavlenda) |
| TEST | JVP | SPISREG-2695 | Nový, "rovnaká chyba ako DP 2688" | Roman |
| TEST | DP | SPISREG-2423 | Nedotiahnutý spracovateľ | Roman (Nuaktiv/Pavlenda) |
| TEST | DP | SPISREG-2688 | Nový | Roman |
| TEST | DP | SPISREG-2690 | **MEGA BUG** ("taký som ešte nezadala") | Tibor |

### Vstup 2: Potvrdenie race condition od Romana (developer)

> "Duálne spracovanie toho istého záznamu výstupných dokumentov kde jeden uspeje a druhý neuspeje a vzájomne si prepíšu výsledok. Scenár 3.1.2. V niektorých prípadoch vznikne aj správny záznam."

### Vstup 3: Email Romana pre Nuaktiv (18.3.2026)

Tri emaily v `docs/podklady_davky/`:
- Originál od Romana pre Nuaktiv (Jurík, Pavlenda, Klučiková) — popis problému, 3 stratégie, test paralelizmu
- Reply od Petra ("Dik. Super napísané")
- Forward od Romana pre Petra (1.4.2026) — "Treba sa tomu venovať"

**Kľúčové dáta z mailu**:
- Požiadavka SP: 100K VD/deň
- Insert: 3s základ, s pečatením +35s, celkom ~40s per VD
- Za každý VD: Insert + ResolveRecord + ResolveFile na IS AR
- Jednovlákno ACTUAL: 2 000 VD/deň (reálne pod 1 000)
- Test 50 vlákien na 100 VD: 25 OK, zvyšok HTTP 403/500/503/504, insert časy polovičné
- Stratégia 1: sekvenčné + optimalizácia (OpenKM→S3, redukcia pečatenia)
- Stratégia 2: paralelné (50 vlákien → nestabilné)
- Stratégia 3: async pečatenie (Insert bez seal, seal neskôr)
- Pavlenda prisľúbil preveriť čas pečatenia

### Vstup 4: Korekcie od riešiteľa (1.4.2026)

Spresňujúce informácie k návrhu:
1. **Kapacita CEP**: Nevieme → počítame worst case
2. **Insert bez pečatenia**: **NEDÁ SA** — pečatenie je neoddeliteľná súčasť Insert
3. **Batch sealing endpoint**: Neexistuje alebo nevieme
4. **PROD časy**: Nevieme. TEST: 3-30s
5. **Príčina 35s**: Nestabilný výkon TEST prostredia NASES (externá služba štátu)
6. **Princíp**: "Implementácia tejto legacy služby musí byť odolná"

**Bod 2 bol kritický** — zmenil celý návrh. Async oddelenie (Stratégia 3) nie je možné. Riešenie musí počítať s tým, že pečatenie je vždy súčasťou Insert volania.

## Analýzy a výstupy

### Analýza 1: F-016 — Pečatenie pri 50ks dávkach spôsobuje timeout a pád

**Analyzované zdroje**:
- DNR dokument (strany 103-147)
- FormConfiguration.java (signature: 1=pečať, 2=fax, 3=mandátny podpis)
- Scenario36Sign.java (SOAP + MTOM + WS-Security na CEP/ÚPVS)
- ObjectsRegistryRecordProcessingSendDto.java (signRegistryRecordData flag)
- application.properties (DB pool, timeouty)
- CommonRouteConfiguration.java (retry pattern)

**Root causes**: Transaction timeout, CEP latencia, connection pool, memory pressure.
**Navrhnuté opatrenia**: 8 (3 okamžité, 3 strednodobé, 2 dlhodobé) — detaily v F-016.

### Analýza 2: F-017 — Race condition v S312

**Analyzované zdroje**:
- EntityBase.java, DocumentBase.java, BatchBase.java, Submission.java — žiadny @Version
- Named queries — žiadny SELECT FOR UPDATE
- JpaRouteConfiguration.java — 3-level error handling
- db.changelog-submission.xml — žiadny version stĺpec
- ReplayMessagesEndpointRoute.java — zraniteľný split() pattern

**Kľúčový nález**: Úplná absencia concurrency control v celom batch pipeline.
**Navrhnuté riešenie**: 4 varianty (optimistický locking, pesimistický locking, claim check, 1 thread).
**Dizajn implementácie**: Variant A (optimistický locking) vrátane DB migrácie, entity zmeny, error handling, sekvenčného diagramu, testovacieho plánu.

### Analýza 3: F-018 — E2E dizajn odolnej implementácie

**Analyzované zdroje**: Všetky z F-016 + F-017 + email od Romana + korekcie.

**8 komponentov riešenia**:
1. Optimistický locking (@Version) — F-017 fix
2. Claim-based loading (processing_node, UPDLOCK READPAST) — zero duplikáty
3. Transakcia per dokument (REQUIRES_NEW) — F-016 fix
4. Odolné error handling (klasifikácia 4xx/5xx, exponenciálny backoff)
5. Circuit breaker (Resilience4j) — ochrana pred preťaženým NASES
6. Adaptívny paralelizmus (auto-scale 3→20 vlákien)
7. Fix F-014 (choice namiesto filter)
8. Paginácia S38 — F-013 fix

**3-fázový implementačný plán**:
- Fáza A (hotfix, 1-3 dni): Locking + timeout + F-014 fix → stabilita
- Fáza B (1-2 týždne): Claim + per-doc tx + error handling + circuit breaker → odolnosť
- Fáza C (2-4 týždne): Adaptive threads + paginácia + monitoring → škálovanie

**Cieľová priepustnosť**: 50-100K VD/deň (závisí od NASES PROD výkonu).

## Kategorizácia bugov

Bugy spadajú do 3 oddelených problémov:

1. **Pečatenie + timeout** (F-016): SPISREG-2688, SPISREG-2695, SPISREG-1517
2. **Race condition / duálne spracovanie** (F-017): SPISREG-2690 (MEGA BUG)
3. **Nedotiahnutý spracovateľ / Nuaktiv**: SPISREG-2394, SPISREG-2422, SPISREG-2423, SPISREG-2428

## Dokumenty vytvorené v session

| Dokument | Cesta |
|----------|-------|
| F-016 | `docs/findings/F-016_2026-04-01_batch-sealing-timeout-crash.md` |
| F-017 | `docs/findings/F-017_2026-04-01_s312-race-condition-duplicate-processing.md` |
| F-018 | `docs/findings/F-018_2026-04-01_navrh-riesenia-vykonnost-davok.md` |
| J-004 | `docs/journal/J-004_2026-04-01_analyza-pecatenie-batch-timeout.md` |
| Podklady | `docs/podklady_davky/` (3 emaily) |
