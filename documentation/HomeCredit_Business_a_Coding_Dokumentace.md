# Home Credit Default Risk – Power BI dokumentace business a technické logiky

## 1. Účel dokumentu

Tento dokument popisuje, proč a jak funguje Power BI report vytvořený nad daty soutěže **Home Credit Default Risk** z Kaggle:

- hlavní zdroj: `https://www.kaggle.com/competitions/home-credit-default-risk/`
- datová stránka: `https://www.kaggle.com/competitions/home-credit-default-risk/data`

Dokument je určen k uložení společně s Power BI souborem jako vysvětlení:

1. business účelu reportu,
2. analytické hodnoty reportu,
3. použité datové logiky,
4. výběru tabulek,
5. transformací v Power Query,
6. SQL agregace v Databricks,
7. dimenzionálního modelu,
8. vztahů mezi tabulkami,
9. DAX měr,
10. logiky jednotlivých reportových úloh.

Dokument záměrně používá pouze informace z uvedeného Kaggle zdroje a z této pracovní konverzace. Nejsou zde doplňována externí vysvětlení, benchmarky, obchodní předpoklady ani domněnky mimo tento kontext.

---

## 2. Business význam reportu

### 2.1 Kontext podle zdroje Kaggle

Soutěž **Home Credit Default Risk** je formulována otázkou, zda lze predikovat, jak schopný je každý žadatel splatit úvěr. Kaggle soutěž popisuje jako úlohu zaměřenou na schopnost žadatele splácet úvěr a na riziko defaultu / problémů se splácením.

Z kontextu zdroje a zadání vyplývá, že data slouží ke zkoumání vztahu mezi:

- aktuální žádostí o úvěr,
- demografickými charakteristikami žadatele,
- aktuálním požadovaným úvěrem,
- příjmem klienta,
- historickými externími úvěry z credit bureau,
- předchozími žádostmi u Home Credit,
- historickými splátkami,
- výskytem problémů se splácením.

Hlavní business otázka reportu je:

> Jak vypadá struktura TRAIN žádostí a jak se jejich rizikový profil liší podle demografie, příjmu, externí úvěrové expozice, credit bureau historie a historické splátkové disciplíny?

### 2.2 Business hodnota reportu

Report nepředstavuje prediktivní model. Je to analytický Power BI report, který pomáhá pochopit strukturu trénovací populace a klíčové rizikové indikátory.

Business hodnota reportu spočívá v tom, že umožňuje:

1. **Porozumět složení portfolia žádostí**
   - podle pohlaví,
   - věkové skupiny,
   - vzdělání,
   - rodinného stavu,
   - počtu předchozích žádostí.

2. **Porovnat jednotkový a objemový pohled**
   - jednotkový pohled odpovídá počtu žádostí,
   - objemový pohled odpovídá součtu `AMT_CREDIT`,
   - díky tomu lze zjistit, zda některé skupiny tvoří malý počet žádostí, ale vysoký objem úvěru.

3. **Analyzovat zadlužení vzhledem k příjmu**
   - pomocí metriky DTI,
   - podle typu příjmu klienta (`NAME_INCOME_TYPE` / `Income Type`).

4. **Sledovat credit bureau historii v relativním čase**
   - vývoj počtu nedelikventních kontraktů (`STATUS = 0`) v credit bureau,
   - pouze pro aktuální cash žádosti,
   - s možností filtrování podle typu bureau úvěru (`CREDIT_TYPE`).

5. **Vyhodnotit historický výskyt DPD5**
   - zda klient někdy překročil DPD5 na předchozích Home Credit žádostech,
   - DPD5 je definováno podle zadání jako případ, kdy splátka není plně uhrazena do 5 dnů po splatnosti.

### 2.3 Proč report pracuje pouze s TRAIN datasetem

Všechny úlohy jsou v zadání formulovány pro **TRAIN dataset**. Proto je hlavní reportová populace postavena na tabulce:

```text
application_train.csv
```

Tabulka `application_test.csv` není pro řešené úlohy potřeba, protože:

- zadání nepožaduje scoring testovacích žádostí,
- `TARGET` existuje pouze v TRAIN datasetu,
- všechny požadované výpočty jsou formulovány pro aplikace v TRAIN datasetu.

---

## 3. Zadání a mapování na reportovou logiku

### 3.1 Úloha 1

Zadání:

> Show the distribution of applications in "TRAIN" dataset by key demographic information - both unit based and volume based (use AMT_CREDIT as a weight). Choose few dimensions like gender, age group, education type, family status, number of previous applications.

Interpretace:

- základní populace: `application_train.csv`,
- jednotková distribuce: počet žádostí,
- objemová distribuce: součet `AMT_CREDIT`,
- dimenze:
  - `Gender`,
  - `Age Group`,
  - `Education Type`,
  - `Family Status`,
  - `Previous Application Count Band`.

Použité tabulky:

- `factApplications`,
- `factPreviousApplications`,
- `dimApplicant`.

---

### 3.2 Úloha 2

Zadání:

> Calculate median DTI (Debt To Income) for each NAME_INCOME_TYPE of all applications in "TRAIN" dataset. DTI is defined as: (AMT_CREDIT_SUM of all clients existing active contracts in credit bureau + AMT_CREDIT of the contract currently applied for)/client's annual income.

Interpretace:

DTI se počítá na úrovni klienta / aktuální žádosti:

```text
DTI =
(
    součet AMT_CREDIT_SUM za aktivní bureau kontrakty klienta
    +
    AMT_CREDIT aktuální žádosti
)
/
AMT_INCOME_TOTAL
```

Aktivní bureau kontrakty:

```text
bureau.CREDIT_ACTIVE = "Active"
```

Výsledná metrika:

```text
Median DTI podle Income Type
```

Použité tabulky:

- `factApplications`,
- `factBureauCredits`,
- `dimApplicant`.

---

### 3.3 Úloha 3

Zadání:

> Create a chart showing a development in time (relative to application time) of average number of non-delinquent (0 /no-DPD) contracts in credit bureau per application in "TRAIN" dataset identified as "cash" (attribute NAME_CONTRACT_TYPE). Add a filter for CREDIT_TYPE of contract in credit bureau.

Interpretace:

- pracuje se pouze s TRAIN žádostmi,
- aktuální žádost musí být typu cash:
  - `NAME_CONTRACT_TYPE = "Cash loans"`,
- časová osa je relativní:
  - `bureau_balance.MONTHS_BALANCE`,
- nedelikventní bureau kontrakt je stav:
  - `bureau_balance.STATUS = "0"`,
- filtr pro typ bureau kontraktu:
  - `bureau.CREDIT_TYPE`.

Výsledná metrika:

```text
Average no-DPD bureau contracts per TRAIN cash application by relative month
```

Použité tabulky:

- `factApplications`,
- `factBureauCredits`,
- `factBureauMonthlyStatus`,
- `dimApplicant`,
- `dimCreditType`,
- `dimBureauStatus`,
- `dimRelativeMonth`.

---

### 3.4 Bonusová úloha

Zadání:

> Calculate the ratio of applications in "TRAIN" dataset which ever exceeded DPD5 on any of client's previous applications. Use the latest instalment version for each instalment. DPD5 is an event when payment arrives 5 days later than it is due or the payment does not arrive in full. Consider only fully paid instalments as paid.

Interpretace:

DPD5 je TRUE, pokud splátka není plně zaplacena nejpozději do 5 dnů po dni splatnosti.

Použité sloupce ze `installments_payments.csv`:

- `SK_ID_CURR`,
- `SK_ID_PREV`,
- `NUM_INSTALMENT_VERSION`,
- `NUM_INSTALMENT_NUMBER`,
- `DAYS_INSTALMENT`,
- `DAYS_ENTRY_PAYMENT`,
- `AMT_INSTALMENT`,
- `AMT_PAYMENT`.

Klíčová pravidla:

1. Pro každou splátku se použije nejnovější verze:
   ```text
   MAX(NUM_INSTALMENT_VERSION)
   ```
   na úrovni:
   ```text
   SK_ID_CURR + SK_ID_PREV + NUM_INSTALMENT_NUMBER
   ```

2. Platby se posuzují pouze do okna:
   ```text
   DAYS_ENTRY_PAYMENT <= DAYS_INSTALMENT + 5
   ```

3. Pokud suma plateb v tomto okně nedosáhne `AMT_INSTALMENT`, pak:
   ```text
   DPD5 = TRUE
   ```

