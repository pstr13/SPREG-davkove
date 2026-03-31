# BREAKDOWN — SPREG-davkove

> Posledná aktualizácia: 2026-03-31

## Prehľad

Analýza legacy integračného kódu IS REG (dávkové spracovanie) s fokuson na problémy pri väčších dávkach. Deadline: ~7. apríl 2026.

## DÔLEŽITÁ POZNÁMKA: Zmena zdroja kódu

Fáza 1 (J-001) analyzovala repo **JVP-nedoplatky** (pomocný zdroj).
Fáza 2+ analyzuje **batch-services-integration** (produkčný kód v UAT).
Findingy F-001 až F-008 sú z JVP-nedoplatky — neaplikovateľné priamo na produkčný kód.

## Fáza 1 — Analýza JVP-nedoplatky (ARCHIVOVANÁ)

| # | Úloha | Status | Finding |
|---|-------|--------|---------|
| 1.1 | Overiť data corruption bug (Scenario351Notify:165) | **confirmed** (z JVP repo) | F-001 |
| 1.2 | Overiť NullPointer v GenerateXmlFormBean | **confirmed** (z JVP repo) | F-002 |
| 1.3 | Overiť type-unsafe list access v Scenario352Debt:162 | **confirmed** (z JVP repo) | F-003 |

## Fáza 2 — Analýza batch-services-integration (AKTUÁLNA)

### 2A — Scenario311Notify

| # | Úloha | Status | Finding |
|---|-------|--------|---------|
| 2A.1 | onException missing endChoice → HTTP 200 namiesto 500 | **identified** | F-009 |
| 2A.2 | Path traversal (priznaná zraniteľnosť v Javadoc) | **identified** | F-016 |
| 2A.3 | seda queue bez size limitu | **identified** | poznámka v J-002 |

### 2B — Scenario312BatchHandling

| # | Úloha | Status | Finding |
|---|-------|--------|---------|
| 2B.1 | InsertStep: NPE pri prázdnych RegistryRecords | **identified** | F-011 |
| 2B.2 | InsertStep: IOException pri prílohe ticho prehltnutá | **identified** | F-012 |
| 2B.3 | RouteConfig: double JPA write pri max retry | **identified** | F-014 |
| 2B.4 | S312CloseBatchWhenReadyStep: chýba @ApplicationScoped | **investigate** | — |
| 2B.5 | ActiveRegException code 0 — tiché zlyhanie | **identified** | poznámka v J-002 |
| 2B.6 | Chýbajúci SOAP timeout (F-004 z JVP) | **needs re-eval** | F-004? |
| 2B.7 | Chýbajúci retry/deadletter (F-005 z JVP) | **needs re-eval** | F-005? |
| 2B.8 | Camel threading model — concurrent routes | **needs analysis** | Q-001 |

### 2C — Scenario38ImportCSV

| # | Úloha | Status | Finding |
|---|-------|--------|---------|
| 2C.1 | Batch filter nested v File filter → Batch nikdy FAILED | **fixed** (commit eabc3f5) | F-010 |
| 2C.2 | Žiadna paginácia pri loadovaní dokumentov → OOM | **identified** | F-013 |
| 2C.3 | Hardcoded threads(10,10) — nekonfigurovateľné | **identified** | (poznámka J-002) |
| 2C.4 | maximumRedeliveries hardcoded na 3 (regres z eabc3f5) | **identified** | F-015 |

## Fáza 3 — Edge cases (ak ostane čas)

| # | Úloha | Status | Finding |
|---|-------|--------|---------|
| 3.1 | Namespace anomália FO↔PO | todo | F-007? |
| 3.2 | Nový parameter alternativeDestination v callbacku | todo | F-008? |

## Legenda
- **F-XXX** = Finding ID (v docs/findings/)
- **identified** = nájdené v kóde, čaká verifikácia Romanom
- **confirmed** = Roman potvrdil
- **dismissed** = false positive
- **needs re-eval** = finding z JVP repo, treba overiť v batch-services-integration
