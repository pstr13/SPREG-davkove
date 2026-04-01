---
id: F-018
date: 2026-04-01
slug: navrh-riesenia-vykonnost-davok
title: "Návrh riešenia: Výkonnosť dávkových scenárov s pečatením — E2E dizajn odolnej implementácie"
severity: critical
status: proposal
scenario: S312 (Scenario312BatchHandling) + Scenario36Sign
source: roman-email-2026-03-18 + code-analysis + F-016 + F-017 + korekcie-od-riesitela
reporter: roman
assignee: roman
vzťah_k_ostatným: Zahŕňa fixy F-014, F-016, F-017. Nenahrádzuje ich — tie zostávajú ako samostatné analýzy.
---

# F-018 — E2E dizajn: Odolná implementácia dávkového spracovania s pečatením

## 1. Východiská a obmedzenia

### 1.1 Biznis požiadavka
- SP požaduje spracovanie **~100 000 VD/deň** (aj 50K akceptovateľné)

### 1.2 Namerané časy (TEST prostredie)

| Operácia | Čas | Zdroj |
|----------|:---:|-------|
| Insert (IS AR) s pečatením | **3–30s** | Roman, mail 18.3.2026; LAB aj 40s+ |
| ResolveRecord | ~1s | Roman, mail 18.3.2026 |
| ResolveFile | ~1s | Roman, mail 18.3.2026 |
| **Celkom per VD** | **5–32s** | |

### 1.3 Overené obmedzenia

| Obmedzenie | Stav | Zdroj |
|------------|------|-------|
| Insert BEZ pečatenia, pečatenie neskôr | **NEDÁ SA** | Korekcia od riešiteľa (1.4.2026) |
| Batch sealing endpoint | **Neexistuje / nevieme** | Korekcia od riešiteľa |
| Kapacita CEP pre paralelné pečatenia | **Nevieme** | Korekcia — počítame worst case |
| PROD časy pečatenia | **Nevieme** | Korekcia — TEST NASES je nestabilné |
| Paralelizmus 50 vlákien | **NEFUNGUJE** | Romanov test 18.3.2026: HTTP 403/500/503/504 |

### 1.4 Príčina dlhého pečatenia
Spôsobená **nestabilným výkonom TEST prostredia NASES** (externá služba štátu), ktoré realizuje pečatenie. Nie je to systematický problém kódu.

### 1.5 Kľúčový princíp
> **Implementácia tejto legacy služby musí byť odolná.**

### 1.6 Identifikované bugy v aktuálnom kóde

| ID | Problém | Dopad | Zdroj |
|----|---------|-------|-------|
| **F-017** | Race condition — žiadny @Version, žiadny locking | Duálne spracovanie, data corruption | Analýza kódu: EntityBase.java, Submission.java, JpaRouteConfiguration.java |
| **F-016** | Transaction timeout pri väčších dávkach s pečatením | Celá dávka padne | Analýza: application.properties, FormConfiguration.java, Scenario36Sign.java |
| **F-014** | Double JPA write pri max retry (filter vs choice) | failedAttempts > max | S312RouteConfiguration.java:168-185 |
| **F-013** | Žiadna paginácia v S38 | OOM riziko | Scenario38ImportCSV.java:162 |
| **F-009** | Missing endChoice v S311 | Tiché zlyhania | Scenario311Notify.java:46-59 |

## 2. Zdrojové súbory (referencie pre developera)

### 2.1 Analyzované zdrojové kódy

Všetky cesty relatívne k: `JVP-nedoplatky/podklady/project-setup-materials/Podklady ku požiadavke/aktualne_zdrojove_kody/registratura-services-integration/`

**Entity a DB model:**
- `common-integration/src/main/java/sk/socpoist/isreg/regint/common/database/EntityBase.java` — base class, **chýba @Version**
- `common-integration/src/main/java/sk/socpoist/isreg/regint/common/database/DocumentBase.java` — document base
- `common-integration/src/main/java/sk/socpoist/isreg/regint/common/database/BatchBase.java` — batch base
- `common-integration/src/main/java/sk/socpoist/isreg/regint/common/entity/Submission.java` — submission entity, **žiadny locking**
- `common-integration/src/main/java/sk/socpoist/isreg/regint/common/database/FormConfiguration.java` — konfigurácia formulárov, pole `signature` (1=pečať, 2=fax, 3=mandátny podpis), pole `batch`
- `common-integration/src/main/java/sk/socpoist/isreg/regint/common/database/Agenda.java` — agenda konfigurácia, `batchOperator`, `batchRegistrar`

