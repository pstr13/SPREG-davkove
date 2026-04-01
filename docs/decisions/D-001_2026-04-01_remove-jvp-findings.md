---
id: D-001
date: 2026-04-01
slug: remove-jvp-findings
status: locked
author: roman
---

# D-001 — Odstránenie findingov F-001, F-002, F-003 (JVP-nedoplatky)

## Rozhodnutie

Findingy F-001, F-002 a F-003 sú trvalo odstránené z projektu SPREG-davkove.

## Dôvod

Tieto findingy boli identifikované v repozitári **JVP-nedoplatky**, ktorý slúžil ako pomocný zdroj v Fáze 1. Triedy, v ktorých boli bugy nájdené (`Scenario351Notify`, `Scenario352Debt`, `GenerateXmlFormBean`), **neexistujú v produkčnom kóde** (batch-services-integration).

Projekt SPREG-davkove sa zameriava výlučne na scenáre S311 (Notify), S312 (BatchHandling) a S38 (ImportCSV) v batch-services-integration. F-001/F-002/F-003 sa týkali iného kódu, iného repozitára, a nie sú relevanté pre UAT ani produkciu SPREG.

## Dopad

- Súbory `F-001_*.md`, `F-002_*.md`, `F-003_*.md` v `docs/findings/` sú zmazané.
- BREAKDOWN Fáza 1 je označená ako archivovaná (bez aktívnych finding odkazov).
- Q-003 v UNKNOWNS aktualizovaná (odstránená väzba na F-002).
- Findingy F-004 až F-008 (tiež z JVP repo) zostávajú mimo aktívneho zoznamu — nikdy neboli vytvorené ako súbory, BREAKDOWN ich označuje ako `needs re-eval`.
