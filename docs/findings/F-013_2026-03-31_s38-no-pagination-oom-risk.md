---
id: F-013
date: 2026-03-31
status: identified
severity: high
scenario: S38
---

# F-013 — S38: Žiadna paginácia pri načítaní dokumentov → OOM riziko

## Súbor
`src/main/java/sk/socpoist/isreg/regint/batchservices/scenario/Scenario38ImportCSV.java:162`

## Popis

```java
// S38 — načíta VŠETKO naraz
.to("jpa:S38Document?namedQuery=selectS38DocumentForProcessing")
.split(body()).parallelProcessing().threads(10, 10)
```

Porovnaj so S312 (správny prístup):
```java
// S312 — stránkuje po BATCH_SIZE kusoch
.loopDoWhile(constant(true))
    .process(em -> em.createNamedQuery(...).setMaxResults(configurationBean.maximumReadSize())...)
    .filter(body.size == 0).stop()
    .split(body()).parallelProcessing().threads(N, N)
```

S38 nemá žiadnu ekvivalentnú pagináciu. Celý dataset sa načíta do pamäte pred spracovaním.

## Dopad

- Pre malé EESSI dávky (desiatky záznamov): žiaden problém
- Pre väčšie dávky (stovky-tisíce záznamov): `OutOfMemoryError` alebo GC pressure
- Pád aplikácie pri OOM = nespracované záznamy, ručné opätovné spustenie

## Fix

Pridať ekvivalentný `loopDoWhile` + `setMaxResults` pattern ako v S312:

```java
.loopDoWhile(constant(true))
    .process(exchange -> {
        try (var em = entityManagerFactory.createEntityManager()) {
            exchange.getMessage().setBody(
                em.createNamedQuery("selectS38DocumentForProcessing", S38Document.class)
                  .setMaxResults(configurationBean.maximumReadSize())
                  .getResultList());
        }
    })
    .filter(simple("${body.size} == 0")).stop().end()
    .split(body()).parallelProcessing().threads(...)
    ...
```

## Verifikácia

Roman: Aký je typický počet EESSI záznamov na dávku v UAT? Je toto realistický problém teraz?
