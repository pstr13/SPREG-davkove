---
id: F-012
date: 2026-03-31
status: identified
severity: high
scenario: S312
---

# F-012 — S312 InsertStep: IOException pri prílohe sa ticho prehltne → dokument bez prílohy

## Súbor
`src/main/java/sk/socpoist/isreg/regint/batchservices/step/S312InsertRegistryRecordOutboundAndProcessStep.java:98-107`

## Popis

```java
for (Attachment attachment : form.getAttachment()) {
    ObjectsRegistryRecordFileSimpleDto att = new ObjectsRegistryRecordFileSimpleDto();
    att.setFileName(attachment.getFileName());
    try {
        att.setContentBase64(fileSystemBean.readBinaryFile(...));
        attachments.add(att);  // pridaná len ak sa podarí čítanie
    } catch (IOException e) {
        LOGGER.warn("Failed to read file: " + attachment.getFileName(), e);
        // ← attachment sa NEVLOŽÍ do zoznamu, ale pokračuje sa ďalej
    }
}
```

Ak príloha neexistuje alebo je neprístupná, vyhodí `IOException`. Táto je zachytená, zalogovaná ako WARN a spracovanie pokračuje BEZ prílohy. Dokument sa odošle do IS AR s prázdnym alebo neúplným zoznamom príloh.

## Dopad

- **Tichá strata dát**: dokument zaevidovaný v registratúre bez prílohy
- S312Document dostane stav DONE (spracovaný úspešne) — žiaden signál o problémoch
- Príjemca dostane neúplný dokument

## Prečo je to problematické

Chýbajúca príloha je pre IS AR validačná chyba len ak IS AR validuje prítomnosť príloh.
Ak nevaliduje (a v niektorých prípadoch nevaliduje), dokument prejde celým workflow, ale
obsah v registratúre je nekompletný.

## Fix

Zmeniť catch block: namiesto WARN-u a pokračovania vyhodiť `InvalidDataException`:

```java
} catch (IOException e) {
    throw new InvalidDataException("Cannot read attachment: " + attachment.getFileName(), e);
}
```

Alebo aspoň zmeniť na `LOGGER.error` a nastaviť dokument do stavu chyby.

## Verifikácia

Roman: Overiť či IS AR pri neúplnom zozname príloh vráti chybu alebo prijme dokument.