4. Poměr se počítá na úrovni TRAIN žádostí:
   ```text
   počet TRAIN žádostí s EverDPD5 = TRUE
   /
   počet všech TRAIN žádostí
   ```

Použité tabulky:

- `factApplications`,
- Databricks agregovaný výstup `factInstallmentDPD5Summary`,
- `dimApplicant`.

---

## 4. Finální seznam použitých tabulek

### 4.1 Fact tabulky

```text
factApplications
factBureauCredits
factBureauMonthlyStatus
factPreviousApplications
factInstallmentDPD5Summary
```

### 4.2 Dimenzní tabulky

```text
dimApplicant
dimCreditType
dimBureauStatus
dimRelativeMonth
```

### 4.3 Tabulky ze zdroje, které se pro toto zadání nenačítají

```text
application_test.csv
POS_CASH_balance.csv
credit_card_balance.csv
sample_submission.csv
```

Důvod:

- úlohy jsou pouze nad TRAIN datasetem,
- POS/CASH a credit card balance nebyly pro konkrétní zadání potřeba,
- `sample_submission.csv` slouží pro Kaggle submission, nikoli pro report,
- `application_test.csv` neobsahuje `TARGET` a není potřeba pro zadané analýzy.

---

## 5. Datový model a grain tabulek

### 5.1 `factApplications`

Grain:

```text
1 řádek = 1 aktuální TRAIN žádost = 1 SK_ID_CURR
```

Účel:

- hlavní aplikační fact tabulka,
- obsahuje výši aktuálně požadovaného úvěru,
- obsahuje roční příjem,
- obsahuje atributy potřebné pro demografické analýzy,
- obsahuje TARGET.

### 5.2 `factBureauCredits`

Grain:

```text
1 řádek = 1 předchozí kontrakt v externím credit bureau
```

Účel:

- aktivní bureau expozice pro DTI,
- typ bureau úvěru pro filtr `CREDIT_TYPE`,
- vazba na měsíční bureau status přes `SK_ID_BUREAU`.

### 5.3 `factBureauMonthlyStatus`

Grain:

```text
1 řádek = 1 měsíční stav jednoho bureau kontraktu
```

Účel:

- relativní časová osa `MONTHS_BALANCE`,
- status kontraktu v daném měsíci,
- identifikace no-DPD stavu přes `STATUS = "0"`.

### 5.4 `factPreviousApplications`

Grain:

```text
1 řádek = 1 předchozí Home Credit žádost
```

Účel:

- výpočet počtu předchozích žádostí na klienta,
- následné zařazení klienta do skupiny počtu předchozích žádostí.

### 5.5 `factInstallmentDPD5Summary`

Grain:

```text
1 řádek = 1 TRAIN žádost / klient
```

Účel:

- bonusová DPD5 úloha,
- tabulka není načtena z raw CSV v Power Query,
- vzniká jako SQL agregace v Databricks,
- do Power BI se načítá pouze malý agregovaný výstup.

```puvodni zdrojova tabulka byla prilis velka na nahrani do power bi a tak sumarizovana tabulka je vytvorena/sumarizovana na zaklade nize uvedeneho SQL dotazu
WITH train_applications AS (

    -- Výběr všech žádostí z TRAIN datasetu
    -- Business logika:
    -- Zadání požaduje výpočet pouze pro aplikace v TRAIN datasetu.
    SELECT DISTINCT
        CAST(SK_ID_CURR AS BIGINT) AS SK_ID_CURR
    FROM home_credit.application_train

),

installment_rows AS (

    -- Načtení pouze potřebných sloupců ze splátkové historie
    -- Business logika:
    -- Pro DPD5 potřebujeme klienta, předchozí žádost, číslo splátky, verzi splátky,
    -- datum splatnosti, datum platby, předepsanou částku a zaplacenou částku.
    SELECT
        CAST(ip.SK_ID_CURR AS BIGINT) AS SK_ID_CURR,
        CAST(ip.SK_ID_PREV AS BIGINT) AS SK_ID_PREV,
        CAST(ip.NUM_INSTALMENT_VERSION AS DOUBLE) AS NUM_INSTALMENT_VERSION,
        CAST(ip.NUM_INSTALMENT_NUMBER AS BIGINT) AS NUM_INSTALMENT_NUMBER,
        CAST(ip.DAYS_INSTALMENT AS BIGINT) AS DAYS_INSTALMENT,
        CAST(ip.DAYS_ENTRY_PAYMENT AS BIGINT) AS DAYS_ENTRY_PAYMENT,
        CAST(ip.AMT_INSTALMENT AS DOUBLE) AS AMT_INSTALMENT,
        CAST(ip.AMT_PAYMENT AS DOUBLE) AS AMT_PAYMENT
    FROM home_credit.installments_payments ip
    INNER JOIN train_applications ta
        ON CAST(ip.SK_ID_CURR AS BIGINT) = ta.SK_ID_CURR
    WHERE ip.SK_ID_CURR IS NOT NULL
      AND ip.SK_ID_PREV IS NOT NULL
      AND ip.NUM_INSTALMENT_VERSION IS NOT NULL
      AND ip.NUM_INSTALMENT_NUMBER IS NOT NULL
      AND ip.DAYS_INSTALMENT IS NOT NULL
      AND ip.AMT_INSTALMENT IS NOT NULL

),

latest_version_rows AS (

    -- Označení nejnovější verze splátky
    -- Business logika:
    -- Zadání říká použít latest instalment version for each instalment.
    -- Úroveň jedné splátky je SK_ID_CURR + SK_ID_PREV + NUM_INSTALMENT_NUMBER.
    SELECT
        *,
        MAX(NUM_INSTALMENT_VERSION) OVER (
            PARTITION BY SK_ID_CURR, SK_ID_PREV, NUM_INSTALMENT_NUMBER
        ) AS LATEST_INSTALMENT_VERSION
    FROM installment_rows

),

latest_installment_rows AS (

    -- Ponechání pouze řádků z nejnovější verze splátkového kalendáře
    -- Business logika:
    -- Starší verze splátky ignorujeme, protože mohou být nahrazené novější verzí.
    SELECT
        SK_ID_CURR,
        SK_ID_PREV,
        NUM_INSTALMENT_VERSION,
        NUM_INSTALMENT_NUMBER,
        DAYS_INSTALMENT,
        DAYS_ENTRY_PAYMENT,
        AMT_INSTALMENT,
        AMT_PAYMENT
    FROM latest_version_rows
    WHERE NUM_INSTALMENT_VERSION = LATEST_INSTALMENT_VERSION

),

installment_level AS (

    -- Agregace na úroveň jedné splátky
    -- Business logika:
    -- Jedna splátka může mít více plateb.
    -- Pro DPD5 nás zajímá, kolik bylo zaplaceno nejpozději do 5 dní po splatnosti.
    SELECT
        SK_ID_CURR,
        SK_ID_PREV,
        NUM_INSTALMENT_NUMBER,
        MAX(NUM_INSTALMENT_VERSION) AS NUM_INSTALMENT_VERSION,
        MAX(DAYS_INSTALMENT) AS DAYS_INSTALMENT,
        MAX(AMT_INSTALMENT) AS AMT_INSTALMENT,

        SUM(
            CASE
                WHEN DAYS_ENTRY_PAYMENT IS NOT NULL
                 AND AMT_PAYMENT IS NOT NULL
                 AND DAYS_ENTRY_PAYMENT <= DAYS_INSTALMENT + 5
                THEN AMT_PAYMENT
                ELSE 0
            END
        ) AS PAID_WITHIN_DPD5_WINDOW

    FROM latest_installment_rows
    GROUP BY
        SK_ID_CURR,
        SK_ID_PREV,
        NUM_INSTALMENT_NUMBER

),

installment_dpd5 AS (

    -- Vyhodnocení DPD5 na úrovni jedné splátky
    -- Business logika:
    -- DPD5 = TRUE, pokud splátka nebyla plně zaplacena do 5 dní po splatnosti.
    -- Částečná platba se nepovažuje za plné zaplacení.
    SELECT
        SK_ID_CURR,
        SK_ID_PREV,
        NUM_INSTALMENT_NUMBER,
        NUM_INSTALMENT_VERSION,
        DAYS_INSTALMENT,
        AMT_INSTALMENT,
        PAID_WITHIN_DPD5_WINDOW,

        CASE
            WHEN PAID_WITHIN_DPD5_WINDOW < AMT_INSTALMENT THEN TRUE
            ELSE FALSE
        END AS IS_DPD5,

        CASE
            WHEN PAID_WITHIN_DPD5_WINDOW < AMT_INSTALMENT THEN 1
            ELSE 0
        END AS DPD5_UNIT

    FROM installment_level

),

client_level AS (

    -- Agregace na úroveň klienta / aktuální TRAIN žádosti
    -- Business logika:
    -- Bonusový úkol požaduje zjistit, zda klient někdy překročil DPD5
    -- na jakékoliv předchozí žádosti.
    SELECT
        SK_ID_CURR,
        COUNT(*) AS INSTALLMENT_COUNT,
        SUM(DPD5_UNIT) AS DPD5_INSTALLMENT_COUNT,
        MAX(DPD5_UNIT) AS EVER_DPD5_UNIT
    FROM installment_dpd5
    GROUP BY
        SK_ID_CURR

)

-- Finální výstup na úrovni všech TRAIN žádostí
-- Business logika:
-- TRAIN žádosti bez splátkové historie musí zůstat v denominatoru.
-- Proto se vrací i klienti bez záznamu v installments_payments s hodnotami 0 / FALSE.
SELECT
    ta.SK_ID_CURR,

    COALESCE(cl.INSTALLMENT_COUNT, 0) AS InstallmentCount,
    COALESCE(cl.DPD5_INSTALLMENT_COUNT, 0) AS DPD5InstallmentCount,

    CASE
        WHEN COALESCE(cl.EVER_DPD5_UNIT, 0) = 1 THEN TRUE
        ELSE FALSE
    END AS EverDPD5,

    COALESCE(cl.EVER_DPD5_UNIT, 0) AS EverDPD5Unit

FROM train_applications ta
LEFT JOIN client_level cl
    ON ta.SK_ID_CURR = cl.SK_ID_CURR
```
### 5.6 `dimApplicant`

