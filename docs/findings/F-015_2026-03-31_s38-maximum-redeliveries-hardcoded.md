---
id: F-015
date: 2026-03-31
status: identified
severity: medium
scenario: S38
introduced_in_commit: eabc3f5
---

# F-015 — S38RouteConfiguration: maximumRedeliveries hardcoded na 3

## Súbor
`src/main/java/sk/socpoist/isreg/regint/batchservices/config/S38RouteConfiguration.java:60`

## Popis

Commit `eabc3f5` (fix F-010) zmenil `maximumRedeliveries` z konfigurovateľnej env premennej
na hardcoded hodnotu:

**Pred:**
```java
.maximumRedeliveries("{{env:MAXIMUM_REDELIVERIES:0}}")
.delayPattern("{{env:DELAY_PATTERN_ONREDELIVERY_ACTIVEREG:1:10000;2:60000;3:300000}}")
```

**Po:**
```java
.maximumRedeliveries(3)
.delayPattern("1:10000;2:30000;3:60000")
```

## Dopad

- Nie je možné zmeniť počet redeliveries bez code change + deployment
- delay pattern je tiež natvrdo zakódovaný (a zmenený — 60s → 30s na 2. pokus, 300s → 60s na 3. pokus)
- S312 má toto konfigurovateľné cez `configurationBean.maximumRedeliveries()` — nekonzistentnosť

## Poznámka

Toto je side-effect opravy F-010. Samotná logika retry je funkčná (3 pokusy),
ale strata konfigurovateľnosti je regres oproti predošlému stavu.

## Odporúčanie

Použiť `configurationBean.maximumRedeliveries()` a `configurationBean.delayPattern()`
analogicky ako v S312RouteConfiguration.
