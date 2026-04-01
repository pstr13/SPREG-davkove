---
id: F-018
date: 2026-04-01
slug: navrh-riesenia-vykonnost-davok
title: "Návrh riešenia: Výkonnosť dávkových scenárov s pečatením — stratégia na 100K VD/deň"
severity: critical
status: proposal
scenario: S312 (Scenario312BatchHandling) + Scenario36Sign
source: roman-email-2026-03-18 + code-analysis + F-016 + F-017
reporter: roman
assignee: roman
---

# F-018 — Návrh riešenia: Výkonnosť dávkových scenárov s pečatením

## Východiská (z mailu Romana 18.3.2026 + testovanie)

### Biznis požiadavka
- SP požaduje spracovanie **~100 000 VD/deň** (EXPECTED)
- Aj polovica (50K) by bola akceptovateľná

### Namerané časy (LAB/TEST)
| Operácia | Bez pečatenia | S pečatením |
|----------|:------------:|:-----------:|
| Insert (IS AR) | ~3s | ~3s |
| Pečatenie (CEP/ÚPVS) | — | **~35s** (fluktuje, aj 40s+) |
| ResolveRecord | ~1s | ~1s |
| ResolveFile | ~1s | ~1s |
| **Celkom per VD** | **~5s** | **~40s** |

### Aktuálna priepustnosť (jednovlákno)
| Metrika | Hodnota |
|---------|---------|
| Čas per VD | ~40s |
| VD/hodinu | 90 |
| VD/deň (24h) | 2 160 |
| VD/deň (reálne, 16h okno) | **~1 440** |
| **GAP oproti požiadavke** | **~70x** |

### Test paralelizmu (Roman, 18.3.2026)
- 100 VD, 50 vlákien, prerušené po 5 min
- 25+ VD úspešne
- Zvyšok: **HTTP 403, 500, 503, 504** — Registratúra nezvláda záťaž
- Insert časy boli polovičné (1.5s) → S3 migrácia pomôže
- Záver: **agresívny paralelizmus nefunguje**, Registratúra má limity

### Čo vieme od Nuaktivu
- Nuaktiv (Pavlenda) žiadal **jednovláknové** spracovanie
- Prisľúbil preveriť čas pečatenia (35s) a možnosti redukcie

## Analýza problému

### Kde je čas?

```
█████████████████████████████████████████████████ 40s celkom
███ Insert 3s (7.5%)
████████████████████████████████████ Pečatenie 35s (87.5%) ← BOTTLENECK
█ Resolve 2s (5%)
```

**87.5% času je pečatenie.** Všetko ostatné je sekundárne.

### Prečo jednoduché škálovanie nefunguje

| Stratégia | VD/deň | Problém |
|-----------|--------|---------|
| 1 vlákno, 40s/VD | 1 440 | 70x pod cieľom |
| 10 vlákien, 40s/VD | 14 400 | HTTP errors, race condition (F-017) |
| 50 vlákien | ≤7 200 | Registratúra padá (403/500/503/504) |
| 1 vlákno, Insert na S3 (<1s) | 2 400* | Stále 35s pečatenie dominuje |

*\* Optimistický odhad: 35s pečatenie + 1s Insert + 2s Resolve = 38s → 2 273 VD/deň*

**Záver: Bez zmeny architektúry pečatenia sa nedá dostať na 100K VD/deň.**

## Navrhované riešenie: Kombinovaná stratégia

### Princíp: Oddelenie synchrónneho a asynchrónneho spracovania

```
FÁZA 1 (synchrónna, rýchla):     FÁZA 2 (asynchrónna, na pozadí):
  Insert → Resolve → DONE          Pečatenie → Potvrdenie
  ~5s per VD                        ~35s per VD, ale neprekáža
```