Grain:

```text
1 řádek = 1 TRAIN žádost / klient
```

Účel:

- centrální dimenze pro aplikační a klientské atributy,
- obsahuje demografické atributy,
- obsahuje cash příznak,
- obsahuje počet předchozích žádostí a bucket.

### 5.7 `dimCreditType`

Grain:

```text
1 řádek = 1 CREDIT_TYPE
```

Účel:

- filtr pro typ bureau úvěru.

### 5.8 `dimBureauStatus`

Grain:

```text
1 řádek = 1 bureau STATUS
```

Účel:

- popis stavu bureau kontraktu,
- řazení statusů,
- možnost filtrování / vysvětlení statusů.

### 5.9 `dimRelativeMonth`

Grain:

```text
1 řádek = 1 MONTHS_BALANCE
```

Účel:

- relativní časová osa vůči aktuální žádosti,
- používá se místo kalendářní tabulky.

---

## 6. Proč není použita kalendářní tabulka `Date[Date]`

V reportu existuje kalendářní tabulka:

```text
Date[Date]
```

Pro toto zadání však není vytvořen vztah na kalendářní tabulku, protože použité Home Credit soubory v řešených úlohách neobsahují skutečné kalendářní datum žádosti nebo transakce.

Používají relativní časové sloupce, například:

```text
DAYS_BIRTH
DAYS_INSTALMENT
DAYS_ENTRY_PAYMENT
MONTHS_BALANCE
```

Pro úlohu 3 je časová osa výslovně zadána jako:

```text
relative to application time
```

Proto je správná časová dimenze:

```text
dimRelativeMonth
```

a nikoli:

```text
Date[Date]
```

Kalendářní tabulka by byla vhodná pouze v případě, že by v modelu existoval skutečný datumový sloupec, který reprezentuje reálný kalendářní den. V tomto modelu pro aktuální zadání takový datumový sloupec nepoužíváme.

---

## 7. Power Query logika fact tabulek

### 7.1 `factApplications`

Zdroj:

```text
application_train.csv
```

Načítají se pouze potřebné sloupce:

```text
SK_ID_CURR
TARGET
NAME_CONTRACT_TYPE
CODE_GENDER
DAYS_BIRTH
NAME_EDUCATION_TYPE
NAME_FAMILY_STATUS
NAME_INCOME_TYPE
AMT_CREDIT
AMT_INCOME_TOTAL
```

Odvozené sloupce:

```text
Gender
Age Years
Age Group
Age Group Sort
Is Cash Application
Application Unit
```

Business logika:

- `Gender` převádí `CODE_GENDER` na čitelný popisek.
- `Age Years` se počítá z absolutní hodnoty `DAYS_BIRTH / 365.25`.
- `Age Group` vytváří věkové skupiny.
- `Age Group Sort` zajišťuje správné řazení věkových skupin.
- `Is Cash Application` označuje žádosti s `NAME_CONTRACT_TYPE = "Cash loans"`.
- `Application Unit = 1` umožňuje jednoduché počítání žádostí.

M code:

```powerquery
let
    Source = Csv.Document(
        File.Contents("C:\Users\cznvosi\Downloads\home-credit-default-risk\application_train.csv"),
        [
            Delimiter = ",",
            Columns = 122,
            Encoding = 1252,
            QuoteStyle = QuoteStyle.None
        ]
    ),

    #"Promoted Headers" = Table.PromoteHeaders(
        Source,
        [PromoteAllScalars = true]
    ),

    #"Selected Needed Columns" = Table.SelectColumns(
        #"Promoted Headers",
        {
            "SK_ID_CURR",
            "TARGET",
            "NAME_CONTRACT_TYPE",
            "CODE_GENDER",
            "DAYS_BIRTH",
            "NAME_EDUCATION_TYPE",
            "NAME_FAMILY_STATUS",
            "NAME_INCOME_TYPE",
            "AMT_CREDIT",
            "AMT_INCOME_TOTAL"
        },
        MissingField.Error
    ),

    #"Changed Type" = Table.TransformColumnTypes(
        #"Selected Needed Columns",
        {
            {"SK_ID_CURR", Int64.Type},
            {"TARGET", Int64.Type},
            {"NAME_CONTRACT_TYPE", type text},
            {"CODE_GENDER", type text},
            {"DAYS_BIRTH", Int64.Type},
            {"NAME_EDUCATION_TYPE", type text},
            {"NAME_FAMILY_STATUS", type text},
            {"NAME_INCOME_TYPE", type text},
            {"AMT_CREDIT", type number},
            {"AMT_INCOME_TOTAL", type number}
        }
    ),

    #"Trimmed Text Columns" = Table.TransformColumns(
        #"Changed Type",
        {
            {"NAME_CONTRACT_TYPE", each if _ = null then null else Text.Trim(_), type text},
            {"CODE_GENDER", each if _ = null then null else Text.Trim(_), type text},
            {"NAME_EDUCATION_TYPE", each if _ = null then null else Text.Trim(_), type text},
            {"NAME_FAMILY_STATUS", each if _ = null then null else Text.Trim(_), type text},
            {"NAME_INCOME_TYPE", each if _ = null then null else Text.Trim(_), type text}
        }
    ),

    #"Added Gender Label" = Table.AddColumn(
        #"Trimmed Text Columns",
        "Gender",
        each
            if [CODE_GENDER] = "M" then "Male"
            else if [CODE_GENDER] = "F" then "Female"
            else "Unknown",
        type text
    ),

    #"Added Age Years" = Table.AddColumn(
        #"Added Gender Label",
        "Age Years",
        each
            if [DAYS_BIRTH] = null then null
            else Number.RoundDown(Number.Abs([DAYS_BIRTH]) / 365.25),
        Int64.Type
    ),

    #"Added Age Group" = Table.AddColumn(
        #"Added Age Years",
        "Age Group",
        each
            if [Age Years] = null then "Unknown"
            else if [Age Years] < 25 then "18-24"
            else if [Age Years] < 35 then "25-34"
            else if [Age Years] < 45 then "35-44"
            else if [Age Years] < 55 then "45-54"
            else if [Age Years] < 65 then "55-64"
            else "65+",
        type text
    ),

    #"Added Age Group Sort" = Table.AddColumn(
        #"Added Age Group",
        "Age Group Sort",
        each
            if [Age Group] = "18-24" then 1
            else if [Age Group] = "25-34" then 2
            else if [Age Group] = "35-44" then 3
            else if [Age Group] = "45-54" then 4
            else if [Age Group] = "55-64" then 5
            else if [Age Group] = "65+" then 6
            else 99,
        Int64.Type
    ),

    #"Added Cash Application Flag" = Table.AddColumn(
        #"Added Age Group Sort",
        "Is Cash Application",
        each [NAME_CONTRACT_TYPE] = "Cash loans",
        type logical
    ),

    #"Added Application Unit" = Table.AddColumn(
        #"Added Cash Application Flag",
        "Application Unit",
        each 1,
        Int64.Type
    ),

    #"Reordered Columns" = Table.ReorderColumns(
        #"Added Application Unit",
        {
            "SK_ID_CURR",
            "TARGET",
            "Application Unit",
            "AMT_CREDIT",
            "AMT_INCOME_TOTAL",
            "NAME_CONTRACT_TYPE",
            "Is Cash Application",
            "CODE_GENDER",
            "Gender",
            "DAYS_BIRTH",
            "Age Years",
            "Age Group",
            "Age Group Sort",
            "NAME_EDUCATION_TYPE",
            "NAME_FAMILY_STATUS",
            "NAME_INCOME_TYPE"
        }
    )
in
    #"Reordered Columns"
```