**Route konfigurácie:**
- `common-integration/src/main/java/sk/socpoist/isreg/regint/common/shared/CommonRouteConfiguration.java` — retry pattern (10s, 60s, 300s), env:MAXIMUM_REDELIVERIES
- `common-integration/src/main/java/sk/socpoist/isreg/regint/common/jpa/JpaRouteConfiguration.java` — 3-level error handling (Level 1: unresponsive, Level 2: API fault, Level 3: system failure)
- `common-integration/src/main/java/sk/socpoist/isreg/regint/common/shared/Constant.java` — konštanty S312_FOLDER_*, BATCH_SUBMISSION_*

**Pečatenie a podpisovanie:**
- `src/main/java/sk/socpoist/isreg/regint/registratura/scenario/Scenario36Sign.java` — SOAP 1.2 + MTOM + WS-Security na CEP/ÚPVS
- `src/main/java/sk/socpoist/isreg/regint/registratura/mapper/SignRequestMapper.java` — mapovanie typov podpisov (XAdES, CAdES, PAdES, ASIC)
- `src/main/java/sk/socpoist/isreg/regint/registratura/step/GetSignContentResponseStep.java` — spracovanie signing response
- `common-integration/src/main/java/lomtec/register/model/ObjectsRegistryRecordProcessingSendDto.java` — `signRegistryRecordData` (Boolean)
- `common-integration/src/main/java/lomtec/register/model/SignRegistryRecordRequestDto.java` — request na podpísanie

**Insert a spracovanie:**
- `src/main/java/sk/socpoist/isreg/regint/registratura/step/InsertRegistryRecordStep.java` — insert záznamu, nastavuje signRegistryRecordData=false (riadky 94-97)
- `src/main/java/sk/socpoist/isreg/regint/registratura/endpoint/ReplayMessagesEndpointRoute.java` — zraniteľný split() pattern bez lockingu

**Konfigurácia:**
- `src/main/resources/application.properties` — DB pool max=30, acquisition timeout=60s, replay age=30min
- `common-integration/src/main/resources/sql/changes/000-init.sql` — form_configuration dáta
- `common-integration/src/main/resources/sql/changes/009-update-agenda.xml` — batch_registrar per agenda
- `common-integration/src/main/resources/sql-submission/db.changelog-submission.xml` — submission schéma, **žiadny version stĺpec**
- `values.yaml` — maximumRedeliveries: 0

### 2.2 Analyzovaná dokumentácia

- `JVP-nedoplatky/podklady/project-setup-materials/R1_1_DNR_DETAILNY_NAVRH_RIESENIA_Projekt_ISREG_2.2_FINAL.pdf` — DNR, strany 103-147 (TO-BE procesy, hromadné spracovanie, prepojenia agendové systémy)

### 2.3 Podklady (emaily)

- `docs/podklady_davky/SocPoist, IS REG, Výkonnosť dávkových scenárov s pečatením.eml` — Romanov mail pre Nuaktiv (18.3.2026) — 3 stratégie, časy, test paralelizmu
- `docs/podklady_davky/Re_ SocPoist, IS REG, Výkonnosť dávkových scenárov s pečatením.eml` — Peter reply
- `docs/podklady_davky/Re_ SocPoist, IS REG, Výkonnosť dávkových scenárov s pečatením 2.eml` — Roman forward (1.4.2026)

### 2.4 Existujúce findingy (v tomto repo)

- `docs/findings/F-009_2026-03-31_s311-onexception-missing-endchoice.md`
- `docs/findings/F-013_2026-03-31_s38-no-pagination-oom-risk.md`
- `docs/findings/F-014_2026-03-31_s312-double-jpa-write-max-retry.md`
- `docs/findings/F-016_2026-04-01_batch-sealing-timeout-crash.md`
- `docs/findings/F-017_2026-04-01_s312-race-condition-duplicate-processing.md`

## 3. E2E dizajn riešenia

### 3.1 Cieľový stav: Architektonický diagram

