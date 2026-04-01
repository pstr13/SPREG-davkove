---
id: F-017
date: 2026-04-01
slug: s312-race-condition-duplicate-processing
title: "S312: Race condition — duálne spracovanie toho istého záznamu výstupných dokumentov"
severity: critical
status: identified
scenario: S312 (Scenario312BatchHandling)
source: roman (developer) + code-analysis
reporter: roman
assignee: roman
related_bugs:
  - SPISREG-2690
  - SPISREG-2688
  - SPISREG-2695
  - SPISREG-1517
related_findings:
  - F-014
---

# F-017 — S312: Race condition — duálne spracovanie toho istého záznamu

## Symptóm

Pri hromadnom spracovaní výstupných dokumentov (scenár 3.1.2) dochádza k:
1. **Duálnemu spracovaniu** toho istého záznamu — dva thready spracúvajú rovnaký dokument
2. Jeden uspeje, druhý neuspeje
3. **Vzájomne si prepíšu výsledok** — posledný write vyhráva (last-write-wins)
4. V niektorých prípadoch vznikne aj správny záznam (keď "úspešný" thread zapíše posledný)

Pravdepodobne zodpovedá za SPISREG-2690 (MEGA BUG).

## Root Cause

### Úplná absencia concurrency control v S312 pipeline

Analýza zdrojového kódu odhalila, že **žiadna vrstva** batch spracovania neimplementuje ochranu proti súbežnému prístupu:

### 1. Žiadny optimistický locking (@Version)

```java
// EntityBase.java — base class pre všetky entity
// CHÝBA:
@Version
private long version;  // ← toto neexistuje
```

Bez `@Version` JPA **nikdy nedeteguje** concurrent update. Dva thready môžu čítať rovnaký záznam, oba ho modifikovať, a posledný zápis ticho prepíše prvý.

### 2. Žiadny pesimistický locking (SELECT FOR UPDATE)

```java
// Named query pre batch dokumenty — žiadny locking
@NamedQuery(
    name = "findByStateAndScenario",
    query = "SELECT s FROM Submission s WHERE s.state = :state AND s.scenario = :scenario")
// CHÝBA: javax.persistence.lock.timeout, LockModeType.PESSIMISTIC_WRITE
```

Keď S312 route načíta dokumenty na spracovanie, query vráti výsledky **bez row-level locku**. Iný thread/route môže v tom istom momente čítať tie isté záznamy.

### 3. Žiadna deduplikácia pri split()

```java
// Typický Camel pattern v batch processing
from("jpa:Document?namedQuery=selectForProcessing")
    .split(body())      // ← rozdelí výsledky na paralelné spracovanie
    .threads(N, N)      // ← N threadov
    // ... spracovanie
```

Camel `split()` rozdelí resultset medzi thready, ale **neposkytuje garanciu**, že ten istý záznam nebude vybraný aj ďalšou iteráciou `loopDoWhile` kým prvý thread ešte nezmenil stav.

### 4. Race condition scenár

```
Čas    Thread A                    Thread B                    DB stav
─────────────────────────────────────────────────────────────────────────
T1     SELECT docs WHERE           SELECT docs WHERE           doc.state = NEW
       state = 'NEW'               state = 'NEW'
       → vráti doc #42             → vráti doc #42

T2     Spracuj doc #42             Spracuj doc #42             doc.state = NEW
       → volaj IS AR               → volaj IS AR               (obe čítajú NEW)
       (insert registry record)    (insert registry record)

T3     IS AR → OK                  IS AR → FAIL                doc.state = NEW
       (záznam vytvorený)          (duplikát? timeout?)

T4     UPDATE doc #42              UPDATE doc #42              
       state = DONE                state = FAILED              
       registryRecordId = 123      errorText = "..."           

T5                                 COMMIT → state = FAILED     doc.state = FAILED !!!
       COMMIT → OptimisticLock?    ← BEZ @Version NIE!        (thread B prepísal A)
       → TICHO PREPÍŠE!!!         
                                                                doc.state = DONE ← posledný
                                                                ALE: záznam v IS AR existuje
                                                                a nie je prepojený!
```

### 5. Submission tabuľka — žiadne concurrency polia