---

### 7.2 `factBureauCredits`

Zdroj:

```text
bureau.csv
```

Načítané sloupce:

```text
SK_ID_CURR
SK_ID_BUREAU
CREDIT_ACTIVE
CREDIT_TYPE
AMT_CREDIT_SUM
```

Odvozené sloupce:

```text
Is Active Bureau Contract
Active Bureau Credit Sum
Bureau Credit Unit
```

Business logika:

- `Is Active Bureau Contract` je TRUE pro `CREDIT_ACTIVE = "Active"`.
- `Active Bureau Credit Sum` je `AMT_CREDIT_SUM` pouze pro aktivní kontrakty, jinak 0.
- `Bureau Credit Unit = 1` slouží k počítání kontraktů.
- `CREDIT_TYPE` je ponechán pro filtr ve třetí úloze.
- `SK_ID_BUREAU` slouží k napojení na `factBureauMonthlyStatus`.

M code:

```powerquery
let
    Source = Csv.Document(
        File.Contents("C:\Users\cznvosi\Downloads\home-credit-default-risk\bureau.csv"),
        [
            Delimiter = ",",
            Columns = 17,
            Encoding = 1252,
            QuoteStyle = QuoteStyle.None
        ]
    ),

    #"Promoted Headers" = Table.PromoteHeaders(
        Source,
        [PromoteAllScalars = true]
    ),

    #"Selected Needed Columns" = Table.SelectColumns(
        #"Promoted Headers",
        {
            "SK_ID_CURR",
            "SK_ID_BUREAU",
            "CREDIT_ACTIVE",
            "CREDIT_TYPE",
            "AMT_CREDIT_SUM"
        },
        MissingField.Error
    ),

    #"Changed Type" = Table.TransformColumnTypes(
        #"Selected Needed Columns",
        {
            {"SK_ID_CURR", Int64.Type},
            {"SK_ID_BUREAU", Int64.Type},
            {"CREDIT_ACTIVE", type text},
            {"CREDIT_TYPE", type text},
            {"AMT_CREDIT_SUM", type number}
        }
    ),

    #"Trimmed Text Columns" = Table.TransformColumns(
        #"Changed Type",
        {
            {"CREDIT_ACTIVE", each if _ = null then null else Text.Trim(_), type text},
            {"CREDIT_TYPE", each if _ = null then null else Text.Trim(_), type text}
        }
    ),

    #"Replaced Null Credit Sum" = Table.ReplaceValue(
        #"Trimmed Text Columns",
        null,
        0,
        Replacer.ReplaceValue,
        {"AMT_CREDIT_SUM"}
    ),

    #"Added Active Contract Flag" = Table.AddColumn(
        #"Replaced Null Credit Sum",
        "Is Active Bureau Contract",
        each [CREDIT_ACTIVE] = "Active",
        type logical
    ),

    #"Added Active Credit Sum" = Table.AddColumn(
        #"Added Active Contract Flag",
        "Active Bureau Credit Sum",
        each
            if [Is Active Bureau Contract] = true then [AMT_CREDIT_SUM]
            else 0,
        type number
    ),

    #"Added Bureau Credit Unit" = Table.AddColumn(
        #"Added Active Credit Sum",
        "Bureau Credit Unit",
        each 1,
        Int64.Type
    ),

    #"Reordered Columns" = Table.ReorderColumns(
        #"Added Bureau Credit Unit",
        {
            "SK_ID_CURR",
            "SK_ID_BUREAU",
            "Bureau Credit Unit",
            "AMT_CREDIT_SUM",
            "Active Bureau Credit Sum",
            "CREDIT_ACTIVE",
            "Is Active Bureau Contract",
            "CREDIT_TYPE"
        }
    )
in
    #"Reordered Columns"
```

---

### 7.3 `factBureauMonthlyStatus`

Zdroj:

```text
bureau_balance.csv
```

Načítané sloupce:

```text
SK_ID_BUREAU
MONTHS_BALANCE
STATUS
```

Odvozené sloupce:

```text
Bureau Monthly Status Unit
Is No DPD
No DPD Bureau Contract Unit
Bureau Status Label
Bureau Status Sort
Months Before Application
Relative Month Label
```

Business logika:

- `STATUS = "0"` znamená no-DPD / bez prodlení.
- `MONTHS_BALANCE` je relativní měsíc vůči žádosti.
- `Months Before Application` převádí záporné měsíce na kladný počet měsíců před žádostí.
- `Relative Month Label` umožňuje zobrazit osu jako relativní čas.
- `Bureau Status Label` vysvětluje hodnoty `STATUS`.

M code:

```powerquery
let
    Source = Csv.Document(
        File.Contents("C:\Users\cznvosi\Downloads\home-credit-default-risk\bureau_balance.csv"),
        [
            Delimiter = ",",
            Columns = 3,
            Encoding = 1252,
            QuoteStyle = QuoteStyle.None
        ]
    ),

    #"Promoted Headers" = Table.PromoteHeaders(
        Source,
        [PromoteAllScalars = true]
    ),

    #"Selected Needed Columns" = Table.SelectColumns(
        #"Promoted Headers",
        {
            "SK_ID_BUREAU",
            "MONTHS_BALANCE",
            "STATUS"
        },
        MissingField.Error
    ),

    #"Changed Type" = Table.TransformColumnTypes(
        #"Selected Needed Columns",
        {
            {"SK_ID_BUREAU", Int64.Type},
            {"MONTHS_BALANCE", Int64.Type},
            {"STATUS", type text}
        }
    ),

    #"Trimmed Status" = Table.TransformColumns(
        #"Changed Type",
        {
            {"STATUS", each if _ = null then null else Text.Trim(_), type text}
        }
    ),

    #"Added Bureau Monthly Status Unit" = Table.AddColumn(
        #"Trimmed Status",
        "Bureau Monthly Status Unit",
        each 1,
        Int64.Type
    ),

    #"Added No DPD Flag" = Table.AddColumn(
        #"Added Bureau Monthly Status Unit",
        "Is No DPD",
        each [STATUS] = "0",
        type logical
    ),

    #"Added No DPD Bureau Contract Unit" = Table.AddColumn(
        #"Added No DPD Flag",
        "No DPD Bureau Contract Unit",
        each if [Is No DPD] = true then 1 else 0,
        Int64.Type
    ),

    #"Added Bureau Status Label" = Table.AddColumn(
        #"Added No DPD Bureau Contract Unit",
        "Bureau Status Label",
        each
            if [STATUS] = "0" then "No DPD"
            else if [STATUS] = "1" then "DPD 1-30"
            else if [STATUS] = "2" then "DPD 31-60"
            else if [STATUS] = "3" then "DPD 61-90"
            else if [STATUS] = "4" then "DPD 91-120"
            else if [STATUS] = "5" then "DPD 120+"
            else if [STATUS] = "C" then "Closed"
            else if [STATUS] = "X" then "Unknown"
            else "Other / Missing",
        type text
    ),

    #"Added Bureau Status Sort" = Table.AddColumn(
        #"Added Bureau Status Label",
        "Bureau Status Sort",
        each
            if [STATUS] = "0" then 1
            else if [STATUS] = "1" then 2
            else if [STATUS] = "2" then 3
            else if [STATUS] = "3" then 4
            else if [STATUS] = "4" then 5
            else if [STATUS] = "5" then 6
            else if [STATUS] = "C" then 7
            else if [STATUS] = "X" then 8
            else 99,
        Int64.Type
    ),

    #"Added Months Before Application" = Table.AddColumn(
        #"Added Bureau Status Sort",
        "Months Before Application",
        each Number.Abs([MONTHS_BALANCE]),
        Int64.Type
    ),

    #"Added Relative Month Label" = Table.AddColumn(
        #"Added Months Before Application",
        "Relative Month Label",
        each "M" & Text.From([MONTHS_BALANCE]),
        type text
    ),

    #"Reordered Columns" = Table.ReorderColumns(
        #"Added Relative Month Label",
        {
            "SK_ID_BUREAU",
            "MONTHS_BALANCE",
            "Months Before Application",
            "Relative Month Label",
            "STATUS",
            "Bureau Status Label",
            "Bureau Status Sort",
            "Is No DPD",
            "No DPD Bureau Contract Unit",
            "Bureau Monthly Status Unit"
        }
    )
in
    #"Reordered Columns"
```