```
                     ┌──────────────────────────────┐
  Dávka VD           │     RIADIACA VRSTVA           │
  (sieťové úložisko) │  - adaptívny paralelizmus     │
        │            │  - circuit breaker             │
        ▼            │  - monitoring                  │
  ┌───────────┐      └──────────┬───────────────────┘
  │ Load      │                 │ riadi počet vlákien
  │ (paginated│                 ▼
  │  chunks)  │      ┌─────────────────────┐
  └─────┬─────┘      │   WORKER POOL       │
        │            │   N = 3..20         │
        ▼            │   adaptívne         │
  ┌──────────┐       └─────────┬───────────┘
  │ DB Queue │                 │
  │ (claimed │─────────────────┘
  │  rows)   │       Každý worker:
  └──────────┘       ┌────────────────────────────────┐
                     │  CLAIM docs (atomic UPDATE)     │
                     │  ┌─────────────────────────┐   │
                     │  │ Per-VD (REQUIRES_NEW tx) │   │
                     │  │  INSERT + seal (IS AR)   │   │ 3-30s
                     │  │  ResolveRecord           │   │ ~1s
                     │  │  ResolveFile             │   │ ~1s
                     │  │  COMMIT → DONE           │   │
                     │  └─────────────────────────┘   │
                     │  OnError:                       │
                     │  - 5xx/timeout → retry (backoff)│
                     │  - 4xx → fail permanent         │
                     │  - circuit open → pause + return│
                     └────────────────────────────────┘
```

### 3.2 Komponent 1: Optimistický locking (F-017 fix)

**Účel**: Zabrániť dvom vláknam modifikovať rovnaký záznam bez detekcie konfliktu.

**Zmeny v kóde:**

```java
// === EntityBase.java ===
// PRIDAŤ:
@Version
@Column(name = "version")
private long version;

public long getVersion() { return version; }
protected void setVersion(long version) { this.version = version; }
```

**DB migrácia (Liquibase):**

```xml
<changeSet id="F017-add-version-columns" author="roman">
    <addColumn tableName="s312_document">
        <column name="version" type="BIGINT" defaultValueNumeric="0">
            <constraints nullable="false"/>
        </column>
    </addColumn>
    <addColumn tableName="submission">
        <column name="version" type="BIGINT" defaultValueNumeric="0">
            <constraints nullable="false"/>
        </column>
    </addColumn>
    <!-- Rovnako pre s311_document, s38_document, s312_batch, s38_batch -->
</changeSet>
```

**Error handling:**

```java
// === JpaRouteConfiguration.java ===
// PRIDAŤ pred existujúce handlery:
routeConfiguration("OptimisticLockConfiguration")
    .onException(OptimisticLockException.class,
                 StaleObjectStateException.class)
    .handled(true)
    .maximumRedeliveries(3)
    .redeliveryDelay(500)
    .retryAttemptedLogLevel(LoggingLevel.WARN)
    .log(LoggingLevel.WARN,
         "OptimisticLock conflict doc=${exchangeProperty.DOCUMENT_ID}, "
         + "retry ${header.CamelRedeliveryCounter}/3")
    .process(exchange -> {
        // MUSÍME reloadnúť entitu z DB — stará verzia je stale
        // EntityManager.find() s novým version
    });
```

### 3.3 Komponent 2: Claim-based loading

**Účel**: Garantovať že žiadne dva vlákna nespracúvajú ten istý dokument.

**DB migrácia:**

```xml
<changeSet id="F017-add-claim-columns" author="roman">
    <addColumn tableName="s312_document">
        <column name="processing_node" type="VARCHAR(50)"/>
    </addColumn>
    <addColumn tableName="s312_document">
        <column name="claimed_at" type="DATETIME2"/>
    </addColumn>
    <createIndex tableName="s312_document" indexName="idx_s312_claim">
        <column name="state"/>
        <column name="processing_node"/>
    </createIndex>
</changeSet>
```

**Claim query (native SQL, atomic):**

```sql
-- Named query: claimS312DocumentsForProcessing
UPDATE s312_document
SET processing_node = :nodeId, claimed_at = GETDATE()
WHERE id IN (
    SELECT TOP :batchSize id
    FROM s312_document WITH (UPDLOCK, READPAST)
    WHERE state = 'NEW' AND processing_node IS NULL
    ORDER BY created ASC
)
```

`WITH (UPDLOCK, READPAST)` je kľúčové pre MSSQL:
- `UPDLOCK`: zamkne riadky pre update
- `READPAST`: preskočí zamknuté riadky (iné vlákno ich už claimuje)

**Load claimnutých:**

```sql
-- Named query: selectClaimedS312Documents
SELECT d FROM S312Document d
WHERE d.processingNode = :nodeId AND d.state = 'NEW'
ORDER BY d.created ASC
```

