# UNKNOWNS

> Posledná aktualizácia: 2026-03-31

## Otvorené

### Q-001 — Ako sa správa Camel threading pri N súbežných /notify volaniach?
- Kontext: IS REG spracuje každé podanie individuálne. Pri väčšej dávke ÚPVS zavolá /notify N-krát. Nie je jasné koľko concurrent routes Camel pustí a či nedôjde k resource exhaustion.
- Priorita: vysoká (Fáza 2)

### Q-002 — Existujú UAT logy z testovania väčších dávok?
- Kontext: Dostupné logy (TEST-528, 531, 535, 536) pokrývajú len jednotlivé manuálne testy. Pre analýzu batch problémov potrebujeme logy z dávok s väčším počtom súborov.
- Priorita: vysoká

### Q-003 — Boli UNIDENTIFIED_PERSON / UNCOMPLETE_DATA scenáre testované v UAT?
- Kontext: F-002 potvrdzuje crash pri týchto odpovediach, ale UAT logy ich neobsahujú.
- Priorita: vysoká
