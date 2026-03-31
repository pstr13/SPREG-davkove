---
id: F-001
date: 2026-03-31
severity: critical
status: confirmed-by-peter
component: Scenario351Notify.java
lines: [126, 135-136, 165]
verified_by: static-analysis
uat_evidence: none (bug invisible in logs, visible only in DB)
---

# F-001 — Data corruption: registryRecordNumber prepísaný assistedValue

## Popis

V `Scenario351Notify.java` sa do `S351Document.registryRecordNumber` zapíše hodnota `"true"` alebo `"false"` (checkbox Assisted z formulára) namiesto skutočného čísla registratúrneho záznamu.

## Mechanizmus

| Krok | Riadok | Čo sa stane | Hodnota |
|------|--------|-------------|---------|
| 1. zápis | 126-127 | `registryRecordNumber(${variable.registryRecordNumber})` — premenná ešte neexistuje | `null` |
| Naplnenie premennej | 135-136 | Camel variable `registryRecordNumber` dostane správnu hodnotu z JSON | (nezapísané do entity) |
| 2. zápis (PREPIS) | 165, 168 | `document.setRegistryRecordNumber(assistedString)` | `"true"` / `"false"` |

## Dopad

- **DB stĺpec `registry_record_number`** obsahuje `"true"`/`"false"` namiesto skutočného čísla záznamu
- **Korelácia S351↔S352** — ak S352 hľadá dokument podľa registryRecordNumber, nenájde ho
- **Audit trail** — skutočné číslo záznamu sa stráca úplne

## Odporúčaný fix

1. Presunúť `document.setRegistryRecordNumber(registryRecordNumber)` za riadok 135-136 kde je premenná naplnená
2. Riadok 165 zmeniť na `document.setAssisted(assistedString)` alebo podobné
