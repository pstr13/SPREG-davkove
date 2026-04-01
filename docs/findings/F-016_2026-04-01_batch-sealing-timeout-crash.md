---
id: F-016
date: 2026-04-01
slug: batch-sealing-timeout-crash
title: "Pečatenie pri väčších dávkach (50+) spôsobuje timeout a pád spracovania"
severity: critical
status: identified
scenario: S312 (Scenario312BatchHandling) + Scenario36Sign
source: tester-report + code-analysis
reporter: milady (tester)
assignee: roman
related_bugs:
  - SPISREG-2688
  - SPISREG-2695
  - SPISREG-1517
  - SPISREG-2690
related_findings:
  - F-013
  - F-014
---

# F-016 — Pečatenie pri väčších dávkach (50+) spôsobuje timeout a pád spracovania

## Symptóm

Pri spustení hromadnej dávky s 50 dokumentmi a **zapnutým pečatením** sa extrémne nafukuje čas spracovania a dávka padne. Bez pečatenia dávka prebehne.

Nahlásené testerkou (Milady) pri testovaní DP hromadných dávok na TEST prostredí.

## Technický kontext

### Ako funguje pečatenie v batch procese

Konfigurácia pečatenia je v entite `FormConfiguration`:

```java
// FormConfiguration.java:72-77
/* Podpisovanie
 * 1 - pecat          (SEAL)
 * 2 - faximile       (FAX)
 * 3 - mandatny podpis (MANDATORY SIGNATURE)
 */
@Column(name = "signature", length = 20)
private Byte signature;
```

Keď je `signature = 1` (pečať), pre každý dokument v dávke sa volá:
1. `ObjectsRegistryRecordProcessingSendDto.setSignRegistryRecordData(true)` — IS AR spustí podpisovanie
2. IS AR zavolá **CEP službu ÚPVS** (Scenario36Sign) — externý SOAP call na digitálne podpísanie/opečatenie
3. CEP služba vráti podpísaný dokument

### Flow bez pečatenia (rýchly)

```
Pre každý dokument (paralelne, max threads):
  1. Insert registry record  → SOAP call IS AR (~0.5-2s)
  2. Process response         → DB write (~0.1s)
  3. Close batch when ready   → DB write (~0.1s)
```

**50 dokumentov × ~2s / 10 threadov ≈ 10 sekúnd**

### Flow s pečatením (pomalý)

```
Pre každý dokument (paralelne, max threads):
  1. Insert registry record   → SOAP call IS AR (~0.5-2s)
  2. Sign/seal record         → SOAP call IS AR → CEP ÚPVS (~2-10s)
  3. Process response          → DB write (~0.1s)
  4. Close batch when ready    → DB write (~0.1s)
```

**50 dokumentov × ~12s / 10 threadov ≈ 60 sekúnd (optimisticky)**
**Ak CEP je pomalšie alebo retry: 50 × 30s / 10 threadov ≈ 150-300+ sekúnd**

## Root Cause analýza — 4 príčiny

### Príčina 1: Transaction timeout (HLAVNÝ PODOZRIVÝ)

Quarkus/JTA má default transaction timeout (typicky **300 sekúnd**). Ak celý batch alebo jeho časť beží v jednej transakcii a pečatenie nafúkne čas nad tento limit:

- `javax.transaction.RollbackException` → celá dávka alebo jej časť sa rollbackne
- Spisy zostanú v stave "V riešení" namiesto "Vybavený" (symptóm SPISREG-2688, SPISREG-1517)
- Čísla záznamov sa neuložia do DB (symptóm SPISREG-1517)

**Overenie**: Hľadať v logoch `TransactionTimeout`, `RollbackException`, `TransactionRolledBackException` pri páde 50ks dávky.

**Kľúčová otázka**: Aký je aktuálny `quarkus.transaction-manager.default-transaction-timeout`?

### Príčina 2: Sériové volanie CEP služby

Pečatenie pridáva pre KAŽDÝ dokument extra SOAP call na externú CEP službu ÚPVS:
- CEP latencia: typicky 2-10 sekúnd per dokument
- Ak CEP je pod záťažou: 10-30+ sekúnd
- Retry pattern (CommonRouteConfiguration): 1. retry po 10s, 2. po 60s, 3. po 300s
- Jeden zlyhávajúci dokument môže pridať **370 sekúnd** do celkového času

### Príčina 3: Connection pool exhaustion

```properties
# application.properties
quarkus.datasource.jdbc.max-size=30
quarkus.datasource.jdbc.acquisition-timeout=60
```

- Každý thread drží DB connection počas celého spracovania dokumentu
- S pečatením drží connection **výrazne dlhšie** (čaká na CEP odpoveď)
- 10 threadov × dlhé pečatenie = connections blokované
- Ostatné thready čakajú na connection → `AcquisitionTimeoutException` po 60s

### Príčina 4: Memory pressure (súvisí s F-013)