**Stale claim cleanup (scheduler):**

```java
@Scheduled(every = "5m")
void cleanupStaleClaims() {
    entityManager.createNativeQuery(
        "UPDATE s312_document SET processing_node = NULL, claimed_at = NULL " +
        "WHERE processing_node IS NOT NULL " +
        "AND claimed_at < DATEADD(MINUTE, -30, GETDATE()) " +
        "AND state = 'NEW'")
        .executeUpdate();
}
```

### 3.4 Komponent 3: Transakcia per dokument (F-016 fix)

**Účel**: Zlyhanie jedného VD neovplyvní ostatné. Žiadny hromadný rollback.

**Zmena v route:**

```java
// === S312RouteConfiguration.java ===
// Namiesto jednej transakcie pre split() výsledok:
from("direct:processS312Document")
    .transacted("PROPAGATION_REQUIRES_NEW")
    .to("direct:insertRegistryRecord")
    .to("direct:resolveRecord")
    .to("direct:resolveFile")
    .to("direct:markDone");
```

**Transaction timeout:**

```properties
# Per-document timeout (nie celá dávka)
quarkus.transaction-manager.default-transaction-timeout=120
```

120s = dostatok pre worst case 30s Insert + Resolve + buffer. Ak NASES odpovedá za 3s, transakcia trvá ~5s.

### 3.5 Komponent 4: Odolné error handling

**Účel**: Inteligentná reakcia na rôzne typy chýb.

```java
// === S312RouteConfiguration.java — nový error handling ===

// 5xx a timeouty → retry s backoffom
onException(IOException.class, HttpOperationFailedException.class,
            SocketTimeoutException.class)
    .onWhen(exchange -> {
        Exception e = exchange.getProperty(Exchange.EXCEPTION_CAUGHT, Exception.class);
        if (e instanceof HttpOperationFailedException) {
            int status = ((HttpOperationFailedException) e).getStatusCode();
            return status >= 500 || status == 429;
        }
        return true; // IOException, timeout → vždy retry
    })
    .handled(true)
    .maximumRedeliveries(5)
    .redeliveryDelay(2000)
    .backOffMultiplier(2.0)
    .maximumRedeliveryDelay(60000)
    .retryAttemptedLogLevel(LoggingLevel.WARN)
    .process(exchange -> {
        // Notifikuj AdaptiveThreadPool o chybe
        adaptiveThreadPool.onSlowOrError(0, true);
    })
    .to("direct:markFailedRetryable");

// 4xx (okrem 429) → permanentný fail, žiadny retry
onException(HttpOperationFailedException.class)
    .onWhen(exchange -> {
        HttpOperationFailedException e = exchange.getProperty(
            Exchange.EXCEPTION_CAUGHT, HttpOperationFailedException.class);
        int status = e.getStatusCode();
        return status >= 400 && status < 500 && status != 429;
    })
    .handled(true)
    .maximumRedeliveries(0)
    .log(LoggingLevel.ERROR,
         "Permanent failure doc=${exchangeProperty.DOCUMENT_ID}: "
         + "${exception.statusCode} ${exception.message}")
    .to("direct:markFailedPermanent");
```

### 3.6 Komponent 5: Circuit breaker

**Účel**: Ak NASES je preťažený, zastaviť spracovanie namiesto zaplavenia chybovými requestami.

```java
// === V S312 route, obalenie IS AR volania ===
from("direct:insertWithResilience")
    .circuitBreaker()
        .resilience4jConfiguration()
            .failureRateThreshold(50)         // 50% zlyhaní → otvor circuit
            .slowCallRateThreshold(80)        // 80% pomalých → otvor circuit
            .slowCallDurationThreshold(30000) // pomalé = >30s
            .waitDurationInOpenState(30000)   // čakaj 30s v otvorenom stave
            .slidingWindowSize(20)            // okno 20 volaní
            .minimumNumberOfCalls(5)          // min 5 pred vyhodnotením
            .permittedNumberOfCallsInHalfOpenState(3) // 3 testovacie volania
        .end()
        .to("direct:insertRegistryRecord")
    .onFallback()
        .log(LoggingLevel.WARN,
             "Circuit OPEN — NASES preťažený. Doc ${exchangeProperty.DOCUMENT_ID} "
             + "vrátený do queue.")
        .process(exchange -> {
            S312Document doc = exchange.getProperty("DOCUMENT", S312Document.class);
            doc.setProcessingNode(null);  // uvoľniť claim
            doc.setClaimedAt(null);
        })
        .to("jpa:S312Document")
    .end();
```

