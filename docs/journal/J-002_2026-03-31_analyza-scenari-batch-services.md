---
id: J-002
date: 2026-03-31
status: draft
participants: [roman, claude]
---

# J-002 — Analýza batch-services-integration: Scenario311, Scenario312, Scenario38

## Čo sa riešilo

Stiahnutie a analýza skutočného produkčného repozitára `batch-services-integration`
(`https://git.isreg.gr8it.cloud/isreg/integration/batch-services-integration`).

Analyzované súbory:
- `scenario/Scenario311Notify.java` + `config/S312RouteConfiguration.java`
- `scenario/Scenario312BatchHandling.java` + `step/S312InsertRegistryRecordOutboundAndProcessStep.java` + `step/S312ResolveRegistryRecordStep.java` + `step/S312CloseBatchWhenReadyStep.java`
- `scenario/Scenario38ImportCSV.java` + `config/S38RouteConfiguration.java` + `step/S38CloseBatchWhenReadyStep.java`

## Dôležitý kontext: zmena zdroja kódu

Predošlá session (J-001) analyzovala repo `JVP-nedoplatky` — pomocný zdroj s kódom IS REG.
**Táto session analyzuje skutočný produkčný kód**: `batch-services-integration`.
Staré findingy F-001 až F-008 boli z iného repozitára (iné názvovanie tried: Scenario351Notify, Scenario352Debt).
**Status F-001 až F-008: neaplikovateľné na produkčný kód** — treba prehodnotiť alebo archivovať.

## Identifikované problémy (táto session)

### KRITICKÉ

#### F-009 — S311 onException: chýba endChoice() → HTTP 200 na chybe
**Súbor**: `Scenario311Notify.java:46-59`
**Problém**: V `onException(Response.class)` bloku je `choice().when(variable(DOCUMENT).isNotNull())` bez `endChoice()` alebo `otherwise()`. Camel DSL interpretuje `removeHeaders()`, `setHeader(500)`, `setBody(null)` ako VNÚTRI `when()` bloku — nie za ním. Ak výnimka nastane pred nastavením premennej DOCUMENT (pred riadkom 84), HTTP 500 sa NENASTAVÍ. Volajúci dostane HTTP 200 OK, akoby request prebehol úspešne.

**Dopad**: ÚPVS si myslí, že notifikácia bola prijatá — neretryuje. Dáta sa strácajú ticho.

#### F-010 — S38RouteConfiguration: Batch filter NESTED vo File filter → Batch nikdy FAILED
**Súbor**: `S38RouteConfiguration.java:131-148`
**Problém**: V `S38EntityBasedUnrecoverableErrorConfiguration` handler-i chýba `.end()` po riadku 137 (za File JPA persist). Camel DSL teda interpreuje `.filter(Constant.BATCH...)` (riadok 140) ako VNORENÝ do `.filter(Constant.FILE...)` (riadok 131). Výsledok: Batch state FAILED sa nastavuje IBA ak existuje aj FILE premenná. Pri bežných chybách S38Document (bez FILE kontextu) batch NIKDY nedostane stav FAILED.

**Dopad**: Dávky zostávajú v stave "processing" navždy. Monitoring a uzavierka dávok nefungujú.

#### F-011 — S312InsertStep: NPE pri prázdnych RegistryRecords v odpovedi IS AR
**Súbor**: `S312InsertRegistryRecordOutboundAndProcessStep.java:172`
**Problém**: `response.getRegistryRecords().getFirst()` — bez null-checku ani isEmpty() checku. Ak IS AR vráti odpoveď s prázdnym alebo null zoznamom RegistryRecords (napr. pri internej chybe IS AR), nastane NPE. Camel to zachytí ako RuntimeException (chyba 6. typu) a dokument zlyhá, ale bez jasnej diagnostiky.

**Dopad**: Neprehľadné zlyhanie dokumentu, ťažká diagnostika.

### VYSOKÉ RIZIKO

#### F-012 — S312InsertStep: IOException pri načítaní prílohy sa ticho prehltne
**Súbor**: `S312InsertRegistryRecordOutboundAndProcessStep.java:98-107`
**Problém**: Ak sa príloha nedá prečítať zo súborového systému, IOException je zachytená, zalogovaná ako WARN a príloha je VYNECHANÁ. Dokument sa odošle do IS AR bez prílohy. Žiadna výnimka, žiadna zmena stavu.

**Dopad**: Tichá strata dát — dokument zaevidovaný v registratúre bez prílohy. Príjemca dostane neúplný dokument.