---

### 7.4 `factPreviousApplications`

Zdroj:

```text
previous_application.csv
```

Načítané sloupce:

```text
SK_ID_CURR
SK_ID_PREV
```

Odvozené sloupce:

```text
Previous Application Unit
```

Business logika:

- Tabulka slouží pouze ke zjištění počtu předchozích žádostí.
- Detailní atributy předchozí žádosti nebyly pro aktuální úlohy potřeba.
- Výpočet bucketu počtu předchozích žádostí probíhá v `dimApplicant`.

M code:

```powerquery
let
    Source = Csv.Document(
        File.Contents("C:\Users\cznvosi\Downloads\home-credit-default-risk\previous_application.csv"),
        [
            Delimiter = ",",
            Columns = 37,
            Encoding = 1252,
            QuoteStyle = QuoteStyle.None
        ]
    ),

    #"Promoted Headers" = Table.PromoteHeaders(
        Source,
        [PromoteAllScalars = true]
    ),

    #"Selected Needed Columns" = Table.SelectColumns(
        #"Promoted Headers",
        {
            "SK_ID_CURR",
            "SK_ID_PREV"
        },
        MissingField.Error
    ),

    #"Changed Type" = Table.TransformColumnTypes(
        #"Selected Needed Columns",
        {
            {"SK_ID_CURR", Int64.Type},
            {"SK_ID_PREV", Int64.Type}
        }
    ),

    #"Removed Rows With Missing Keys" = Table.SelectRows(
        #"Changed Type",
        each [SK_ID_CURR] <> null and [SK_ID_PREV] <> null
    ),

    #"Removed Duplicate Previous Applications" = Table.Distinct(
        #"Removed Rows With Missing Keys",
        {"SK_ID_PREV"}
    ),

    #"Train Application Keys" = Table.Distinct(
        Table.SelectColumns(
            factApplications,
            {"SK_ID_CURR"}
        )
    ),

    #"Merged With Train Applications" = Table.NestedJoin(
        #"Removed Duplicate Previous Applications",
        {"SK_ID_CURR"},
        #"Train Application Keys",
        {"SK_ID_CURR"},
        "TrainApplications",
        JoinKind.Inner
    ),

    #"Filtered To Train Applications" = Table.RemoveColumns(
        #"Merged With Train Applications",
        {"TrainApplications"}
    ),

    #"Added Previous Application Unit" = Table.AddColumn(
        #"Filtered To Train Applications",
        "Previous Application Unit",
        each 1,
        Int64.Type
    ),

    #"Reordered Columns" = Table.ReorderColumns(
        #"Added Previous Application Unit",
        {
            "SK_ID_CURR",
            "SK_ID_PREV",
            "Previous Application Unit"
        }
    )
in
    #"Reordered Columns"
```

---

## 8. Databricks SQL logika pro `factInstallmentDPD5Summary`

Raw soubor `installments_payments.csv` nebyl načítán do Power Query, protože jeho zpracování bylo v Power BI příliš pomalé. V konverzaci bylo potvrzeno, že se načítání zasekávalo přibližně na velikosti 690 MB z celkové velikosti 706 MB. Proto byla logika přesunuta do Databricks SQL a do Power BI se načítá pouze agregovaný výstup.

SQL query:

```sql
WITH train_applications AS (

    SELECT DISTINCT
        CAST(SK_ID_CURR AS BIGINT) AS SK_ID_CURR
    FROM home_credit.application_train

),

installment_rows AS (

    SELECT
        CAST(ip.SK_ID_CURR AS BIGINT) AS SK_ID_CURR,
        CAST(ip.SK_ID_PREV AS BIGINT) AS SK_ID_PREV,
        CAST(ip.NUM_INSTALMENT_VERSION AS DOUBLE) AS NUM_INSTALMENT_VERSION,
        CAST(ip.NUM_INSTALMENT_NUMBER AS BIGINT) AS NUM_INSTALMENT_NUMBER,
        CAST(ip.DAYS_INSTALMENT AS BIGINT) AS DAYS_INSTALMENT,
        CAST(ip.DAYS_ENTRY_PAYMENT AS BIGINT) AS DAYS_ENTRY_PAYMENT,
        CAST(ip.AMT_INSTALMENT AS DOUBLE) AS AMT_INSTALMENT,
        CAST(ip.AMT_PAYMENT AS DOUBLE) AS AMT_PAYMENT
    FROM home_credit.installments_payments ip
    INNER JOIN train_applications ta
        ON CAST(ip.SK_ID_CURR AS BIGINT) = ta.SK_ID_CURR
    WHERE ip.SK_ID_CURR IS NOT NULL
      AND ip.SK_ID_PREV IS NOT NULL
      AND ip.NUM_INSTALMENT_VERSION IS NOT NULL
      AND ip.NUM_INSTALMENT_NUMBER IS NOT NULL
      AND ip.DAYS_INSTALMENT IS NOT NULL
      AND ip.AMT_INSTALMENT IS NOT NULL

),

latest_version_rows AS (

    SELECT
        *,
        MAX(NUM_INSTALMENT_VERSION) OVER (
            PARTITION BY SK_ID_CURR, SK_ID_PREV, NUM_INSTALMENT_NUMBER
        ) AS LATEST_INSTALMENT_VERSION
    FROM installment_rows

),

latest_installment_rows AS (

    SELECT
        SK_ID_CURR,
        SK_ID_PREV,
        NUM_INSTALMENT_VERSION,
        NUM_INSTALMENT_NUMBER,
        DAYS_INSTALMENT,
        DAYS_ENTRY_PAYMENT,
        AMT_INSTALMENT,
        AMT_PAYMENT
    FROM latest_version_rows
    WHERE NUM_INSTALMENT_VERSION = LATEST_INSTALMENT_VERSION

),

installment_level AS (

    SELECT
        SK_ID_CURR,
        SK_ID_PREV,
        NUM_INSTALMENT_NUMBER,
        MAX(NUM_INSTALMENT_VERSION) AS NUM_INSTALMENT_VERSION,
        MAX(DAYS_INSTALMENT) AS DAYS_INSTALMENT,
        MAX(AMT_INSTALMENT) AS AMT_INSTALMENT,

        SUM(
            CASE
                WHEN DAYS_ENTRY_PAYMENT IS NOT NULL
                 AND AMT_PAYMENT IS NOT NULL
                 AND DAYS_ENTRY_PAYMENT <= DAYS_INSTALMENT + 5
                THEN AMT_PAYMENT
                ELSE 0
            END
        ) AS PAID_WITHIN_DPD5_WINDOW

    FROM latest_installment_rows
    GROUP BY
        SK_ID_CURR,
        SK_ID_PREV,
        NUM_INSTALMENT_NUMBER

),

installment_dpd5 AS (

    SELECT
        SK_ID_CURR,
        SK_ID_PREV,
        NUM_INSTALMENT_NUMBER,
        NUM_INSTALMENT_VERSION,
        DAYS_INSTALMENT,
        AMT_INSTALMENT,
        PAID_WITHIN_DPD5_WINDOW,

        CASE
            WHEN PAID_WITHIN_DPD5_WINDOW < AMT_INSTALMENT THEN TRUE
            ELSE FALSE
        END AS IS_DPD5,

        CASE
            WHEN PAID_WITHIN_DPD5_WINDOW < AMT_INSTALMENT THEN 1
            ELSE 0
        END AS DPD5_UNIT

    FROM installment_level

),

client_level AS (

    SELECT
        SK_ID_CURR,
        COUNT(*) AS INSTALLMENT_COUNT,
        SUM(DPD5_UNIT) AS DPD5_INSTALLMENT_COUNT,
        MAX(DPD5_UNIT) AS EVER_DPD5_UNIT
    FROM installment_dpd5
    GROUP BY
        SK_ID_CURR

)

SELECT
    ta.SK_ID_CURR,

    COALESCE(cl.INSTALLMENT_COUNT, 0) AS InstallmentCount,
    COALESCE(cl.DPD5_INSTALLMENT_COUNT, 0) AS DPD5InstallmentCount,

    CASE
        WHEN COALESCE(cl.EVER_DPD5_UNIT, 0) = 1 THEN TRUE
        ELSE FALSE
    END AS EverDPD5,

    COALESCE(cl.EVER_DPD5_UNIT, 0) AS EverDPD5Unit

FROM train_applications ta
LEFT JOIN client_level cl
    ON ta.SK_ID_CURR = cl.SK_ID_CURR
```