```sql
CREATE TABLE submission (
    submission_id    bigint identity,
    state            varchar(50),     -- menený bez lockingu
    error_text       varchar(max),
    created          datetime2,
    modified         datetime2,       -- audit trigger, NIE version control
    -- CHÝBA: version bigint
    -- CHÝBA: processing_thread varchar
    -- CHÝBA: locked_at datetime
);
```

`modified` trigger je čisto audit — **neposkytuje** conflict detection.

## Dopad

| Scenár | Výsledok | Frekvencia |
|--------|----------|------------|
| Thread A = OK, Thread B = FAIL, B zapíše posledný | Záznam v IS AR existuje, ale dokument je FAILED | Častý pri väčších dávkach |
| Thread A = OK, Thread B = OK (duplikát) | Dva záznamy v IS AR pre ten istý dokument | Menej častý, závisí od IS AR idempotency |
| Thread A = FAIL, Thread B = OK, A zapíše posledný | Dokument FAILED, ale záznam v IS AR existuje "osirotený" | Najhorší scenár — inconsistentný stav |
| Obaja OK, obaja zapíšu | Posledný write vyhráva, potenciálne iné registryRecordId | Dátová korupcia |

**Korelácia s bugmi:**
- **SPISREG-2690 (MEGA BUG)**: Veľmi pravdepodobne tento scenár
- **SPISREG-2688/2695**: Spisy "V riešení" — mohlo byť spôsobené prepísaním stavu
- **SPISREG-1517**: Chýbajúce čísla záznamov — thread A zapísal číslo, thread B ho prepísal

## Navrhované riešenie

### Variant A: Optimistický locking (ODPORÚČANÝ — najmenší zásah)

**Krok 1**: Pridať `@Version` do `EntityBase` (base class):

```java
// EntityBase.java
@Version
@Column(name = "version")
private long version;
```

**Krok 2**: ALTER TABLE pre všetky relevantné tabuľky:

```sql
ALTER TABLE submission ADD version BIGINT NOT NULL DEFAULT 0;
ALTER TABLE s312_document ADD version BIGINT NOT NULL DEFAULT 0;
-- ... pre každú entity tabuľku
```

**Krok 3**: Ošetriť `OptimisticLockException` v Camel route:

```java
// V JpaRouteConfiguration alebo S312RouteConfiguration
onException(OptimisticLockException.class)
    .handled(true)
    .maximumRedeliveries(3)
    .delay(1000)  // krátky delay, potom retry
    .log("Optimistic lock conflict on ${body.id}, retrying...")
    .to("direct:reprocessDocument");
```

**Výhoda**: JPA automaticky deteguje concurrent update → výnimka → retry.
**Nevýhoda**: Retry pridáva latenciu. Ale konflikt nastane len ak dva thready naozaj spracúvajú rovnaký záznam.

### Variant B: Pesimistický locking (SELECT FOR UPDATE)

**Krok 1**: Upraviť named query pre batch loading:

```java
@NamedQuery(
    name = "selectS312DocumentForProcessing",
    query = "SELECT d FROM S312Document d WHERE d.state = :state",
    hints = @QueryHint(name = "javax.persistence.lock.mode", value = "PESSIMISTIC_WRITE")
)
```

Alebo v Camel JPA consumer:

```java
from("jpa:S312Document?namedQuery=selectForProcessing&lockModeType=PESSIMISTIC_WRITE")
```

**Výhoda**: Garantuje exkluzívny prístup — žiadne duplikáty.
**Nevýhoda**: Znižuje paralelizmus. Potenciálne deadlocky ak transakcie zamykajú v rôznom poradí.

### Variant C: Claim check pattern (NAJROBUSTNEJŠÍ)

**Krok 1**: Pridať `processing_thread` a `locked_at` do entity:

```sql
ALTER TABLE s312_document ADD processing_thread VARCHAR(100) NULL;
ALTER TABLE s312_document ADD locked_at DATETIME2 NULL;
```

**Krok 2**: Atomický claim pri načítaní:

```sql
UPDATE s312_document 
SET processing_thread = :threadId, locked_at = GETDATE()
WHERE state = 'NEW' AND processing_thread IS NULL
AND id IN (SELECT TOP :batchSize id FROM s312_document WHERE state = 'NEW' AND processing_thread IS NULL)
```

**Krok 3**: Spracovanie len claimnutých záznamov:

```sql
SELECT d FROM S312Document d 
WHERE d.processingThread = :threadId AND d.state = 'NEW'
```

**Krok 4**: Cleanup pre stuck záznamy (locked_at > 30 min):

```sql
UPDATE s312_document SET processing_thread = NULL, locked_at = NULL
WHERE locked_at < DATEADD(MINUTE, -30, GETDATE())
```

**Výhoda**: Žiadne duplikáty, žiadne deadlocky, jasný audit trail.
**Nevýhoda**: Najväčší implementačný zásah.

### Variant D: Zníženie paralelizmu (DOČASNÝ WORKAROUND)

```java
// Namiesto paralelného spracovania:
threads(10, 10)

// Znížiť na 1 thread:
threads(1, 1)
```

**Výhoda**: Eliminuje race condition úplne.
**Nevýhoda**: Výrazne pomalšie spracovanie. Len ako dočasné opatrenie.

## Odporúčanie

**Okamžite (hotfix)**: **Variant D** — znížiť thready na 1 pre S312. Eliminuje race condition bez zmeny DB schémy.

**Do produkcie**: **Variant A** (optimistický locking) — najmenší zásah, JPA to rieši automaticky. Pridať `@Version` + ALTER TABLE + ošetriť `OptimisticLockException`.

**Ideálne (ak je čas)**: **Variant C** (claim check) — najrobustnejší, ale najväčší implementačný zásah.

## Dizajn implementácie — Variant A (optimistický locking)

Najmenší zásah s najväčším efektom. Implementácia v 4 krokoch.

### Krok 1: DB migrácia — pridať version stĺpec

Nový Liquibase changeset:

```xml
<!-- db.changelog-add-version-column.xml -->
<changeSet id="add-version-column" author="roman">
    <addColumn tableName="submission">
        <column name="version" type="BIGINT" defaultValueNumeric="0">
            <constraints nullable="false"/>
        </column>
    </addColumn>
    <addColumn tableName="s312_document">
        <column name="version" type="BIGINT" defaultValueNumeric="0">
            <constraints nullable="false"/>
        </column>
    </addColumn>
    <addColumn tableName="s312_batch">
        <column name="version" type="BIGINT" defaultValueNumeric="0">
            <constraints nullable="false"/>
        </column>
    </addColumn>
    <!-- Rovnako pre S311, S38 entity tabuľky -->
</changeSet>
```

### Krok 2: Entity zmena — EntityBase.java

```java
// EntityBase.java (base class pre všetky entity)
package sk.socpoist.isreg.regint.common.database;

import javax.persistence.MappedSuperclass;
import javax.persistence.Version;
import javax.persistence.Column;

@MappedSuperclass
public abstract class EntityBase {

    // ... existujúce polia ...

    @Version
    @Column(name = "version")
    private long version;

    public long getVersion() {
        return version;
    }

    // Setter NEMÁ byť public — JPA manažuje version automaticky
    protected void setVersion(long version) {
        this.version = version;
    }
}
```

**Poznámka**: Ak `EntityBase` je base class pre `DocumentBase`, `BatchBase`, `Submission` atď., stačí pridať `@Version` sem a všetky entity ho zdedia. Ak nie, pridať do každej entity zvlášť.

### Krok 3: Error handling — OptimisticLockException v Camel route

```java
// JpaRouteConfiguration.java — pridať nový exception handler
// PRED existujúce handlery (Level 1, 2, 3)

routeConfiguration("OptimisticLockConfiguration")
    .onException(OptimisticLockException.class, StaleObjectStateException.class)
    .handled(true)
    .maximumRedeliveries(3)
    .redeliveryDelay(500)         // 0.5s delay pred retry
    .retryAttemptedLogLevel(LoggingLevel.WARN)
    .log(LoggingLevel.WARN, 
         "Optimistic lock conflict na dokumente ${exchangeProperty.DOCUMENT_ID}, "
         + "attempt ${header.CamelRedeliveryCounter}/3. Reload a retry.")
    .process(exchange -> {
        // Reload entity z DB pred retry — získa aktuálnu verziu
        Object doc = exchange.getProperty(Constant.DOCUMENT);
        if (doc != null) {
            // EntityManager.refresh() alebo nový SELECT
            // Tým sa načíta aktuálna version z DB
            exchange.getMessage().setBody(doc);
        }
    })
    .to("direct:reprocessDocument");
```

