---
id: J-001
date: 2026-03-31
status: draft
participants: [roman, claude]
---

# J-001 — INTENT interview + prvá analýza kódu

## Čo sa riešilo
- Dokončenie INTENT interview — schválené
- Pripojenie zdrojového repo JVP-nedoplatky (git pull, inventúra obsahu)
- Kompletná analýza zdrojových kódov IS REG integrácie (Scenario351, Scenario352, IDebtServiceStep, GenerateXmlFormBean, InsertRegistryRecordStep, DebtCallbackService)
- Identifikácia 8 potenciálnych problémov vrátane 3 kritických

## Kľúčové zistenie
IS REG kód nemá batch logiku na aplikačnej úrovni — každé podanie sa spracuje individuálne cez Camel route. Pri väčších dávkach ÚPVS volá /notify N-krát nezávisle, čo vysvetľuje problémy pri väčšom počte súborov.

## Identifikované problémy

### Kritické
1. **Data corruption** — Scenario351Notify:165 — registryRecordNumber prepísaný assistedValue
2. **NullPointer** — GenerateXmlFormBean — chýbajú null checky pri UNCOMPLETE_DATA odpovediach
3. **Type-unsafe access** — Scenario352Debt:162 — body().method("get(1)") predpokladá fixnú pozíciu

### Vysoké riziko (batch-related)
4. **Žiadny timeout** — IDebtServiceStep synchrónne SOAP volanie bez timeoutu
5. **Žiadny retry/deadletter** — zlyhané callbacky sa strácajú
6. **JPA bez timeoutu** — DB spomalenie pri záťaži

### Stredné riziko
7. **Namespace anomália** — FO↔PO namespace swap
8. **alternativeDestination** — nový parameter v callbacku, nehandlovaný

## Ďalšie kroky
- Vytvorený BREAKDOWN s 3-fázovým prístupom
- Fáza 1: verifikácia kritických bugov proti UAT logom
