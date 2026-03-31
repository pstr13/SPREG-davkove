---
id: L-001
date: 2026-03-31
scope: framework
propagated: false
---

# L-001 — Journal finalizácia nie je voliteľná

## Čo sa stalo

Na konci session facilitátor nezfinalizoval draft journal (nenastavil status: done) a nespustil koniec session protokol automaticky. Správca musel upozorniť manuálne.

## Poučenie

Framework jasne definuje "Koniec session protokol" — finalizácia journalu je krok 1. Facilitátor to musí urobiť **automaticky** keď sa session končí (riešiteľ sa odpája, mení sa účastník, explicitné ukončenie). Nečakať na pripomienku.

## Signály konca session

- Riešiteľ povie "odpájam sa", "končím", "pokračuj s X"
- Zmena účastníkov (Peter → Roman/Stano)
- Explicitné "koniec session"

Pri akomkoľvek z týchto signálov okamžite spustiť koniec session protokol: finalizovať journal, aktualizovať STATE, BREAKDOWN, pushnúť.