**Kľúčové**: Pri `OptimisticLockException` MUSÍME reloadnúť entitu z DB, lebo stará verzia je stale. Bez reload by retry opäť zlyhal na rovnakej version.

### Krok 4: Idempotency check v IS AR volaniach

Aby sa predišlo duplikátom v IS AR pri retry:

```java
// S312InsertRegistryRecordOutboundAndProcessStep.java
// Pred volaním IS AR skontrolovať či záznam už neexistuje

public void process(Exchange exchange) {
    S312Document doc = exchange.getProperty(Constant.DOCUMENT, S312Document.class);
    
    // Ak dokument už má registry record ID → IS AR bol úspešne zavolaný
    // predchádzajúcim threadom, len DB zápis stavu zlyhal
    if (doc.getRegistryRecordId() != null) {
        log.info("Doc {} already has registryRecordId={}, skipping IS AR call",
                 doc.getId(), doc.getRegistryRecordId());
        // Preskočiť IS AR call, ísť rovno na uloženie stavu
        exchange.getMessage().setHeader("SKIP_ISAR_INSERT", true);
        return;
    }
    
    // ... existujúci kód pre IS AR call ...
}
```

### Sekvenčný diagram — spracovanie s optimistickým lockingom

```
Thread A                       DB                          Thread B
────────                      ────                        ────────
SELECT doc #42                                            SELECT doc #42
(version=0)                                               (version=0)
    │                                                         │
    ▼                                                         ▼
Volaj IS AR                                               Volaj IS AR
→ OK, record #123                                         → OK, record #124 (duplikát!)
    │                                                         │
    ▼                                                         ▼
UPDATE doc #42                                            UPDATE doc #42
SET state=DONE,                                           SET state=DONE,
    version=1                                                 version=1
WHERE id=42                                               WHERE id=42
  AND version=0                                             AND version=0
→ OK (1 row updated)                                      → FAIL! 0 rows updated
    │                                                     → OptimisticLockException
    │                                                         │
    │                                                         ▼
    │                                                     RETRY: reload doc #42
    │                                                     (version=1, state=DONE)
    │                                                         │
    │                                                         ▼
    │                                                     Doc je DONE → skip
    │                                                     (idempotency check)
    ▼                                                         ▼
COMMIT                                                    COMMIT (no-op)
```

### Čo sa zmení v správaní

| Pred fixom | Po fixe |
|-----------|---------|
| Oba thready zapíšu, posledný vyhráva (silent data loss) | Prvý zapíše OK, druhý dostane OptimisticLockException |
| Duplikáty v IS AR nikto nedeteguje | Retry reloadne entitu, zistí že je DONE, preskočí |
| Stav záznamu závisí od timingu threadov | Stav je deterministický — prvý writer vyhráva |
| Žiadne logy o konflikte | WARN log pri každom konflikte → viditeľnosť |

### Rollback plán

Ak `@Version` spôsobí problémy (napr. neočakávané OptimisticLockException v iných scenároch):

1. Odstrániť `@Version` z EntityBase
2. Nechať version stĺpec v DB (nebude mať efekt bez anotácie)
3. Fallback na Variant D (1 thread)

### Testovací plán

1. **Unit test**: Simulovať concurrent update na rovnakej entite — overiť že OptimisticLockException nastane
2. **Integračný test**: 50ks dávka s N threadmi — overiť že žiadny dokument nie je spracovaný duplikátne
3. **Regression test**: Existujúce scenáre (S311, S38, EESSI) — overiť že @Version nespôsobuje false positive konflikty
4. **DB kontrola po teste**: COUNT(registryRecordId) GROUP BY documentId — žiadne duplikáty

## Verifikácia

1. **DB kontrola**: Existujú v DB záznamy kde ten istý dokument má dva rôzne registry record ID? Alebo záznamy kde stav je DONE ale registryRecordId je NULL?
2. **IS AR kontrola**: Existujú duplikátne záznamy v IS AR pre ten istý zdrojový dokument?
3. **Logy**: Hľadať situácie kde dva thready logujú spracovanie rovnakého document ID v rovnakom čase.
4. **Test**: Spustiť 50ks dávku s 1 threadom — ak problém zmizne, potvrdená race condition.