**Závislosť**: Quarkus Resilience4j extension (`quarkus-smallrye-fault-tolerance`).

### 3.7 Komponent 6: Adaptívny paralelizmus

**Účel**: Automatické prispôsobenie počtu vlákien podľa aktuálneho stavu NASES.

```java
@ApplicationScoped
public class AdaptiveThreadPool {

    private final AtomicInteger activeThreads = new AtomicInteger(3);
    private final AtomicLong lastScaleUp = new AtomicLong(0);

    @ConfigProperty(name = "batch.threads.min", defaultValue = "1")
    int minThreads;

    @ConfigProperty(name = "batch.threads.max", defaultValue = "20")
    int maxThreads;

    @ConfigProperty(name = "batch.threads.scaleup.cooldown.ms", defaultValue = "10000")
    long scaleUpCooldownMs;

    public void onSuccess(long responseTimeMs) {
        long now = System.currentTimeMillis();
        if (responseTimeMs < 5000
                && activeThreads.get() < maxThreads
                && now - lastScaleUp.get() > scaleUpCooldownMs) {
            activeThreads.incrementAndGet();
            lastScaleUp.set(now);
            log.info("NASES fast ({}ms), threads → {}", responseTimeMs, activeThreads.get());
        }
    }

    public void onSlowOrError(long responseTimeMs, boolean isError) {
        if (isError) {
            int newCount = Math.max(minThreads, activeThreads.get() / 2);
            activeThreads.set(newCount);
            log.warn("NASES ERROR, threads → {}", newCount);
        } else if (responseTimeMs > 20000) {
            int newCount = Math.max(minThreads, activeThreads.get() - 1);
            activeThreads.set(newCount);
            log.warn("NASES SLOW ({}ms), threads → {}", responseTimeMs, newCount);
        }
    }

    public int getActiveThreads() {
        return activeThreads.get();
    }
}
```

### 3.8 Komponent 7: Fix F-014 (double write)

```java
// === S312RouteConfiguration.java:168-185 ===
// NAHRADIŤ .filter() ZA .choice():

.choice()
    .when(simple("${body.getFailedAttempts()} > ${bean:configurationBean.maximumRetries()}"))
        .to(DIRECT_MOVE_FAILED_DOCUMENT)
        .transform(Body.state(S312DocumentState.FAILED_DATA_ERROR))
        .to(JPA_DOCUMENT)
    .otherwise()
        .transform(Body.increaseFailedAttempt())
        .transform(simple("${body.errorText(...)}"))
        .to(JPA_DOCUMENT)
.endChoice()

// ROVNAKO pre riadky 228-246 (S312ActiveAPIRetry route)
```

### 3.9 Komponent 8: Paginácia S38 (F-013 fix)

```java
// === Scenario38ImportCSV.java:162 ===
// NAHRADIŤ jednorazový load za paginated loop:

from("timer:s38poll?period={{env:S38_POLL_INTERVAL:30000}}")
    .loopDoWhile(simple("${body.size()} > 0"))
        .to("jpa:S38Document?namedQuery=selectForProcessing"
            + "&maximumResults={{env:S38_PAGE_SIZE:50}}")
        .split(body())
            .to("direct:processS38Document")
        .end()
    .end();
```

### 3.10 Konfigurácia (environment variables)

```properties
# === Paralelizmus ===
batch.threads.min=1
batch.threads.max=20
batch.threads.initial=3
batch.threads.scaleup.cooldown.ms=10000

# === Retry ===
MAXIMUM_REDELIVERIES=5
DELAY_PATTERN_ONREDELIVERY_ACTIVEREG=1:2000;2:4000;3:8000;4:16000;5:32000
MAXIMUM_RETRIES=5

# === Transaction ===
quarkus.transaction-manager.default-transaction-timeout=120

# === Connection pool ===
quarkus.datasource.jdbc.max-size=50
quarkus.datasource.jdbc.acquisition-timeout=120

# === Circuit breaker ===
cb.failure-rate-threshold=50
cb.slow-call-duration-threshold=30000
cb.wait-duration-open-state=30000

# === Paginácia ===
S38_PAGE_SIZE=50
S38_POLL_INTERVAL=30000
SEALING_BATCH_SIZE=50
```

