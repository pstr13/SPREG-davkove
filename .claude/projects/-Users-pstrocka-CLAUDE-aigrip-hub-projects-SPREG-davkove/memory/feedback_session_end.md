---
name: Koniec session protokol je automatický
description: Pri signáloch konca session (odpájam sa, zmena účastníkov) okamžite spustiť koniec session protokol — nečakať na pripomienku
type: feedback
---

Pri signáloch konca session (riešiteľ sa odpája, mení sa účastník, explicitné ukončenie) VŽDY automaticky spustiť koniec session protokol: finalizovať journal (status: done), aktualizovať STATE/BREAKDOWN, pushnúť.

**Why:** Peter musel manuálne upozorniť že journal nebol finalizovaný. Framework to vyžaduje automaticky.

**How to apply:** Kedykoľvek zaznamenaš signál konca session — okamžite koniec session protokol, až potom rozlúčka.