Význam výstupních sloupců:

| Sloupec | Význam |
|---|---|
| `SK_ID_CURR` | Klíč TRAIN žádosti |
| `InstallmentCount` | Počet historických splátek v nejnovější verzi |
| `DPD5InstallmentCount` | Počet splátek, kde nastalo DPD5 |
| `EverDPD5` | TRUE, pokud klient někdy měl DPD5 |
| `EverDPD5Unit` | Číselná verze příznaku pro agregaci |

---

## 9. Power Query logika dimenzí

### 9.1 `dimApplicant`

Účel:

- centrální dimenze pro TRAIN žádosti,
- obsahuje všechny demografické atributy,
- obsahuje bucket počtu předchozích žádostí.

M code:

```powerquery
let
    Source = factApplications,

    #"Selected Needed Columns" = Table.SelectColumns(
        Source,
        {
            "SK_ID_CURR",
            "TARGET",
            "NAME_CONTRACT_TYPE",
            "Is Cash Application",
            "CODE_GENDER",
            "Gender",
            "Age Years",
            "Age Group",
            "Age Group Sort",
            "NAME_EDUCATION_TYPE",
            "NAME_FAMILY_STATUS",
            "NAME_INCOME_TYPE"
        },
        MissingField.Error
    ),

    #"Removed Duplicate Applicants" = Table.Distinct(
        #"Selected Needed Columns",
        {"SK_ID_CURR"}
    ),

    #"Renamed Columns" = Table.RenameColumns(
        #"Removed Duplicate Applicants",
        {
            {"NAME_CONTRACT_TYPE", "Contract Type"},
            {"NAME_EDUCATION_TYPE", "Education Type"},
            {"NAME_FAMILY_STATUS", "Family Status"},
            {"NAME_INCOME_TYPE", "Income Type"}
        }
    ),

    #"Added Target Label" = Table.AddColumn(
        #"Renamed Columns",
        "Target Label",
        each
            if [TARGET] = 0 then "No Repayment Difficulty"
            else if [TARGET] = 1 then "Repayment Difficulty"
            else "Unknown",
        type text
    ),

    #"Previous Application Counts" = Table.Group(
        factPreviousApplications,
        {"SK_ID_CURR"},
        {
            {
                "Previous Application Count",
                each Table.RowCount(_),
                Int64.Type
            }
        }
    ),

    #"Merged Previous Application Counts" = Table.NestedJoin(
        #"Added Target Label",
        {"SK_ID_CURR"},
        #"Previous Application Counts",
        {"SK_ID_CURR"},
        "PreviousApplicationCounts",
        JoinKind.LeftOuter
    ),

    #"Expanded Previous Application Counts" = Table.ExpandTableColumn(
        #"Merged Previous Application Counts",
        "PreviousApplicationCounts",
        {"Previous Application Count"},
        {"Previous Application Count"}
    ),

    #"Replaced Null Previous Application Count" = Table.ReplaceValue(
        #"Expanded Previous Application Counts",
        null,
        0,
        Replacer.ReplaceValue,
        {"Previous Application Count"}
    ),

    #"Changed Previous Application Count Type" = Table.TransformColumnTypes(
        #"Replaced Null Previous Application Count",
        {
            {"Previous Application Count", Int64.Type}
        }
    ),

    #"Added Previous Application Count Band" = Table.AddColumn(
        #"Changed Previous Application Count Type",
        "Previous Application Count Band",
        each
            if [Previous Application Count] = 0 then "0"
            else if [Previous Application Count] = 1 then "1"
            else if [Previous Application Count] <= 3 then "2-3"
            else if [Previous Application Count] <= 5 then "4-5"
            else "6+",
        type text
    ),

    #"Added Previous Application Count Band Sort" = Table.AddColumn(
        #"Added Previous Application Count Band",
        "Previous Application Count Band Sort",
        each
            if [Previous Application Count Band] = "0" then 1
            else if [Previous Application Count Band] = "1" then 2
            else if [Previous Application Count Band] = "2-3" then 3
            else if [Previous Application Count Band] = "4-5" then 4
            else if [Previous Application Count Band] = "6+" then 5
            else 99,
        Int64.Type
    ),

    #"Reordered Columns" = Table.ReorderColumns(
        #"Added Previous Application Count Band Sort",
        {
            "SK_ID_CURR",
            "TARGET",
            "Target Label",
            "Contract Type",
            "Is Cash Application",
            "CODE_GENDER",
            "Gender",
            "Age Years",
            "Age Group",
            "Age Group Sort",
            "Education Type",
            "Family Status",
            "Income Type",
            "Previous Application Count",
            "Previous Application Count Band",
            "Previous Application Count Band Sort"
        }
    )
in
    #"Reordered Columns"
```

---

### 9.2 `dimCreditType`

```powerquery
let
    Source = factBureauCredits,

    #"Selected Needed Columns" = Table.SelectColumns(
        Source,
        {
            "CREDIT_TYPE"
        },
        MissingField.Error
    ),

    #"Removed Duplicates" = Table.Distinct(
        #"Selected Needed Columns"
    ),

    #"Added Credit Type Label" = Table.AddColumn(
        #"Removed Duplicates",
        "Credit Type",
        each
            if [CREDIT_TYPE] = null or Text.Trim([CREDIT_TYPE]) = "" then "Unknown"
            else Text.Trim([CREDIT_TYPE]),
        type text
    ),

    #"Sorted Rows" = Table.Sort(
        #"Added Credit Type Label",
        {
            {"Credit Type", Order.Ascending}
        }
    )
in
    #"Sorted Rows"
```

---

### 9.3 `dimBureauStatus`

```powerquery
let
    Source = factBureauMonthlyStatus,

    #"Selected Needed Columns" = Table.SelectColumns(
        Source,
        {
            "STATUS",
            "Bureau Status Label",
            "Bureau Status Sort"
        },
        MissingField.Error
    ),

    #"Removed Duplicates" = Table.Distinct(
        #"Selected Needed Columns"
    ),

    #"Sorted Rows" = Table.Sort(
        #"Removed Duplicates",
        {
            {"Bureau Status Sort", Order.Ascending}
        }
    )
in
    #"Sorted Rows"
```

---

### 9.4 `dimRelativeMonth`

```powerquery
let
    Source = factBureauMonthlyStatus,

    #"Selected Needed Columns" = Table.SelectColumns(
        Source,
        {
            "MONTHS_BALANCE",
            "Months Before Application",
            "Relative Month Label"
        },
        MissingField.Error
    ),

    #"Removed Duplicates" = Table.Distinct(
        #"Selected Needed Columns"
    ),

    #"Added Month Description" = Table.AddColumn(
        #"Removed Duplicates",
        "Relative Month Description",
        each
            if [MONTHS_BALANCE] = 0 then "Application month"
            else Text.From([Months Before Application]) & " month(s) before application",
        type text
    ),

    #"Sorted Rows" = Table.Sort(
        #"Added Month Description",
        {
            {"MONTHS_BALANCE", Order.Ascending}
        }
    )
in
    #"Sorted Rows"
```

---

## 10. Vztahy v modelu

Finální vztahy:

```text
dimApplicant[SK_ID_CURR] 1 → * factApplications[SK_ID_CURR]
dimApplicant[SK_ID_CURR] 1 → * factBureauCredits[SK_ID_CURR]
dimApplicant[SK_ID_CURR] 1 → * factPreviousApplications[SK_ID_CURR]
dimApplicant[SK_ID_CURR] 1 → * factInstallmentDPD5Summary[SK_ID_CURR]

factBureauCredits[SK_ID_BUREAU] 1 → * factBureauMonthlyStatus[SK_ID_BUREAU]

dimCreditType[CREDIT_TYPE] 1 → * factBureauCredits[CREDIT_TYPE]
dimBureauStatus[STATUS] 1 → * factBureauMonthlyStatus[STATUS]
dimRelativeMonth[MONTHS_BALANCE] 1 → * factBureauMonthlyStatus[MONTHS_BALANCE]
```

