---
id: F-002
date: 2026-03-31
severity: critical
status: confirmed
component: Scenario352Debt.java, GenerateXmlFormBean.java
lines: [Scenario352Debt:138-142 vs 162, GenerateXmlFormBean:95-156]
verified_by: static-analysis + XML examples
uat_evidence: none (non-OK scenáre neboli testované)
---

# F-002 — NullPointerException pri UNIDENTIFIED_PERSON a UNCOMPLETE_DATA odpovediach

## Popis

`Scenario352Debt` volá `GenerateXmlFormBean.handleFO/handlePO` **pred** kontrolou `resultCode`. Non-OK odpovede (UNIDENTIFIED_PERSON, UNCOMPLETE_DATA) majú chýbajúce polia, čo spôsobí NullPointerException.

## Mechanizmus

Flow v Scenario352Debt:
1. Riadky 138-142: `GenerateXmlFormBean.handleFO/handlePO` — **vždy sa volá**
2. Riadok 162: `body().method("get(1)").isNotEqualTo("OK")` — check až potom

## Crash body pre UNIDENTIFIED_PERSON

| Pole | Stav | Kde padne |
|------|------|-----------|
| `branchOffice` | null | `response.getBranchOffice().getPersonData()` → NPE |
| `applicantHasArrears` | null | `response.isApplicantHasArrears().toString()` → NPE |
| `physicalAddress` | prázdny list | `getPhysicalAddress().getFirst()` → NoSuchElementException |

Minimálne 3 nezávislé crash body — garantovaný pád.

## Dopad

- Pri UNIDENTIFIED_PERSON odpovedi z DTLN: **kompletné zlyhanie spracovania**
- Žiadny retry/deadletter → odpoveď sa stratí
- Testované nebolo — UAT logy pokrývajú len OK flow

## Odporúčaný fix

Presunúť `resultCode` check (riadok 162) **pred** volanie `GenerateXmlFormBean` (riadky 138-142). Non-OK odpovede by mali ísť priamo na `UpdateRegistryRecordSpecifiedStep` bez XML generovania.