Roman v maile správne identifikoval (Stratégia 3):
> "Pre hromadné spracovanie je kľúčová až odpoveď (potvrdenie o odoslaní do eDesk) ktoré príde asynchrónne neskôr cez EOD. Samotným Insertom sa pre nás nepotvrdzuje fakt umiestnenia VD do eDesk, čiže pre integračný komponent je akceptovateľné ak sa pečatenie udeje až po úspešne odpovedanej Insert operácii."

Toto je kľúčový biznis predpoklad — **pečatenie nemusí byť synchrónne s Insertom**.

### Architektúra riešenia

```
                    FÁZA 1: Batch Insert Pipeline
                    ────────────────────────────
                    ┌─────────────┐
  Dávka VD ──────► │ Load batch  │
  (zo sieťového    │ (paginated) │
  úložiska)        └──────┬──────┘
                          │
                    ┌─────▼──────┐
                    │  INSERT    │ ~1-3s per VD
                    │  (IS AR)   │ signRegistryRecordData=FALSE
                    └──────┬─────┘
                          │
                    ┌─────▼──────────┐
                    │ ResolveRecord  │ ~1s
                    │ ResolveFile    │ ~1s
                    └──────┬─────────┘
                          │
                    ┌─────▼──────┐
                    │  Mark as   │
                    │ INSERTED   │ (nový stav, nie DONE)
                    └──────┬─────┘
                          │
              ┌───────────▼────────────┐
              │ Enqueue for sealing    │
              │ (DB queue / seda)      │
              └───────────┬────────────┘
                          │
                    FÁZA 2: Sealing Pipeline (async)
                    ─────────────────────────────────
                          │
              ┌───────────▼────────────┐
              │ Pick INSERTED docs     │
              │ (throttled, N vlákien) │
              └───────────┬────────────┘
                          │
              ┌───────────▼────────────┐
              │ SignRegistryRecord     │ ~35s per VD
              │ (CEP/ÚPVS)            │ ale v separátnej transakcii
              └───────────┬────────────┘
                          │
              ┌───────────▼────────────┐
              │ Mark as SEALED/DONE    │
              │ Close batch if all OK  │
              └────────────────────────┘
```

### Kľúčové zmeny

#### 1. Insert BEZ pečatenia

```java
// S312InsertRegistryRecordOutboundAndProcessStep.java
// ZMENA: Vždy signRegistryRecordData = false pri batch Insert
ObjectsRegistryRecordProcessingSendDto sendDto = new ObjectsRegistryRecordProcessingSendDto();
sendDto.setSignRegistryRecordData(false);  // ← pečatenie ODDELENÉ
request.getProcessing().setSending(sendDto);
```

Toto je **minimálny zásah** — jeden riadok kódu. Insert sa zrýchli z 40s na ~5s.

#### 2. Nový stav INSERTED

```sql
-- Nový stav v stavovom automate dokumentu
-- NEW → INSERTED → SEALED → DONE
--                → SEAL_FAILED → (retry) → SEALED → DONE
ALTER TABLE s312_document ADD seal_status VARCHAR(20) DEFAULT 'PENDING';
```

#### 3. Nový Camel route: SealingPipeline

```java
// S312SealingRouteConfiguration.java — NOVÝ route
from("jpa:S312Document?namedQuery=selectInsertedForSealing"
     + "&maximumResults={{env:SEALING_BATCH_SIZE:50}}"
     + "&consumeDelay={{env:SEALING_POLL_INTERVAL:5000}}"
     + "&lockModeType=PESSIMISTIC_WRITE")  // ← F-017 fix: locking!
    .routeId("S312SealingPipeline")
    .threads({{env:SEALING_THREADS:5}}, {{env:SEALING_THREADS:5}})
    .throttle({{env:SEALING_THROTTLE:10}}).timePeriodMillis(1000)  // max 10/s
    .process(exchange -> {
        S312Document doc = exchange.getMessage().getBody(S312Document.class);
        // Volanie SignRegistryRecord na IS AR
        SignRegistryRecordRequestDto req = new SignRegistryRecordRequestDto();
        req.setRegistryRecordID(doc.getRegistryRecordId());
        // ...
    })
    .to("direct:signRegistryRecord")
    .process(exchange -> {
        // Mark as SEALED
        S312Document doc = exchange.getProperty(Constant.DOCUMENT, S312Document.class);
        doc.setSealStatus("SEALED");
    })
    .to("jpa:S312Document")
    .onException(Exception.class)
        .handled(true)
        .process(exchange -> {
            S312Document doc = exchange.getProperty(Constant.DOCUMENT, S312Document.class);
            doc.setSealStatus("SEAL_FAILED");
            doc.incrementSealAttempts();
        })
        .to("jpa:S312Document");
```

