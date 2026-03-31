---
id: F-011
date: 2026-03-31
status: identified
severity: critical
scenario: S312
---

# F-011 — S312 InsertStep: NPE keď IS AR vráti prázdny zoznam RegistryRecords

## Súbor
`src/main/java/sk/socpoist/isreg/regint/batchservices/step/S312InsertRegistryRecordOutboundAndProcessStep.java:172`

## Popis

```java
var record = response.getRegistryRecords().getFirst();
```

Dvojitý problém:
1. `response.getRegistryRecords()` môže byť `null` → NPE
2. `.getFirst()` na prázdnom zozname → `NoSuchElementException`

Žiaden null-check ani isEmpty() check pred prístupom.

## Kedy nastáva

- IS AR vráti HTTP 200 ale s prázdnym alebo null `registryRecords` zoznamom (napr. interná chyba IS AR, partial response)
- IS AR vráti neočakávanú odpoveď pri zaťažení

## Dopad

- NPE / NoSuchElementException je zachytená ako `RuntimeException` → chyba 6. typu
- S312Document dostane stav `FAILED_DATA_ERROR` s generickou chybou
- Diagnostika je neprehľadná — developer musí dohľadávať stacktrace
- V horšom prípade, ak `EvaluateResponseStep` neprechádza správne, exception handler nemusí byť aktívny

## Fix

```java
var records = response.getRegistryRecords();
if (records == null || records.isEmpty()) {
    throw new RegintException("IS AR returned empty RegistryRecords list");
}
var record = records.getFirst();
```

## Verifikácia

Roman: Overiť čo `EvaluateResponseStep` robí — či zachytáva chybové odpovede pred unmarshal-om.
