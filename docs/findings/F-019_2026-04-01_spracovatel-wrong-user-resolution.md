---
id: F-019
date: 2026-04-01
severity: high
status: open
scenario: S334 (Scenario334Receive) + S311/S312 (batch-services-integration)
bug_refs: [SPISREG-2394, SPISREG-2422, SPISREG-2423, SPISREG-2428]
---

# F-019 — Chybný spracovateľ na zázname: hardcoded getNpUser() namiesto agenda-specific user

## Symptóm

Na **zázname** (nie spise) sa zobrazuje:
- **Spracovateľ**: `WsUser ArAutomat` (technický WS účet) namiesto pobočkového účtu (napr. `socpoist\bam-ar_jvp`)
- **Organizačný útvar**: prázdny namiesto `700`

Pozorované na TEST aj DEV, pre JVP aj DP agendu.

## Očakávaný stav (podľa excelu s formulármi)

| Pole | Očakávaná hodnota | Aktuálna hodnota |
|------|-------------------|------------------|
| Spracovateľ | `socpoist\{pobočka}-ar_jvp` (napr. `socpoist\bam-ar_jvp`) | `WsUser ArAutomat` |
| Zaevidoval / Vlastník spisu | `AR_AUTOMAT` | OK (na spise) |
| Org. jednotka spracovateľa | `700` | prázdne |

## Root cause analýza

### Problém 1: S334 (WS inbound) — hardcoded `getNpUser()`

**Súbor**: `registratura-services-integration/src/main/java/.../scenario/Scenario334Receive.java`

**Riadky 239-246** (non-assisted flow):
```java
.otherwise()
.process(s334BranchCodeParser)          // extrahuje branchCode z formulára
.toD("jpa-common:Branch?...&parameters.code=${variable.branchCode}")
.setVariable(Constant.COMPANY_ID, body().method("getRegistryId()"))
.setBody(body().method("getNpUser"))    // ← BUG: vždy NP user, nie agenda-specific
.to("direct:GetUserStep")
.setVariable(Constant.REGISTRAR_ID, body().method("getId"))
```

**Problém**: Metóda `getNpUser()` je hardcoded. Pre JVP agendu by sa mal použiť `getJvpUser()`, pre NASP `getNaspUser()`.

**Entity `Branch.java`** má tri oddelené polia:
```java
@Column(name = "np_user")    private String npUser;     // socpoist\{branch}-ar_dp
@Column(name = "jvp_user")   private String jvpUser;    // socpoist\{branch}-ar_jvp
@Column(name = "nasp_user")  private String naspUser;   // socpoist\{branch}-ar_nasp
```

Kód vždy volá `getNpUser()` bez ohľadu na agendu → pre JVP sa nastaví NP účet (ak existuje) alebo null.

### Problém 2: Prázdny batch_registrar pre JVP v Agenda tabuľke

**Súbor**: `common-integration/src/main/resources/sql/changes/009-update-agenda.xml`

```xml
<update tableName="agenda">
    <column name="batch_registrar" value=""/>     <!-- PRÁZDNY pre JVP -->
    <where>code = 'JVP'</where>
</update>
```

Pre porovnanie — DP má nastavený batch_registrar:
```xml
<column name="batch_registrar" value="socpoist\\ba-ar_dp"/>
```

Ak S311/S312 (batch-services-integration) používa `Agenda.getBatchRegistrar()` pre REGISTRAR_ID, pre JVP dostane prázdny string → GetUserStep zlyhá alebo vráti null → IS AR fallbackne na WS calling user (svc_ar_automat = WsUser ArAutomat).

### Problém 3: Chýbajúci organizačný útvar

`REGISTRAR_BUSINESS_UNIT_ID` sa odvodzuje od `REGISTRAR_ID` cez `GetBusinessUnitStep`. Ak je REGISTRAR_ID nesprávny alebo null, business unit sa nerozlíši → org. útvar je prázdny.

## Dôsledky