- S38 načíta ALL dokumenty naraz bez paginácie (F-013)
- Pečatenie pridáva Base64 encoded content pre podpis do pamäte
- 50 dokumentov s prílohami + signing buffers → GC pauses alebo OOM
- GC pauses spomaľujú spracovanie → zhoršujú transaction timeout

## Korelácia s nahlásenými bugmi

| Bug | Prostredie | Agenda | Symptóm | Pravdepodobná príčina |
|-----|-----------|--------|---------|----------------------|
| **SPISREG-2688** | TEST | DP | Nový, assignutý na Romana | Transaction rollback po čiastočnom spracovaní |
| **SPISREG-2695** | TEST | JVP | "Rovnaká chyba ako 2688" | Rovnaký mechanizmus, iná agenda |
| **SPISREG-1517** | DEV | DP | Spisy "V riešení", chýbajúce čísla záznamov | Transaction rollback: insert prebehol, close nie |
| **SPISREG-2690** | TEST | DP | MEGA BUG (Tibor) | Možná kombinácia F-014 (double write) + timeout |
| **SPISREG-2394/2422/2423/2428** | DEV/TEST | JVP/DP | Nedotiahnutý spracovateľ | Oddelený problém — Nuaktiv / batch_registrar konfigurácia |

## Navrhované opatrenia

### OKAMŽITÉ (hotfix pred UAT)

#### O-1: Zvýšiť transaction timeout pre batch operácie

```properties
# application.properties alebo env variable
quarkus.transaction-manager.default-transaction-timeout=600
```

Alebo cielene na batch route:

```java
@TransactionTimeout(value = 600, unit = TimeUnit.SECONDS)
```

**Riziko**: Nízke. Dlhší timeout nezhoršuje výkon, len dáva viac času.
**Effort**: 1 konfiguračná zmena.

#### O-2: Zvýšiť DB connection pool pre prostredie s pečatením

```properties
quarkus.datasource.jdbc.max-size=50
quarkus.datasource.jdbc.acquisition-timeout=120
```

**Riziko**: Nízke. Pozor na limit DB servera.
**Effort**: 1 konfiguračná zmena.

#### O-3: Znížiť veľkosť dávky pre formuláre s pečatením

Ak existuje konfigurácia veľkosti dávky — znížiť na **20-25 dokumentov** pre formuláre kde `signature = 1`.

**Riziko**: Žiadne. Dávky sa spracujú v menších chunkoch.
**Effort**: Konfiguračná zmena v DB (FormConfiguration alebo Agenda).

### STREDNODOBÉ (pred produkciou)

#### O-4: Oddeliť pečatenie od hlavnej transakcie

Namiesto synchronného pečatenia v rámci batch transakcie:
1. Batch insert všetkých záznamov (bez pečatenia) → commit
2. Asynchrónne pečatenie per záznam v samostatných transakciách
3. Po opečatení všetkých → uzavrieť batch

**Výhoda**: Insert nikdy nevyexpiruje. Pečatenie môže byť retry-ované individuálne.
**Effort**: Stredný — vyžaduje nový Camel route step a stavový automat.

#### O-5: Implementovať pagináciu pre S38 (F-013)

```java
// Namiesto:
jpa:S38Document?namedQuery=selectS38DocumentForProcessing
// Použiť:
loopDoWhile + setMaxResults(maximumReadSize) // ako S312
```

**Výhoda**: Eliminuje OOM riziko. Zmenšuje transakcie.
**Effort**: Stredný — vyžaduje refaktoring S38 route.

#### O-6: Konfigurovateľný thread pool (F-015 súvisiace)

```java
// Namiesto hardcoded:
threads(10, 10)
// Použiť:
threads(configurationBean.minThreads(), configurationBean.maxThreads())
```

**Výhoda**: Laditeľné per prostredie bez redeployu.
**Effort**: Nízky.

### DLHODOBÉ (architektonické)

#### O-7: Circuit breaker na CEP službu

Implementovať circuit breaker (Camel alebo MicroProfile Fault Tolerance) na volanie CEP:
- Ak CEP nereaguje v X sekúnd → fail fast, nečakať
- Ak Y po sebe idúcich zlyhaní → otvoriť circuit, pozastaviť batch

**Výhoda**: Rýchly fail namiesto dlhého čakania a kaskádového zlyhania.

#### O-8: Monitoring a alerting

Pridať metriky:
- Čas pečatenia per dokument (percentily)
- Connection pool utilization
- Transaction duration
- Batch processing time vs. batch size

## Verifikačné kroky

1. **Logy**: Pri ďalšom páde 50ks dávky s pečatením zachytiť logy a hľadať:
   - `javax.transaction.RollbackException`
   - `TransactionTimeout`
   - `AcquisitionTimeoutException`
   - `java.lang.OutOfMemoryError`
   - CEP response times

2. **A/B test**: Rovnaká 50ks dávka — raz bez pečatenia, raz s pečatením. Porovnať časy.

3. **DB kontrola**: Overiť `quarkus.transaction-manager.default-transaction-timeout` na TEST prostredí.

4. **CEP latencia**: Zmerať priemerný response time CEP služby pod záťažou (50 concurrent requests).