#### 4. Throttling na CEP — rešpektovanie limitov Registratúry

```properties
# Konfigurovateľné parametre
SEALING_THREADS=5           # Počet vlákien pre pečatenie
SEALING_THROTTLE=10         # Max requestov za sekundu na CEP
SEALING_BATCH_SIZE=50       # Koľko docs načítať naraz
SEALING_POLL_INTERVAL=5000  # Ako často pollovať nové INSERTED docs
SEALING_MAX_RETRIES=3       # Max retry pre seal failure
```

### Výpočet priepustnosti

#### Fáza 1 (Insert pipeline) — bottleneck je IS AR Insert

| Parametre | Jednovlákno | 5 vlákien | 10 vlákien |
|-----------|:-----------:|:---------:|:----------:|
| Insert + Resolve per VD | ~5s | ~5s | ~5s |
| VD/hodinu | 720 | 3 600 | 7 200 |
| VD/deň (16h) | 11 520 | **57 600** | **115 200** |

Po S3 migrácii (Insert < 1s):

| Parametre | Jednovlákno | 5 vlákien | 10 vlákien |
|-----------|:-----------:|:---------:|:----------:|
| Insert + Resolve per VD | ~3s | ~3s | ~3s |
| VD/hodinu | 1 200 | 6 000 | 12 000 |
| VD/deň (16h) | 19 200 | **96 000** | **192 000** |

**S 5 vláknami po S3 migrácii: ~96K VD/deň → spĺňa požiadavku 100K (s reservou pri 10 vláknach).**

#### Fáza 2 (Sealing pipeline) — beží na pozadí

| Parametre | 5 vlákien, 35s/VD |
|-----------|:-----------------:|
| VD/hodinu | 514 |
| VD/deň (24h) | **12 343** |

**Pečatenie nestihne 100K/deň s 5 vláknami.** Ale:
- Pečatenie môže bežať 24/7 (nie je viazané na biznis hodiny)
- S 20 vláknami a throttlingom: **~49K VD/deň**
- S 50 vláknami: **~123K VD/deň** — ale Registratúra to nemusí zvládnuť

**Otázka pre Nuaktiv**: Koľko paralelných pečatení CEP služba zvládne bez HTTP 5xx?

### Alternatíva: Batch sealing API

Ak Nuaktiv/Registratúra podporuje alebo by vedeli implementovať **batch sealing** (opečatiť N záznamov v jednom volaní), dramaticky by to zmenilo výpočet:

| Batch size | Čas | VD/hodinu (1 vlákno) | VD/deň |
|-----------|:---:|:-------------------:|:------:|
| 1 (teraz) | 35s | 103 | 2 469 |
| 10 | 40s? | 900 | 21 600 |
| 50 | 60s? | 3 000 | 72 000 |
| 100 | 90s? | 4 000 | **96 000** |

**Otázka pre Nuaktiv**: Existuje alebo dá sa implementovať batch sealing endpoint?

## Implementačný plán

### Fáza A: Okamžitý hotfix (1-2 dni)

| # | Akcia | Effort | Efekt |
|---|-------|--------|-------|
| A1 | `signRegistryRecordData = false` pre batch Insert | 1 riadok kódu | Insert z 40s na 5s |
| A2 | Zvýšiť transaction timeout (F-016 O-1) | Konfig | Eliminuje timeout crash |
| A3 | `@Version` na EntityBase (F-017 Variant A) | DB + Java | Eliminuje race condition |
| A4 | Znížiť thready na 3-5 (nie 10, nie 50) | Konfig | Stabilné paralelné spracovanie |