#### F-013 — S38: Žiadna paginácia pri načítaní S38Document — OOM riziko
**Súbor**: `Scenario38ImportCSV.java:162`
**Problém**: `to("jpa:S38Document?namedQuery=selectS38DocumentForProcessing")` bez `setMaxResults`. Načíta VŠETKY dokumenty na spracovanie do pamäte naraz. Scenario312 toto rieši cez `loopDoWhile` + `setMaxResults(configurationBean.maximumReadSize())`. S38 nemá ekvivalent.

**Dopad**: Pri väčšom počte EESSI záznamov → OutOfMemoryError, pád aplikácie.

#### F-014 — S312RouteConfiguration chyba 4: double JPA write pri max retry
**Súbor**: `S312RouteConfiguration.java:168-185`
**Problém**: Keď `failedAttempts > maximumRetries`, filter zachytí a spustí `moveFailed + FAILED_DATA_ERROR`. Ale po `.end()` filtra Camel POKRAČUJE a spustí aj `increaseFailedAttempt() + errorText + JPA persist` — pre VŠETKY exchanges (filter v Camel nestopuje exchange). Dokument sa zapíše do DB dvakrát — raz pri max-retry branchi, raz za filtrom.

**Dopad**: Dvojitý DB write, failedAttempts = maximumRetries+1 (nie maximumRetries), errorText môže byť prepísaný nesprávnou hodnotou z chyby 4. Rovnaký vzor existuje aj v `S312ActiveAPIRetry` route (riadky 224-246).

### STREDNÉ RIZIKO

#### F-015 — S38: Hardcoded `threads(10, 10)` — nekonfigurovateľné
**Súbor**: `Scenario38ImportCSV.java:165`
**Problém**: `threads(10, 10)` je natvrdo zakódovaných. Scenario312 používa `configurationBean.threads()` — konfigurovateľné cez env var. S38 nemá analogické nastavenie.

**Dopad**: Nie je možné optimalizovať thread count podľa prostredia (UAT vs PROD) bez code change.

#### F-016 — S311: Path traversal — priznaná bezpečnostná zraniteľnosť
**Súbor**: `Scenario311Notify.java:28` (Javadoc)
**Problém**: Javadoc triedy explicitne uvádza: "Ma kriticku zranitelnost, dokaze prepisat potencialne lubovolny subor fs, na ktory ma pravo a to aj mimo cieloveho adresara." Zraniteľnosť je známa, ale kód nie je opravený.

**Dopad**: Bezpečnostný audit — pri HTTP POST na `/notify` s manipulovaným payloadom môže útočník prepísať ľubovoľný súbor na serveri. V kontexte UAT/PROD je expozícia limitovaná, ale riziko existuje.

## Ďalšie pozorovania

### ActiveRegException code 0 — tiché zlyhanie
**Súbor**: `S312RouteConfiguration.java:193-221` a `S38RouteConfiguration.java:67-106`
V oboch route konfiguráciách ActiveREG fault handler pokrýva len `code == 99` a `code > 0`. Ak IS AR vráti `code == 0` alebo záporný kód: `handled(true)` prehltne výnimku, zaloguje sa ERROR, ale žiaden DB update sa nespustí. Dokument ostáva v pôvodnom stave bez záznamu chyby.

### S312CloseBatchWhenReadyStep chýba @ApplicationScoped
**Súbor**: `step/S312CloseBatchWhenReadyStep.java:35`
Trieda nemá `@ApplicationScoped` anotáciu, na rozdiel od `S38CloseBatchWhenReadyStep` (riadok 37). Quarkus Camel extension dokáže RouteBuilder-y discoveriť aj bez explicitnej CDI scope anotácie, takže toto PRAVDEPODOBNE nie je runtime bug. Ale je to nekonzistentnosť, ktorá stojí za overenie.

### seda queue uzavierky dávok bez size limitu (S311)
**Súbor**: `Scenario311Notify.java:209`
`seda:closeBatchWhenReady?waitForTaskToComplete=Never` — pri vysokom objeme notifikácií sa queue môže nekontrolovane nafúknuť. Consumer `threads(1,1)` limituje paralelizmus, ale nie rast fronty.

## Záver

**Číslo potvrdených nových bugov: 6 (F-009 až F-014)**
**Kritické: 3 (F-009, F-010, F-011)**
**Vysoké riziko: 3 (F-012, F-013, F-014)**

Staré findingy F-001 až F-008 boli z JVP-nedoplatky repo — nie z produkčného kódu.
Odporúčam archivovať alebo explicitne označiť ako "iný zdroj".

## Ďalšie kroky

1. Roman overí F-009 (onException scope) — triviálne reprodukovateľné
2. Roman overí F-010 (Batch nested filter) — kritické pre S38
3. Roman overí F-011 a F-012 (NPE a attachment loss) — pri S312
4. Rozhodnutie: archivovať F-001 až F-008 alebo remapovať na nový kód?
