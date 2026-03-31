---
id: F-014
date: 2026-03-31
status: identified
severity: high
scenario: S312
---

# F-014 — S312RouteConfiguration: Dvojitý JPA write pri dosiahnutí max retry

## Súbor
`src/main/java/sk/socpoist/isreg/regint/batchservices/config/S312RouteConfiguration.java:168-185`
(a zrkadlovo v riadkoch 228-246 pre S312ActiveAPIRetry route)

## Popis

V `S312EntityBasedUnresponsiveServiceConfiguration` (chyba 4. typu):

```java
.filter(simple("${body.getFailedAttempts()} > ${bean:configurationBean.maximumRetries()}"))
    .to(DIRECT_MOVE_FAILED_DOCUMENT)               // moveFailed
    .transform(Body.state(S312DocumentState.FAILED_DATA_ERROR))
.end() // filter — Camel NEPRESTÁVA exchange, iba skakáme za blok

// Toto sa vykoná PRE VŠETKY exchanges — aj pre max-retry prípad!
.transform(Body.increaseFailedAttempt())  // ← failedAttempts = max+1
.transform(simple("${body.errorText(...)}"))  // ← prepíše errorText
.to(JPA_DOCUMENT)  // ← 2. JPA write (1. bol v moveFailed)
```

V Camel DSL, `.filter(predicate)` pri false predikáte PRESKOČÍ obsah bloku,
ale exchange POKRAČUJE ďalej. Kód za `.end()` filtra sa vykoná pre VŠETKY exchanges.

## Dopad

**Keď max retry je DOSIAHNUTÝ:**
1. `moveFailed()` — presun súboru, zmena stavu
2. `FAILED_DATA_ERROR` — nastavenie stavu (vnútri filtra)
3. `.end()` filtra
4. `increaseFailedAttempt()` — failedAttempts sa zvýši na max+1
5. `errorText` s chybou 4 — prepíše prípadný iný errorText
6. `JPA_DOCUMENT` — 2. write do DB

Výsledok v DB: stav závisí od poradia transformácií, failedAttempts = maximumRetries+1 (nie max), errorText z posledného write.

## Fix

Použiť `.choice()` namiesto `.filter()` pre binárne rozvetvenie:

```java
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
```

## Verifikácia

Roman: Overiť v DB — sú dokumenty s failedAttempts > maximumRetries (napr. > 4 ak max je 3)?
Ak áno, potvrdzuje to double-write.
