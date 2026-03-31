# BREAKDOWN — SPREG-davkove

> Posledná aktualizácia: 2026-03-31

## Prehľad

Analýza legacy integračného kódu IS REG (dávkové spracovanie) s fokuson na problémy pri väčších dávkach. Deadline: ~7. apríl 2026.

## Fáza 1 — Verifikácia kritických bugov (priorita: najvyššia)

| # | Úloha | Status | Finding |
|---|-------|--------|---------|
| 1.1 | Overiť data corruption bug (Scenario351Notify:165 — registryRecordNumber vs assistedValue) | **confirmed** | F-001 |
| 1.2 | Overiť NullPointer v GenerateXmlFormBean pri UNCOMPLETE_DATA / UNIDENTIFIED_PERSON | **confirmed** | F-002 |
| 1.3 | Overiť type-unsafe list access v Scenario352Debt:162 | **confirmed** (tech debt) | F-003 |

## Fáza 2 — Analýza batch/concurrency správania (priorita: vysoká)

| # | Úloha | Status | Finding |
|---|-------|--------|---------|
| 2.1 | Analyzovať chýbajúci timeout na IDebtServiceStep SOAP volanie | todo | F-004 |
| 2.2 | Analyzovať chýbajúci retry/deadletter mechanizmus | todo | F-005 |
| 2.3 | Analyzovať JPA query timeout pri záťaži | todo | F-006 |
| 2.4 | Preskúmať Camel threading model — koľko concurrent routes beží | todo | — |
| 2.5 | Analyzovať UAT logy pre vzory zlyhania pri väčších dávkach | todo | — |

## Fáza 3 — Edge cases (ak ostane čas)

| # | Úloha | Status | Finding |
|---|-------|--------|---------|
| 3.1 | Namespace anomália FO↔PO | todo | F-007 |
| 3.2 | Nový parameter alternativeDestination v callbacku | todo | F-008 |

## Legenda
- **F-XXX** = Finding ID (bude v docs/findings/)
- **todo** = neoverené, **confirmed** = potvrdený bug, **dismissed** = false positive
