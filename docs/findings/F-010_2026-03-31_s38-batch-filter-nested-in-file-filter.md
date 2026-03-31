---
id: F-010
date: 2026-03-31
status: fixed
severity: critical
scenario: S38
fixed_in_commit: eabc3f5
fixed_date: 2026-03-31
---

# F-010 — S38RouteConfiguration: Batch filter je VNORENÝ vo File filter → Batch nikdy FAILED

## Súbor
`src/main/java/sk/socpoist/isreg/regint/batchservices/config/S38RouteConfiguration.java:131-148`

## Popis

V `S38EntityBasedUnrecoverableErrorConfiguration` handler-i chýba `.end()` po File JPA persist:

```java
// Document filter — správne, má .end() na riadku 128
.filter(variable(Constant.DOCUMENT).isInstanceOf(EntityBase.class))
    ...
    .to(JPA_ENTITY_BASE)
.end() // filter — riadok 128 ✓

// File filter — PROBLÉM
.filter(variable(Constant.FILE).isInstanceOf(EntityBase.class)) // riadok 131 — scope A otvorený
    .setBody(variable(Constant.FILE))
    .transform(...)
    .transform(Body.state(CommonDocumentState.FAILED))
    .to(JPA_ENTITY_BASE)  // riadok 137
    // ← CHÝBA .end() pre scope A
    
    // Batch filter — NEÚMYSELNE VNORENÝ v scope A
    .filter(variable(Constant.BATCH).isNotNull())  // riadok 140 — scope B vnorený
        .setBody(variable(Constant.BATCH))
        .transform(Body.state(CommonDocumentState.FAILED))
        .transform(...)
        .to("jpa:BatchBase")
    .end()  // riadok 147 — zatvára scope B (Batch filter)
    .stop() // riadok 148 — VNÚTRI scope A (File filter) !!!
// scope A nikdy nie je zatvorený
```

## Dopad

1. **Batch NIKDY nedostane stav FAILED** keď zlyhá iba S38Document (FILE premenná je null):
   - Batch ostáva v stave "processing" navždy
   - `closeCompleteBatches` ho nikdy neuzavrie (čaká na FAILED alebo DONE dokumenty)
   - Monitoring vidí batch ako "otvorený" — nekonečne

2. **`.stop()` sa nevykoná** keď FILE je null:
   - Exchange pokračuje po exception handler-i
   - Správanie závisí od Camel internals — potenciálne ďalšie neočakávané spracovanie

## Fix

Pridať `.end()` po riadku 137 (File JPA persist):

```java
.filter(variable(Constant.FILE).isInstanceOf(EntityBase.class))
    .setBody(variable(Constant.FILE))
    .transform(...)
    .transform(Body.state(CommonDocumentState.FAILED))
    .to(JPA_ENTITY_BASE)
.end() // ← TOTO CHÝBA — zatvoriť File filter

.filter(variable(Constant.BATCH).isNotNull())
    ...
.end()
.stop();
```

## Verifikácia

Roman: Skontrolovať UAT logy — existujú dávky S38 v stave IN_PROGRESS/otvorenom stave dlhšie ako 1 deň?

## Fix — APLIKOVANÝ

Commit `eabc3f5` (2026-03-31) pridal `.end() // filter file` na riadok 140.
Batch filter je teraz na správnej úrovni — nie vnorený vo File filter.

```java
.to(JPA_ENTITY_BASE)
.end() // filter file  ← PRIDANÉ

// Batch — teraz správne na top-level
.filter(variable(Constant.BATCH).isNotNull())
    ...
.end() // filter batch
.stop();
```

Rovnaký commit tiež opravil double-JPA-write v `DIRECT_RETRYABLE_DOCUMENT_ERROR`
(`increaseFailedAttempt()` sa volá pred max-retries filtrom, nie za ním).