**Po Fáze A**: Insert pipeline funguje, ale dokumenty NIE SÚ opečatené. Pečatenie sa musí vyriešiť v Fáze B.

**POZOR**: Fáza A sa dá nasadiť len ak je biznesovo akceptovateľné, že Insert prebehne bez okamžitého pečatenia. Roman toto potvrdil v maile ako akceptovateľné.

### Fáza B: Async Sealing Pipeline (1-2 týždne)

| # | Akcia | Effort | Popis |
|---|-------|--------|-------|
| B1 | Nový stav `seal_status` v DB | DB migrácia | PENDING/SEALED/SEAL_FAILED |
| B2 | Nový Camel route `S312SealingPipeline` | Nový Java súbor | Načíta INSERTED docs, volá SignRegistryRecord |
| B3 | Throttling + konfigurovateľný thread pool | Konfig | Rešpektuje limity Registratúry |
| B4 | Error handling + retry pre sealing | Java | Separátne od Insert retry |
| B5 | Monitoring: metriky pečatenia | Java | Čas, success rate, queue depth |
| B6 | Batch close adjustment | Java | Batch = DONE až keď všetky docs SEALED |

### Fáza C: Optimalizácia (priebežne)

| # | Akcia | Efekt |
|---|-------|-------|
| C1 | OpenKM → S3 migrácia | Insert pod 1s |
| C2 | Batch sealing API (ak Nuaktiv podporí) | 10-50x zrýchlenie pečatenia |
| C3 | Connection pooling optimization | Lepšia stabilita pod záťažou |
| C4 | Circuit breaker na CEP | Rýchly fail namiesto čakania |

## Riziká a mitigácia

| Riziko | Pravdepodobnosť | Dopad | Mitigácia |
|--------|:---------------:|:-----:|-----------|
| Nuaktiv odmietne async pečatenie | Stredná | Blocker | Escalalovať na SP, biznis argument (100K/deň nemožné inak) |
| CEP nezvládne ani 5 paralelných sealingov | Vysoká | Spomalenie | Throttling, monitoring, postupné zvyšovanie |
| Resolve operácie zlyhajú na neopečatenom zázname | Nízka | Stredný | Roman potvrdil že Resolve je nezávislý od pečatenia |
| Race condition v novom sealing pipeline | Stredná | Vysoký | @Version + PESSIMISTIC_WRITE od začiatku |
| Batch nie je DONE kým nie sú všetky sealed | Istá | Biznis | Nový stav PARTIALLY_SEALED, reporting pre SP |

## Otázky pre Nuaktiv (URGENTNÉ)

1. **Koľko paralelných pečatení CEP služba zvládne?** (5? 10? 20? Aký je rate limit?)
2. **Dá sa Insert zavolať bez pečatenia a pečatenie urobiť neskôr separátnym volaním?** (SignRegistryRecord)
3. **Existuje alebo plánuje sa batch sealing endpoint?** (opečatiť N záznamov naraz)
4. **Aký je reálny čas pečatenia v PROD prostredí?** (LAB = 35-40s, je PROD rýchlejší?)
5. **Čo spôsobuje 35s pečatenie?** (Je to CEP služba ÚPVS, alebo interná operácia Registratúry?)

## Záver

**Bez oddelenia pečatenia od Insertu sa 100K VD/deň nedá dosiahnuť.** Ani so S3 migráciou, ani s paralelizmom. Pečatenie (35s) je 87.5% celkového času a Registratúra nezvláda agresívny paralelizmus.

Odporúčaný prístup: **Stratégia 3 z Romanovho mailu** (async pečatenie) + kontrolovaný paralelizmus (5-10 vlákien) + throttling.

Cieľová priepustnosť po implementácii: **~100K VD/deň** (Insert pipeline) s pečatením dobiehaným asynchrónne.