Vztahy jsou koncipované jako single-direction z dimenze do fact tabulky, případně z bureau kontraktů do jejich měsíční historie.

TMDL:

```tmdl
createOrReplace

	relationship 6f1e21c7-7d7f-4c2f-a23e-111111111001
		fromColumn: factApplications.SK_ID_CURR
		toColumn: dimApplicant.SK_ID_CURR

	relationship 6f1e21c7-7d7f-4c2f-a23e-111111111002
		fromColumn: factBureauCredits.SK_ID_CURR
		toColumn: dimApplicant.SK_ID_CURR

	relationship 6f1e21c7-7d7f-4c2f-a23e-111111111003
		fromColumn: factPreviousApplications.SK_ID_CURR
		toColumn: dimApplicant.SK_ID_CURR

	relationship 6f1e21c7-7d7f-4c2f-a23e-111111111004
		fromColumn: factInstallmentDPD5Summary.SK_ID_CURR
		toColumn: dimApplicant.SK_ID_CURR

	relationship 6f1e21c7-7d7f-4c2f-a23e-111111111005
		fromColumn: factBureauMonthlyStatus.SK_ID_BUREAU
		toColumn: factBureauCredits.SK_ID_BUREAU

	relationship 6f1e21c7-7d7f-4c2f-a23e-111111111006
		fromColumn: factBureauCredits.CREDIT_TYPE
		toColumn: dimCreditType.CREDIT_TYPE

	relationship 6f1e21c7-7d7f-4c2f-a23e-111111111007
		fromColumn: factBureauMonthlyStatus.STATUS
		toColumn: dimBureauStatus.STATUS

	relationship 6f1e21c7-7d7f-4c2f-a23e-111111111008
		fromColumn: factBureauMonthlyStatus.MONTHS_BALANCE
		toColumn: dimRelativeMonth.MONTHS_BALANCE
```

Poznámka:

V TMDL skriptu byly použity názvy tabulek bez mezer, například:

```text
factBureauCredits
```

nikoli:

```text
Fact Bureau Credits
```

protože tak byl model pojmenován v reportu.

---

## 11. Míry v tabulce `Measures Technical`

Míry jsou uloženy v jedné měrové tabulce:

```text
Measures Technical
```

Měry jsou organizované do složek:

```text
01 Základní metriky
02 Distribuce
03 DTI
04 Bureau trend
05 Bonus DPD5
```

### 11.1 Základní metriky

```DAX
Applications Count =
COUNTROWS ( factApplications )
```

Význam:

- počet TRAIN žádostí,
- používá se jako základní denominator.

```DAX
Credit Volume =
SUM ( factApplications[AMT_CREDIT] )
```

Význam:

- objem požadovaných úvěrů,
- používá se jako váha pro objemovou distribuci.

```DAX
Average Credit Amount =
DIVIDE (
    [Credit Volume],
    [Applications Count]
)
```

Význam:

- průměrná výše požadovaného úvěru.

```DAX
Annual Income =
SUM ( factApplications[AMT_INCOME_TOTAL] )
```

Význam:

- roční příjem v aktuálním filtračním kontextu.

```DAX
Current Application Credit =
SUM ( factApplications[AMT_CREDIT] )
```

Význam:

- aktuálně požadovaný úvěr v aktuálním kontextu.

```DAX
Repayment Difficulty Applications =
CALCULATE (
    [Applications Count],
    dimApplicant[TARGET] = 1
)
```

Význam:

- počet žádostí s `TARGET = 1`.

```DAX
No Repayment Difficulty Applications =
CALCULATE (
    [Applications Count],
    dimApplicant[TARGET] = 0
)
```

Význam:

- počet žádostí s `TARGET = 0`.

```DAX
Repayment Difficulty Rate =
DIVIDE (
    [Repayment Difficulty Applications],
    [Applications Count]
)
```

Význam:

- podíl žádostí s problémem se splácením.

---

### 11.2 Distribuce

```DAX
Applications Visible Total =
CALCULATE (
    [Applications Count],
    ALLSELECTED ()
)
```

Význam:

- denominator pro jednotkovou distribuci v rámci aktuálního reportového výběru.

```DAX
Applications % of Visible Total =
DIVIDE (
    [Applications Count],
    [Applications Visible Total]
)
```

Význam:

- jednotkový podíl skupiny na celkovém počtu žádostí.

```DAX
Credit Volume Visible Total =
CALCULATE (
    [Credit Volume],
    ALLSELECTED ()
)
```

Význam:

- denominator pro objemovou distribuci.

```DAX
Credit Volume % of Visible Total =
DIVIDE (
    [Credit Volume],
    [Credit Volume Visible Total]
)
```

Význam:

- objemový podíl skupiny na celkovém objemu `AMT_CREDIT`.

---

### 11.3 DTI

```DAX
Active Bureau Credit Exposure =
COALESCE (
    CALCULATE (
        SUM ( factBureauCredits[Active Bureau Credit Sum] ),
        REMOVEFILTERS ( dimCreditType )
    ),
    0
)
```

Význam:

- součet aktivní credit bureau expozice,
- `REMOVEFILTERS(dimCreditType)` je použit proto, aby DTI nebylo zkresleno filtrem typu bureau úvěru, pokud je filtr použit pro jinou vizualizaci.

```DAX
DTI =
DIVIDE (
    [Active Bureau Credit Exposure] + [Current Application Credit],
    [Annual Income]
)
```

Význam:

- Debt To Income podle definice v zadání.

```DAX
Median DTI =
MEDIANX (
    FILTER (
        VALUES ( dimApplicant[SK_ID_CURR] ),
        CALCULATE ( SUM ( factApplications[AMT_INCOME_TOTAL] ) ) > 0
    ),
    [DTI]
)
```

Význam:

- medián klientského DTI,
- počítá se přes jednotlivé `SK_ID_CURR`,
- vylučuje případy, kde příjem není větší než 0.

---

### 11.4 Bureau trend

```DAX
Cash Applications Count =
CALCULATE (
    [Applications Count],
    dimApplicant[Is Cash Application] = TRUE ()
)
```

Význam:

- počet TRAIN cash žádostí.

```DAX
Cash Applications Count - Trend Denominator =
CALCULATE (
    [Applications Count],
    dimApplicant[Is Cash Application] = TRUE (),
    REMOVEFILTERS ( dimCreditType ),
    REMOVEFILTERS ( dimBureauStatus ),
    REMOVEFILTERS ( dimRelativeMonth ),
    REMOVEFILTERS ( factBureauCredits ),
    REMOVEFILTERS ( factBureauMonthlyStatus )
)
```

Význam:

- pevný denominator pro trend,
- počítá všechny TRAIN cash žádosti,
- ignoruje filtry z bureau historie a relativního měsíce.

```DAX
Bureau Contracts Count =
DISTINCTCOUNT ( factBureauCredits[SK_ID_BUREAU] )
```

Význam:

- počet unikátních bureau kontraktů.

```DAX
No DPD Bureau Contracts =
CALCULATE (
    DISTINCTCOUNT ( factBureauMonthlyStatus[SK_ID_BUREAU] ),
    REMOVEFILTERS ( dimBureauStatus ),
    factBureauMonthlyStatus[STATUS] = "0"
)
```

Význam:

- počet unikátních bureau kontraktů v no-DPD stavu,
- no-DPD je definováno jako `STATUS = "0"`.

```DAX
No DPD Bureau Contracts - Cash Applications =
CALCULATE (
    [No DPD Bureau Contracts],
    dimApplicant[Is Cash Application] = TRUE ()
)
```

Význam:

- no-DPD kontrakty pouze pro cash žádosti.

```DAX
Avg No DPD Bureau Contracts per Cash Application =
DIVIDE (
    [No DPD Bureau Contracts - Cash Applications],
    [Cash Applications Count - Trend Denominator]
)
```

Význam:

- finální metrika pro úlohu 3,
- průměrný počet no-DPD bureau kontraktů na jednu TRAIN cash žádost v relativním čase.

---

### 11.5 Bonus DPD5

```DAX
Installment Count =
SUM ( factInstallmentDPD5Summary[InstallmentCount] )
```

Význam:

- počet splátek z agregovaného Databricks výstupu.

```DAX
DPD5 Installment Count =
SUM ( factInstallmentDPD5Summary[DPD5InstallmentCount] )
```

Význam:

- počet splátek, kde nastalo DPD5.

```DAX
Applications Ever DPD5 =
SUM ( factInstallmentDPD5Summary[EverDPD5Unit] )
```

Význam:

- počet TRAIN žádostí, kde klient někdy měl DPD5.

```DAX
Applications Never DPD5 =
[Applications Count] - [Applications Ever DPD5]
```

Význam:

- počet TRAIN žádostí bez pozorované DPD5 historie.

```DAX
Ever DPD5 Application Ratio =
DIVIDE (
    [Applications Ever DPD5],
    [Applications Count]
)
```

Význam:

- finální metrika pro bonusovou úlohu.

```DAX
Applications With Installment History =
COUNTROWS (
    FILTER (
        factInstallmentDPD5Summary,
        factInstallmentDPD5Summary[InstallmentCount] > 0
    )
)
```

Význam:

- počet TRAIN žádostí s dostupnou splátkovou historií.

```DAX
Applications Without Installment History =
[Applications Count] - [Applications With Installment History]
```

Význam:

- počet TRAIN žádostí bez pozorované splátkové historie.

```DAX
Installment History Coverage % =
DIVIDE (
    [Applications With Installment History],
    [Applications Count]
)
```

Význam:

- pokrytí splátkové historie v TRAIN populaci.

---

## 12. Doporučené vizuály pro splnění zadání

### 12.1 Úloha 1 – distribuce TRAIN žádostí

Dimenze:

```text
dimApplicant[Gender]
dimApplicant[Age Group]
dimApplicant[Education Type]
dimApplicant[Family Status]
dimApplicant[Previous Application Count Band]
```

Míry:

```text
Applications Count
Applications % of Visible Total
Credit Volume
Credit Volume % of Visible Total
```

Interpretace:

- jednotkový pohled ukazuje počet žádostí,
- objemový pohled ukazuje objem požadovaného úvěru,
- rozdíl mezi těmito pohledy ukazuje, zda určitá skupina reprezentuje více hodnoty než počtu.

---

### 12.2 Úloha 2 – Median DTI podle Income Type

Řádky / osa:

```text
dimApplicant[Income Type]
```

Míra:

```text
Median DTI
```

Interpretace:

- vyšší DTI znamená vyšší poměr úvěrové expozice k příjmu,
- výpočet kombinuje aktivní bureau kontrakty a aktuální žádost,
- používá se medián, ne průměr.

---

### 12.3 Úloha 3 – vývoj no-DPD bureau kontraktů v čase

Osa:

```text
dimRelativeMonth[MONTHS_BALANCE]
```

nebo:

```text
dimRelativeMonth[Relative Month Label]
```

Míra:

```text
Avg No DPD Bureau Contracts per Cash Application
```

Slicer:

```text
dimCreditType[Credit Type]
```

Filtr:

```text
Cash applications jsou zahrnuty přes dimApplicant[Is Cash Application] v míře
```

Interpretace:

- metrika ukazuje průměrný počet bureau kontraktů bez prodlení,
- čas je relativní vůči aktuální žádosti,
- filtr `Credit Type` umožňuje analyzovat vývoj podle typu credit bureau kontraktu.

---

### 12.4 Bonus – DPD5 ratio

Míra:

```text
Ever DPD5 Application Ratio
```

Doplňkové míry:

```text
Applications Ever DPD5
Applications Never DPD5
Installment History Coverage %
```

Interpretace:

- podíl TRAIN žádostí, kde klient někdy měl DPD5 na předchozí Home Credit žádosti,
- denominator je celá TRAIN populace,
- klienti bez splátkové historie zůstávají v denominatoru.

---

## 13. Řazení sloupců v Power BI

Nastavit sort by column:

```text
dimApplicant[Age Group] → dimApplicant[Age Group Sort]
dimApplicant[Previous Application Count Band] → dimApplicant[Previous Application Count Band Sort]
dimBureauStatus[Bureau Status Label] → dimBureauStatus[Bureau Status Sort]
dimRelativeMonth[Relative Month Label] → dimRelativeMonth[MONTHS_BALANCE]
```

Důvod:

- bez pomocných sort sloupců by Power BI mohl řadit textové skupiny abecedně,
- například věkové skupiny nebo buckety předchozích žádostí by nebyly v logickém pořadí.

---

## 14. Důležitá omezení a vědomá rozhodnutí

### 14.1 Report není prediktivní model

Report nepredikuje default. Slouží k analýze struktury TRAIN datasetu a vybraných rizikových ukazatelů.

### 14.2 Nepoužívají se všechny dostupné Kaggle soubory

Použity jsou pouze tabulky potřebné pro zadání. Ostatní tabulky nebyly načteny, protože by zvyšovaly složitost modelu bez přímé potřeby pro úlohy.

### 14.3 `installments_payments.csv` se neagreguje v Power Query

Důvod:

- raw soubor byl v Power Query příliš pomalý,
- potřebná logika je pouze application-level DPD5 summary,
- SQL agregace v Databricks je vhodnější pro velký objem dat.

### 14.4 Kalendářní tabulka není použita

Důvod:

- relevantní časový sloupec pro úlohu 3 je relativní `MONTHS_BALANCE`,
- report nepracuje s reálným datumem.

### 14.5 `Active Bureau Credit Exposure` ignoruje filtr `Credit Type`

Důvod:

- DTI podle zadání používá všechny aktivní bureau kontrakty klienta,
- filtr typu bureau kontraktu je požadován pro úlohu 3, nikoli pro DTI,
- proto DTI nemá být náhodně změněno slicerem `Credit Type`.

---

## 15. Kontrolní seznam pro validaci reportu

### 15.1 Model

- [ ] `dimApplicant` má jeden řádek na `SK_ID_CURR`.
- [ ] `factApplications` má jeden řádek na `SK_ID_CURR`.
- [ ] `factBureauCredits` obsahuje `SK_ID_CURR` a `SK_ID_BUREAU`.
- [ ] `factBureauMonthlyStatus` obsahuje `SK_ID_BUREAU` a `MONTHS_BALANCE`.
- [ ] `factInstallmentDPD5Summary` obsahuje jeden řádek na `SK_ID_CURR`.
- [ ] Vztahy jsou vytvořeny podle sekce 10.
- [ ] Není vytvořen vztah na `Date[Date]`.

### 15.2 Vizualizace

- [ ] Úloha 1 používá jednotkové i objemové míry.
- [ ] Úloha 2 používá `Median DTI` podle `dimApplicant[Income Type]`.
- [ ] Úloha 3 používá `Avg No DPD Bureau Contracts per Cash Application` a slicer `dimCreditType[Credit Type]`.
- [ ] Bonus používá `Ever DPD5 Application Ratio`.

### 15.3 Výpočty

- [ ] `Applications Count` odpovídá počtu řádků v `factApplications`.
- [ ] `Credit Volume` odpovídá součtu `AMT_CREDIT`.
- [ ] DTI používá aktivní bureau expozici + aktuální úvěr / roční příjem.
- [ ] No-DPD používá `STATUS = "0"`.
- [ ] DPD5 používá platby do `DAYS_INSTALMENT + 5`.
- [ ] DPD5 používá pouze nejnovější verzi splátky.
- [ ] Bonusový denominator je celá TRAIN populace.

---

## 16. Shrnutí

Power BI report je postavený jako analytický model nad TRAIN částí Home Credit Default Risk dat. Hlavní logika je:

1. `factApplications` definuje základní populaci TRAIN žádostí.
2. `dimApplicant` drží demografické a aplikační atributy.
3. `factBureauCredits` přidává externí credit bureau expozici a typy kontraktů.
4. `factBureauMonthlyStatus` umožňuje časový trend no-DPD kontraktů relativně vůči žádosti.
5. `factPreviousApplications` umožňuje počítat počet předchozích žádostí.
6. `factInstallmentDPD5Summary` řeší bonusovou DPD5 logiku přes agregaci v Databricks.
7. `Measures Technical` obsahuje veškerou výpočetní logiku pro report.

Report tím pokrývá všechny čtyři zadané oblasti:

- distribuce žádostí podle demografie,
- objemová distribuce podle `AMT_CREDIT`,
- median DTI podle typu příjmu,
- vývoj no-DPD bureau kontraktů v relativním čase,
- bonusový DPD5 poměr.

