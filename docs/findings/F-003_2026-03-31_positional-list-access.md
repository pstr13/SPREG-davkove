---
id: F-003
date: 2026-03-31
severity: medium
status: confirmed
component: Scenario352Debt.java
lines: [121, 123, 126, 162]
verified_by: static-analysis + WSDL
uat_evidence: funguje s aktuálnym WSDL
---

# F-003 — Pozičný prístup k CXF MessageContentsList

## Popis

`Scenario352Debt` pristupuje k parametrom SOAP callbacku cez `body().method("get(N)")` — pozičný prístup do `MessageContentsList`. Aktuálne funguje, ale pri zmene WSDL (pridanie/preusporiadanie parametrov) sa ticho rozbije.

## Aktuálne mapovanie

| Index | Parameter | Typ |
|-------|-----------|-----|
| 0 | requestId | String |
| 1 | resultCode | String |
| 2 | resultDetail | String |
| 3 | applicant | ApplicantCorporateBody / ApplicantPhysicalPerson |
| 4 | applicantHasArrears | Boolean |
| 5 | arrears | Arrears |
| 6 | branchOffice | BranchOffice |
| 7 | creationOfSubmissions | XMLGregorianCalendar |

## Riziko

- Pri pridaní parametra pred pozíciu 3: indexy sa posunú, String pozície (0,1,2) vrátia zlé hodnoty **bez výnimky**
- Tichá data corruption — ťažko detekovateľná

## Hodnotenie

Technický dlh s rizikom pri WSDL zmene. Nie je aktívny bug — priorita nižšia ako F-001, F-002.