## 4. Implementačný plán

### Fáza A: Stabilizácia (hotfix, 1-3 dni)

| # | Komponent | Čo zmeniť | Súbory | Finding |
|---|-----------|-----------|--------|---------|
| A1 | 1 (locking) | @Version na EntityBase | EntityBase.java + DB migrácia | F-017 |
| A2 | 1 (locking) | OnException(OptimisticLockException) | JpaRouteConfiguration.java | F-017 |
| A3 | 3 (tx) | Transaction timeout 120s | application.properties | F-016 |
| A4 | 7 (double write) | choice() namiesto filter() | S312RouteConfiguration.java:168-185, 228-246 | F-014 |
| A5 | — | Znížiť vlákna na 3-5 | Konfig / env variable | F-017 |

**Výsledok Fázy A**: Dávky fungujú stabilne. ~10-20K VD/deň.

### Fáza B: Odolnosť (1-2 týždne)

| # | Komponent | Čo zmeniť | Nové súbory |
|---|-----------|-----------|-------------|
| B1 | 2 (claim) | processing_node + claimed_at + claim query | DB migrácia + úprava route |
| B2 | 2 (claim) | Stale claim cleanup scheduler | Nový @Scheduled bean |
| B3 | 3 (tx) | PROPAGATION_REQUIRES_NEW per dokument | S312RouteConfiguration.java |
| B4 | 4 (error) | Klasifikácia chýb (5xx retry, 4xx fail) | S312RouteConfiguration.java |
| B5 | 4 (error) | Exponenciálny backoff | S312RouteConfiguration.java |
| B6 | 5 (circuit) | Circuit breaker na IS AR volania | S312RouteConfiguration.java |

**Výsledok Fázy B**: Odolný pipeline. ~20-40K VD/deň.

### Fáza C: Škálovanie (2-4 týždne)

| # | Komponent | Čo zmeniť |
|---|-----------|-----------|
| C1 | 6 (adaptive) | Adaptívny thread pool | 
| C2 | 8 (pagination) | Paginácia S38 |
| C3 | — | Monitoring (Micrometer metriky) |
| C4 | — | Load test na PROD-like prostredí |

**Výsledok Fázy C**: ~50-100K VD/deň (závisí od NASES PROD).

## 5. Výpočet priepustnosti

### Worst case (NASES: 30s, počítame konzervatívne)

| Fáza | Vlákna | VD/deň (16h) |
|------|:------:|:------------:|
| A (hotfix) | 3-5 | 5 760–9 600 |
| B (odolnosť) | 10 | 19 200 |
| C (škálovanie) | 15-20 | 28 800–38 400 |

### Realistický (NASES: priemer ~12s)

| Fáza | Vlákna | VD/deň (16h) |
|------|:------:|:------------:|
| A | 3-5 | 14 400–24 000 |
| B | 10 | 48 000 |
| C | 15-20 | 72 000–**96 000** |

### Best case (NASES PROD: priemer ~5s)

| Fáza | Vlákna | VD/deň (16h) |
|------|:------:|:------------:|
| A | 3-5 | 17 280–28 800 |
| B | 10 | 57 600 |
| C | 15-20 | 86 400–**115 200** |

## 6. Možné prvky budúceho zlepšenia

Tieto nie sú súčasťou aktuálneho návrhu. Ak sa stanú dostupnými, dramaticky zlepšia priepustnosť:

| Prvok | Potenciálny efekt | Závislosť |
|-------|-------------------|-----------|
| **Batch sealing API** | 10-50x zrýchlenie pečatenia | Nuaktiv/NASES implementácia |
| **OpenKM → S3 migrácia** | Insert z 3s pod 1s | Infraštruktúra |
| **Async pečatenie** (zmena API) | Oddelenie bottlenecku | Nuaktiv zmena Insert API |
| **Zistenie kapacity CEP na PROD** | Presnejšie nastavenie vlákien | Meranie |
| **Dedikovaný NASES tenant** | Stabilnejšie response times | NASES |

## 7. Otvorené otázky

1. **Aký je reálny výkon NASES na PROD?** (3-30s z TEST je príliš široký rozsah)
2. **Koľko paralelných requestov NASES zvládne na PROD?** (Kde je hranica stability?)
3. **Existuje SLA od NASES na response time?**
4. **Je akceptovateľné 24h spracovanie namiesto 16h?** (Zvyšuje priepustnosť o 50%)