1. **Spisy nie sú evidované na oddelení** — chýba organizačný útvar
2. **Doručenky nebudú odoslané do DB** — systém ich nedokáže priradiť správnemu oddeleniu
3. **Postihnuté všetky JVP dávky** (a potenciálne aj NASP, EUP, NPLPC — všetky s prázdnym batch_registrar)
4. Bug sa prejavuje na DEV aj TEST → blocker pre UAT

---

## Návrh implementácie pre developera

### FIX 1: S334 (Scenario334Receive) — agenda-specific user resolution

**Kde**: `Scenario334Receive.java`, riadky 239-246

**Pred (current)**:
```java
.otherwise()
.process(s334BranchCodeParser)
.toD("jpa-common:Branch?singleResult=true&namedQuery=selectBranchByCode&parameters.code=${variable.branchCode}")
.setVariable(Constant.COMPANY_ID, body().method("getRegistryId()"))
.setBody(body().method("getNpUser"))     // ← hardcoded NP
.to("direct:GetUserStep")
.setVariable(Constant.REGISTRAR_ID, body().method("getId"))
```

**Po (fix)**:
```java
.otherwise()
.process(s334BranchCodeParser)
.toD("jpa-common:Branch?singleResult=true&namedQuery=selectBranchByCode&parameters.code=${variable.branchCode}")
.setVariable(Constant.COMPANY_ID, body().method("getRegistryId()"))
.setVariable("resolvedBranch", body())
.process(exchange -> {
    String agendaCode = exchange.getVariable(Constant.AGENDA, 
        sk.socpoist.isreg.regint.common.database.Agenda.class).getCode();
    sk.socpoist.isreg.regint.common.database.Branch branch = 
        exchange.getVariable("resolvedBranch", 
            sk.socpoist.isreg.regint.common.database.Branch.class);
    
    String registrarUser;
    switch (agendaCode) {
        case "JVP":
            registrarUser = branch.getJvpUser();
            break;
        case "NASP":
            registrarUser = branch.getNaspUser();
            break;
        default:
            registrarUser = branch.getNpUser();
            break;
    }
    
    if (registrarUser == null || registrarUser.isBlank()) {
        throw new InvalidDataException(
            "Chýba konfigurácia branch user pre agendu " + agendaCode 
            + " a pobočku " + branch.getCode());
    }
    exchange.getMessage().setBody(registrarUser);
})
.to("direct:GetUserStep")
.setVariable(Constant.REGISTRAR_ID, body().method("getId"))
```

**Alternatíva (čistejšia)** — nová metóda na `Branch` entite:

```java
// Branch.java — pridať metódu
public String getUserByAgenda(String agendaCode) {
    switch (agendaCode) {
        case "JVP": return jvpUser;
        case "NASP": return naspUser;
        default: return npUser;  // DP, EZ, EESSI → NP user
    }
}
```

Potom v Scenario334Receive:
```java
.setBody(exchange -> {
    Branch branch = exchange.getMessage().getBody(Branch.class);
    String agendaCode = exchange.getVariable(Constant.AGENDA, Agenda.class).getCode();
    return branch.getUserByAgenda(agendaCode);
})
```

### FIX 2: Agenda tabuľka — nastaviť batch_registrar pre JVP

**Kde**: Nový Liquibase changeset (napr. `0XX-update-agenda-jvp-registrar.xml`)

**Poznámka**: `batch_registrar` v Agenda tabuľke je **centrálny default** (nie per-branch). Pre agendy kde spracovateľ závisí od pobočky, `batch_registrar` nestačí — je potrebný per-branch lookup (viď FIX 1). Preto:

**Možnosť A** — Ak S311/S312 (batch-services-integration) používa `batchRegistrar` a nepotrebuje per-branch rozlíšenie:
```xml
<changeSet id="0XX-update-agenda-jvp-registrar" author="Roman">
    <update tableName="agenda">
        <column name="batch_registrar" value="socpoist\\ba-ar_jvp"/>
        <where>code = 'JVP'</where>
    </update>
</changeSet>
```
Toto nastaví default `socpoist\ba-ar_jvp` (BA = Bratislava = centrála). Pre iné pobočky to nebude správne — ale je to lepšie ako prázdny string.

