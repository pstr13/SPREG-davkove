---
id: F-009
date: 2026-03-31
status: identified
severity: critical
scenario: S311
---

# F-009 — S311 onException: chýba endChoice() → HTTP 200 vrátený namiesto HTTP 500

## Súbor
`src/main/java/sk/socpoist/isreg/regint/batchservices/scenario/Scenario311Notify.java:46-59`

## Popis

V `onException(Response.class)` bloku je choice/when konštrukcia bez `endChoice()`:

```java
onException(Response.class)
    .handled(true)
    .log(...)
    .choice()
    .when(variable(Constant.DOCUMENT).isNotNull())
        .setBody(variable(Constant.DOCUMENT))
        .transform(Body.state(S311DocumentState.FAILED))
        .transform(Body.errorTextFromException())
        .to(JPA_S311)
    // ← chýba endChoice() alebo otherwise()
    .removeHeaders(Constant.WILDMARK)                      // ← vnútri when() !!!
    .setHeader(Exchange.HTTP_RESPONSE_CODE, constant(500)) // ← vnútri when() !!!
    .setBody(Body.asNull());                               // ← vnútri when() !!!
```

V Camel Java DSL, bez `endChoice()`, sú `removeHeaders`, `setHeader(500)` a `setBody(null)` súčasťou `when()` bloku — nie za ním. Ak výnimka nastane pred nastavením premennej DOCUMENT (pred riadkom 84), `when()` predikát je false, celý blok sa preskočí a HTTP response code sa NENASTAVÍ.

## Dopad

- ÚPVS dostane HTTP 200 OK namiesto HTTP 500
- ÚPVS usúdi, že notifikácia bola úspešne prijatá — neretryuje
- S311Document sa nezapíše do DB v stave FAILED
- Dáta sa ticho strácajú bez akéhokoľvek záznamu

## Kedy nastáva

Výnimka vznikne PRED riadkom 84 (`setVariable(Constant.DOCUMENT, body())`), napr.:
- Pri JSONPath chybe pri parsovaní payloadu
- Ak `to(JPA_S311)` na riadku 83 vyhodí výnimku (DB nedostupná)

## Fix

Pridať `endChoice()` pred `removeHeaders()`:

```java
.when(variable(Constant.DOCUMENT).isNotNull())
    .setBody(...)
    .transform(...)
    .to(JPA_S311)
.endChoice()  // ← TOTO CHÝBA
.removeHeaders(Constant.WILDMARK)
.setHeader(Exchange.HTTP_RESPONSE_CODE, constant(500))
.setBody(Body.asNull());
```

## Verifikácia

Roman: overiť Camel DSL scope pravidlá alebo napísať unit test s null DOCUMENT scenárom.
