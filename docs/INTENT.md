# INTENT — SPREG-davkove

> Status: approved
> Dátum: 2026-03-31

## Čo riešime

Komplexné mapovanie a analýza kódu legacy integrácií projektu IS REG (dávkové spracovanie — SPREG), ktoré sú v tejto chvíli už nasadené v UAT prostredí. Cieľom je systematicky prejsť integračný kód, identifikovať skryté bugy, logické chyby, edge cases a architektonické slabiny skôr, ako sa odhalia počas UAT testovania alebo sa prejavia až v produkčnom prostredí.

## Prečo to riešime

Legacy integračný kód IS REG je komplexný a historicky narastený. Bugy odhalené až v UAT alebo produkcii sú výrazne drahšie na opravu — vyžadujú rollback, hotfix cykly a koordináciu medzi tímami. Proaktívna analýza kódu pred UAT testami umožňuje identifikovať a opraviť problémy v kontrolovanom prostredí s nižšími nákladmi.

## Pre koho je výsledok

- Roman (riešiteľ) — vykonáva analýzu, produkuje zistenia
- Tibor (reviewer) — validuje zistenia, rozhoduje o prioritách opráv
- Vývojový tím IS REG — dostane konkrétny zoznam nálezov na opravu

## Ako vyzerá úspech

- Všetky potenciálne chyby na dávkových integráciách legacy rozhraní IS REG sú identifikované
- Každý finding je potvrdený Romanom (validácia že je reálny problém)
- Potvrdené findingy sú opravené v kóde

## Čo výslovne neriešime (out-of-scope)

- Vedome nedefinované — scope nie je obmedzený. Ak analýza odhalí problém, riešime ho bez ohľadu na kategóriu.

## Obmedzenia

- UAT končí ~7. apríl 2026 — všetko čo má hodnotu musí byť identifikované do vtedy
- Roman robí analýzu popri bug fixingu na IS REG — kapacita je obmedzená, AI facilitátor poskytuje maximálnu podporu
- Fokus: problémy s dávkami s väčším počtom súborov/dávok

## Kontext

- Zdrojové kódy IS REG sú dostupné v repo [JVP-nedoplatky](https://github.com/pstr13/JVP-nedoplatky) — klonované v hub projekte, slúži výlučne ako zdroj (INTENT JVP-nedoplatky sa nepreberá)
- JVP-nedoplatky obsahuje: zdrojové kódy IS REG, enterprise architekt dáta, XML príklady callback odpovedi, logy z manuálnych testov, e-mailové podklady
- Kód je už v UAT — analýza prebieha paralelne s riešením bugov na projekte IS REG
- Účel: proaktívne identifikovať potenciálne chyby na dávkových integráciách legacy rozhraní IS REG skôr, ako sa prejavia počas UAT alebo PROD
- Analytik Stano poskytuje doménový kontext

## Tím

- **Riešiteľ**: Roman
- **Reviewer**: Tibor
- **Analytik**: Stano