**Možnosť B** (odporúčaná) — Rovnaký per-branch lookup ako FIX 1 aj pre S311/S312:
Analyzovať kód v `batch-services-integration` a implementovať rovnakú logiku:
1. Z formulára/dokumentu extrahovať pobočkový kód
2. Lookup v Branch tabuľke
3. Použiť `getUserByAgenda(agendaCode)` namiesto hardcoded metódy

### FIX 3: Overenie Branch dát v DB

**Kde**: Produkčná/testovacia databáza

Overiť že `branch` tabuľka má vyplnený stĺpec `jvp_user` pre všetky pobočky:

```sql
-- Kontrola: pobočky bez JVP usera
SELECT code, title, jvp_user, np_user, nasp_user 
FROM branch 
WHERE jvp_user IS NULL OR jvp_user = '';
```

Ak `jvp_user` nie je vyplnený, je potrebný Liquibase changeset na doplnenie hodnôt:

```sql
-- Vzor: jvp_user = socpoist\{code_lowercase}-ar_jvp
UPDATE branch SET jvp_user = 'socpoist\' || LOWER(code) || '-ar_jvp';
```

**Overiť vzor pomenovania** s Nuaktivom — vzor `socpoist\{branch}-ar_jvp` je odvodený z excelu (socpoist\bam-ar_jvp, socpoist\bb-ar_jvp, ...).

## Poradie implementácie

| Krok | Čo | Kto | Blocker |
|------|----|-----|---------|
| 1 | Overiť `jvp_user` stĺpec v branch tabuľke (DEV/TEST) | Roman | — |
| 2 | Ak chýba → Liquibase changeset na doplnenie `jvp_user` | Roman | Krok 1 |
| 3 | FIX 1: Scenario334Receive — agenda-specific user lookup | Roman | Krok 2 |
| 4 | FIX 2: Analyzovať S311/S312 v batch-services-integration | Roman | — (paralelne) |
| 5 | Smoke test na DEV: JVP dávka → overiť spracovateľ + org. útvar | Roman | Krok 3 |
| 6 | Deployment na TEST + retest SPISREG-2422 | Roman | Krok 5 |

## Testovanie

### Verifikácia po fixe

1. Poslať JVP dávku na pobočku BAM (Bratislava)
2. Overiť na **zázname**:
   - Spracovateľ = `socpoist\bam-ar_jvp` (nie WsUser ArAutomat)
   - Organizačný útvar = `700`
3. Overiť na **spise**:
   - Údaje o spracovateľovi a vlastníkovi konzistentné
4. Zreprodukovať pre ďalšie pobočky (BB, BJ, CA) — overiť per-branch rozlíšenie

### Regresné testy

- DP dávky musia naďalej fungovať s NP userom
- EESSI dávky — overiť batch_registrar (`socpoist\ba-ar_eessi`)
- Assisted flow (sAMAccountName) — nesmie byť ovplyvnený

## Súvisiace bugy

| Bug | Prostredie | Agenda | Rovnaký root cause? |
|-----|-----------|--------|---------------------|
| SPISREG-2394 | DEV | JVP | Áno — rovnaký problém |
| SPISREG-2422 | TEST | JVP | Áno — primárny bug |
| SPISREG-2423 | TEST | DP | Pravdepodobne — overiť či DP používa správny NP user |
| SPISREG-2428 | DEV | DP | Pravdepodobne — overiť |

**SPISREG-2423/2428 (DP)**: Ak DP dávky tiež majú zlý spracovateľ, root cause môže byť iný — DP má vyplnený `batch_registrar` a `getNpUser()` by mal byť správny pre DP. Treba overiť či `npUser` je vyplnený v branch tabuľke pre príslušné pobočky.
