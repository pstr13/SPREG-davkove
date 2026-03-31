# INTENT — SPREG-davkove

> Status: draft
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

_Čaká na INTENT interview s riešiteľom (Roman)._

## Čo výslovne neriešime (out-of-scope)

_Čaká na INTENT interview s riešiteľom (Roman)._

## Obmedzenia

_Čaká na INTENT interview s riešiteľom (Roman)._

## Kontext

- Zdrojové kódy IS REG sú dostupné v repo [JVP-nedoplatky](https://github.com/pstr13/JVP-nedoplatky)
- Kód je už v UAT — analýza prebieha paralelne s UAT testovaním
- Projekt JVP-nedoplatky slúži výlučne ako zdroj kódu — jeho INTENT a kontext sa nepreberajú

## Tím

- **Riešiteľ**: Roman
- **Reviewer**: Tibor
