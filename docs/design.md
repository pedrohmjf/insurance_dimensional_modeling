# Insurance Dimensional Model: Design and Rationale

Nov 16, 2026 · @Pedro Henrique Marcondes João Fernandes

## 1. Overview

This document describes a Kimball-style dimensional model (star schema) for an auto insurance company: three fact tables, nine dimensions, and the reasoning and assumptions behind each design decision. The model is specified in DBML (dbdiagram.io); physical implementation is a later phase.

### Why a dimensional model

A transactional system (**OLTP**) is normalized so that each fact is stored once, which makes writes fast and consistent. Analytical questions against it require long chains of joins. **Dimensional modeling** optimizes for reading and understanding instead, accepting controlled redundancy.

|  | OLTP (operational) | OLAP (analytical) |
| --- | --- | --- |
| Goal | Record transactions | Answer business questions |
| Shape | Third normal form, many tables | Facts + dimensions |
| Typical query | One record by key | Aggregation over many rows |

### Method: Kimball's four steps

The data warehouse is built **one business process at a time**, integrated through shared dimensions. Every part of this model followed Kimball's four steps, in order:

1. **Select the business process**: an event the company measures.
2. **Declare the grain**: exactly what one row of the fact table represents.
3. **Identify the dimensions**: the event's context (who, what, when, where, how).
4. **Identify the facts**: the numeric measures, consistent with the grain.

The order matters: an undeclared grain leads to errors in every later step. Other common approaches are **Inmon** (a normalized enterprise warehouse first, data marts later) and **Data Vault** (an integration and history layer that usually feeds a Kimball model).

### Star schema, not snowflake

In a **star schema**, each dimension is a single denormalized table joined directly to the fact. In a **snowflake**, dimensions are split into sub-tables (product → category), which adds joins. The star schema was chosen: it is simpler to query and is the shape that tools such as Power BI and Microsoft Fabric expect.

"Denormalized" is relative: the star still has joins, but of a different shape. It removes join *chains*, so any attribute is at most one hop from the fact. A query only joins the dimensions it needs, usually two or three, and these joins are cheap because dimensions are small. The zero-join alternative is a single flat table (OBT), which is exactly what the source file is, with the problems described in section 2. A flat table can still be generated from the star later as a serving layer.

## 2. Source data

The source is the Kaggle **Auto Insurance Claims** dataset (`insurance_claims.csv`): about 1,000 rows and 40 columns covering policy, insured, incident, vehicle, compensation amounts and a fraud flag.

| Option | Strength | Why it was not the first choice |
| --- | --- | --- |
| Auto Insurance Claims (Kaggle) | One flat table with dates: decomposing it into facts and dimensions is the modeling exercise itself | Selected. Small and US-based |
| freMTPL2 (France) | Real, large data; policies and claims in separate files | No dates, so the time dimension would be poor |
| AUTOSEG (SUSEP, Brazil) | Official Brazilian data | Pre-aggregated, with no individual policy or claim. A candidate for a future additional process |

Only public data is used, to avoid any confidentiality concerns.

Source columns, in file order:

```
months_as_customer, age, policy_number, policy_bind_date, policy_state, policy_csl,
policy_deductable, policy_annual_premium, umbrella_limit, insured_zip, insured_sex,
insured_education_level, insured_occupation, insured_hobbies, insured_relationship,
capital-gains, capital-loss, incident_date, incident_type, collision_type,
incident_severity, authorities_contacted, incident_state, incident_city,
incident_location, incident_hour_of_the_day, number_of_vehicles_involved,
property_damage, bodily_injuries, witnesses, police_report_available,
total_claim_amount, injury_claim, property_claim, vehicle_claim,
auto_make, auto_model, auto_year, fraud_reported
```

This shape, with everything in one row, is an **OBT** (*One Big Table*): zero joins, but with mixed grains, no history and no way to share entities across processes. The model turns this OBT into a star schema.

## 3. Step 1: business processes

The source file mixes two business processes: **claims** (the incident) and **policy issuance**.

**Criterion:** a business process is an event that happens at a specific moment and produces numbers someone wants to add up. Each process has its own date and its own values.

| Process | Own date | Own values |
| --- | --- | --- |
| Claims | `incident_date` | compensation amounts (`injury_claim`, `property_claim`, `vehicle_claim`) |
| Policy issuance | `policy_bind_date` | premium (`policy_annual_premium`) |

**Why they are separated:** a single fact table would mix events that happen at different moments and are measured differently, producing wrong totals and ambiguous answers.

Claims were modeled first, as the richer process in this dataset.

## 4. Step 2: claims grain

The claims grain is **one row per incident × compensation type**: each incident becomes three rows, one each for bodily injury, property damage and vehicle.

**Grain:** a business statement of the form *"one row per…"* that defines exactly what each fact row represents. The rule is to prefer the **atomic grain**, the most detailed level the data allows. Aggregating later is easy; disaggregating is impossible.

**How it was derived:** `total_claim_amount` is the sum of `injury_claim`, `property_claim` and `vehicle_claim`. Storing each component separately is more atomic than storing the total.

**Why not one row per incident with three amount columns:** the chosen grain is more flexible. A fourth compensation category becomes new rows, with no schema change.

**Terminology:** these are *compensation types*, not *claim types*. In insurance, the claim type is `incident_type` (collision, theft, etc.).

**Assumption to verify during profiling:** `total_claim_amount = injury_claim + property_claim + vehicle_claim` on every row.

## 5. Testing the grain: header and line facts

Testing the grain showed that claims need **two fact tables**: one at incident grain (header) and one at incident × compensation type grain (line).

**The grain test:** every number in a fact must be true at the declared grain. With three rows per incident, two things do not fit:

- `total_claim_amount` is not stored; it is always derived with `SUM()` over the atomic grain.
- `number_of_vehicles_involved`, `witnesses` and `bodily_injuries` describe the whole incident. Repeated across three rows, they would be counted three times.

**Fan-out:** the row multiplication that occurs when a coarser-grain value is repeated at a finer grain. If `witnesses = 2` appears on three rows, `SUM(witnesses)` returns 6.

**Rule:** repetition is acceptable for **descriptive text in a dimension** ("Collision" repeated across rows). Repeating an **additive number** in a finer-grained fact is a correctness error. Descriptions may repeat; measures may not.

**Solution: the header/line pattern**, as in order × order line in retail. **A business process may have more than one fact table, as long as each has a distinct, well-declared grain.**

| Fact | Grain | Measures |
| --- | --- | --- |
| `fct_claim` (header) | 1 row per incident | vehicles involved, witnesses, bodily injuries, fraud flag |
| `fct_claim_compensation` (line) | 1 row per incident × compensation type | compensation amount |

**Numbers in dimensions:** a number may also live in a dimension when it is used to **filter or group** ("incidents with 3+ vehicles") rather than to sum. Practical test: *will it be calculated with, or used to slice the data?* If both, the measure stays in the fact and a band goes to the dimension.

## 6. Step 3: claims dimensions

`fct_claim` has eight dimensions.

**Dimension:** a wide, descriptive table that gives the event its context: who, what, when, where and how. Dimensions become the filters and axes of reports. These five questions served as a checklist.

| Dimension | Content | Design note |
| --- | --- | --- |
| `dim_date` | Calendar day | Kept separate from the hour: the policy process has a date but no hour, so a daily date dimension can be shared across processes, while a date-hour dimension cannot |
| `dim_hour` | Hour of the incident (0 to 23) and day period | Claims only |
| `dim_insured` | Insured profile | Time-variant attributes handled in section 12 |
| `dim_auto` | Make, model, year | Describes the car model, not the specific vehicle: no plate or VIN in source |
| `dim_policy` | Policy attributes | Conformed with the policy process (section 7) |
| `dim_geography` | City and state of the incident | Street address kept out (section 13) |
| `dim_incident_type` | Incident type, collision type, severity |  |
| `dim_report` | Authorities contacted, police report, property damage | Junk dimension |

**Junk dimension:** a dimension that groups low-cardinality flags and categories that belong to no particular entity. It keeps the fact table free of scattered columns.

**Grouping attributes:** the five questions are a checklist, not a strict taxonomy. Attributes belong together when they are analyzed together and their combinations produce few rows. Keeping `dim_incident_type` and `dim_report` separate, or merging them, is a design choice; they are kept separate here.

**Flags in the fact:** text does not go in a fact table; a 0/1 numeric flag does, because it can be summed and turned into rates (`SUM(fraud_flag) / COUNT(*)`). `fraud_reported` (Y/N) therefore becomes `fraud_flag` (0/1) in the fact. `authorities_contacted` has several categories and stays in a dimension.

**Unknown values:** `collision_type`, `property_damage` and `police_report_available` contain `?` in the source and become "Unknown" in the dimension. Every dimension also has an **unknown member** (row `-1`), so that an unmatched foreign key never drops fact rows from a join.

## 7. The policy as a conformed dimension

The policy is modeled as **`dim_policy`**, a dimension shared by both processes. It is the link between claims and premium.

**Conformed dimension:** a dimension with the same meaning, keys and values, used by more than one fact table. It is what allows different processes to be combined. In Kimball's terms, the set of conformed dimensions forms the **bus architecture**.

**Relationship:** each fact holds a **foreign key to the conformed dimension** (`sk_policy`), not the policy attributes themselves.

**Premium does not belong in `fct_claim`.** It is a measure of the policy process and lives in `fct_policy`. Combining premium and claims is the basis of the **loss ratio** (compensation ÷ premium), an insurer's main KPI, which is computed by **drill-across**.

**Drill-across:** the technique for combining two fact tables. Each fact is first aggregated separately by attributes of a conformed dimension; the results, now at the same grain, are then joined. Section 16 shows the full calculation.

**Mapping of the policy-related columns:**

| Column | Destination |
| --- | --- |
| `policy_number` | `dim_policy` (natural key) |
| `policy_state`, `policy_csl`, `policy_deductable`, `umbrella_limit` | `dim_policy` (attributes) |
| `policy_bind_date` | Foreign key to `dim_date` in `fct_policy` |
| `policy_annual_premium` | Measure in `fct_policy` |
| `months_as_customer`, `capital-gains`, `capital-loss` | Time-variant insured attributes (section 12) |

## 8. The line fact: inheritance and unpivot

`fct_claim_compensation` inherits the eight header dimensions and adds one new dimension, **`dim_compensation_type`**.

**The line-fact rule** has two parts:

1. **Inherit everything that is true at the header.** The three compensation rows of an incident share its date, insured, car, location and policy, so every `fct_claim` dimension also applies to the line.
2. **Add whatever distinguishes the lines from each other.** Every part of the finer grain must be represented in the fact.

**Why inherit rather than look up the header:** fact tables are never joined directly to each other, since that join reproduces fan-out. Each fact must be analyzable on its own, joining only to dimensions.

**Inheritance covers dimensions, not measures.** Witnesses, vehicles and injuries stay in `fct_claim` only. Age and tenure are also excluded from the line: an average age computed there would count each insured three times.

**The unpivot.** In the line fact, `injury_claim`, `property_claim` and `vehicle_claim` do not remain three columns. The column **names** become rows of `dim_compensation_type`, and the **values** become a single measure, `compensation_amount`.

Before, in source format (example incident):

| incident | injury\_claim | property\_claim | vehicle\_claim |
| --- | --- | --- | --- |
| A | 6,510 | 13,020 | 52,080 |

After, in `fct_claim_compensation`:

| incident | compensation type | compensation\_amount |
| --- | --- | --- |
| A | bodily injury | 6,510 |
| A | property damage | 13,020 |
| A | vehicle | 52,080 |

**Why compensation type is a dimension:** it answers a context question (*what was compensated?*) and is used to filter and group. The header has no such dimension because, at incident level, the type has no single value: one incident carries all three.

## 9. The claim identifier: `nr_claim`

Incidents are identified by **`nr_claim`**, generated at load from `policy_number + incident_date + incident_hour_of_the_day` and stored in both claim facts as a degenerate dimension.

**The problem:** the source has no claim ID column, yet the line fact needs to know which three rows belong to the same incident.

**Why not `policy_number` alone:** it is unique in this file only because each policy appears with a single incident. With another year of data, one policy would have several claims. The incident date is therefore part of the key.

**Why the hour:** two incidents on the same policy and day are rare but possible, and the hour strengthens uniqueness. It is still not an absolute guarantee, so it is a **declared assumption** (section 17).

**Why one generated column instead of three:** a single column is simpler to count (`COUNT(DISTINCT nr_claim)`) and to combine facts on. In a production system, the claim number from the claims system plays this role.

**Degenerate dimension:** a dimension key stored in the fact with no table of its own, typically the transaction identifier (order, invoice or claim number). It is "degenerate" because everything that would describe the claim already lives in other dimensions; a `dim_claim` would hold only its own key. It is used to group a transaction's rows, count transactions and trace back to the source.

**Combining header and line safely:** `nr_claim` does not license a row-by-row join between the facts, which would cause fan-out. The line is first aggregated to incident grain, then joined 1:1 to the header.

```sql
-- Wrong: join row by row, then aggregate (witnesses multiplied by 3)
SELECT s.nr_claim, SUM(s.witnesses), SUM(i.compensation_amount)
FROM fct_claim s
JOIN fct_claim_compensation i ON i.nr_claim = s.nr_claim
GROUP BY s.nr_claim;

-- Right: aggregate to the same grain, then join (1:1)
WITH comp AS (
  SELECT nr_claim, SUM(compensation_amount) AS total
  FROM fct_claim_compensation GROUP BY nr_claim
)
SELECT s.nr_claim, s.witnesses, comp.total
FROM fct_claim s JOIN comp ON comp.nr_claim = s.nr_claim;
```

**Rule:** never join facts of different grains before aggregating.

## 10. Policy grain: written vs earned premium

The `fct_policy` grain is **one row per policy issuance** (new business or renewal), a transactional fact. In this dataset, that is one row per policy.

**Grain is stated by event, not by measure.** "One row per annual premium" would describe the grain by what is measured. The grain names the event the row records; the premium is what is measured within it. A useful test: can a business analyst tell what one row represents without seeing the table?

**Two candidate grains were considered:**

|  | Option A | Option B |
| --- | --- | --- |
| Grain | one row per policy issuance | one row per policy per coverage period |
| Fact type | transactional | periodic snapshot |
| What a row records | an event: the policy was sold | a state: how much coverage belonged to the period |
| Measure | written premium | earned premium |
| Question answered | how much premium was written in 2014? | how much premium was in force in 2015? |

**Where they differ:** when a policy crosses a year boundary. A policy issued in October 2014, with a 1,200 premium and 12 months of coverage, has 1,200 written in 2014 (option A). Under option B, 300 is earned in 2014 and 900 in 2015.

**Business relevance:** a 2015 loss ratio should compare 2015 compensation with premium **earned** in 2015, not premium written in 2014.

**Decision: option A.** Option B is the better business model, but the source has no renewal history, no year-by-year premium and no cancellations; building it would require inventing data. **The grain must be supported by the available data.** Option B is documented as the model's evolution path: with coverage and cancellation history, a periodic snapshot of earned premium would be added.

**Fact table types, for reference:**

| Type | Grain | Example |
| --- | --- | --- |
| Transactional | 1 row per event | each policy issuance |
| Periodic snapshot | 1 row per entity per period | earned premium per policy per month |
| Accumulating snapshot | 1 row per process, updated at each milestone | order: purchase → approval → delivery |
| Factless | event with no measure | student attendance |

## 11. Policy dimensions

`fct_policy` has four dimensions: `dim_date` (issue date), `dim_policy`, `dim_insured` and `dim_auto`. Its measure is written premium.

**The event-moment test:** the grain test applied to dimensions. Each dimension must describe the row exactly at the declared grain; for a transactional fact, the question is *when the event happened, did this information already exist and have a single value?*

| Dimension | Included | Reason |
| --- | --- | --- |
| `dim_date` | Yes | Via `policy_bind_date`; conformed with claims |
| `dim_policy` | Yes | Conformed; links the two processes |
| `dim_insured` | Yes | The insured exists at issuance |
| `dim_auto` | Yes | The insured vehicle exists at issuance |
| `dim_hour` | No | It would pass the test, but the source has no issue time. A dimension without data is not included |
| `dim_incident_type`, `dim_geography`, `dim_report` | No | At issuance, no incident has occurred |

**Role-playing:** `dim_date` plays different roles: incident date in the claim facts and issue date in `fct_policy`. It is one table; the foreign key name (`sk_claim_date`, `sk_issue_date`) states the role.

## 12. Time-variant attributes

Insured attributes that change over time are handled in three ways: deriving a stable attribute, SCD Type 2, and bands. The issue arises because `dim_insured` is linked to two facts whose events may be years apart.

### Age (`age`)

**Problem:** a dimension row stores one value per attribute, but age changes. An insured who bought the policy at 30 and had a claim at 45 cannot have both ages on the same row. With `age = 45`, "written premium by age band" would be silently wrong.

**Solution:** derive the attribute that never changes and store it in the dimension:

```latex
\text{birth\_year} = \text{YEAR(incident\_date)} - \text{age}
```

Each fact then computes age at its own moment with the same rule: `age_in_claim = YEAR(incident_date) - birth_year` and `age_at_issue = YEAR(policy_bind_date) - birth_year`. In this dataset `age_in_claim` equals `age`, because source age was measured at the incident date; the rule matters at issuance. Precision is ±1 year (section 17).

**Alternative considered:** subtracting the date difference directly from age at load. It works for one moment only and requires converting days to years; the stable attribute serves any future process.

### Customer tenure (`months_as_customer`)

Same problem as age. The deciding question is *when was this number measured?* It arrives with the incident, so the assumption is the incident date:

```latex
\text{start\_date\_as\_client} = \text{incident\_date} - \text{months\_as\_customer}
```

**Why not `policy_bind_date`:** the insured may have been a customer before this policy. This yields a validation rule: `start_date_as_client` must be on or before `policy_bind_date`; otherwise the data or the assumption is wrong.

### Unstable attributes and SCD Type 2

Marital relationship, ZIP code, occupation and education level change over time.

**SCD (Slowly Changing Dimension):** the strategy for dimension attributes that change.

| Type | Behavior | When to use |
| --- | --- | --- |
| 0 | Never changes | date of birth |
| 1 | Overwrites, loses history | error corrections |
| 2 | Adds a new row with `valid_from`, `valid_to`, `flag_is_current` | when history matters |
| 3 | Keeps the previous value in a column | rare |

**Data limitation:** SCD2 creates a new version when it observes a change, and the source holds a single snapshot per insured. `dim_insured` is therefore **designed for SCD2**, with one version per insured in the current data, observed at incident date.

### Insured key

SCD2 needs to know that two rows are the same person, but the source has **no insured identifier**, not even a name.

| Option | Assessment |
| --- | --- |
| Combine attributes (ZIP, sex, birth year, occupation) | Fragile: different people collide, and the same person changes ZIP. An entity-resolution problem, out of scope |
| `policy_number` as a proxy | Selected. Assumption: one insured per policy |

**Limitation:** one person with two policies becomes two insured records. **In production**, the key would be the national ID (CPF), hashed or masked for LGPD compliance.

**Why `dim_insured` stays separate from `dim_policy`, even though they are 1:1 in this data:** they are distinct entities; in reality an insured holds many policies over time. Kept separate, only the `dim_insured` key changes when a real identifier becomes available.

### Capital gains and losses

**Definition:** *capital gains* are the profit from selling an asset (shares, property) above its cost; *capital losses* are the loss. They describe the insured's financial profile. The source does not document the period these values refer to, and the insured columns closely resemble the *Adult Census Income* dataset, which suggests synthetic assembly.

**Decision:** used for slicing and descriptive of the insured, they go to `dim_insured`. Unlike age, there is no rule to derive a stable value, so they follow the same SCD2 assumption as occupation. Each also gets a **band attribute** (no gain / up to X / above X), since slicing by a continuous value is not useful.

## 13. Geography and the incident address

`dim_geography` is at **city** grain (city + state), and the street address (`incident_location`) is stored in `fct_claim` as a degenerate attribute.

**Reuse as a design goal:** a geography dimension can serve several processes, which brings two Kimball patterns into play:

- **Conformed geography:** one list of states and cities, with the same names, used across processes.
- **Role-playing:** the same dimension in different roles (incident location, policy state, insured ZIP), as with `dim_date`.

**Grain obstacle:** incidents have state and city, but policies have state only (`policy_state`). With one row per city, a policy has no row to point to.

**Rollup (*shrunken*) dimension:** a version of a dimension at a higher grain with the same values: a `dim_state` with one row per state and the same state codes as `dim_geography`. Facts with city detail point to the full geography; facts with state only point to the shrunken version. Both remain conformed because they share the state values.

**Decision:** `policy_state` is kept as a **plain attribute of `dim_policy`**, for simplicity; the rollup dimension is documented as an alternative. If adopted, the state list must include every state from every source, not only incident states.

**Street address:** placing it inside `dim_geography` would change the dimension's grain to *address*. Since almost every incident has a different address, the dimension would grow to roughly one row per incident, the size of the fact, and could no longer serve processes that know only the city or state. The address therefore stays in the fact.

**Degenerate attribute:** a descriptive attribute stored in the fact with no table of its own, because it is nearly unique per row.

**Data profiling comes first:** values, nulls and uniqueness should be checked before modeling decisions. The nature of `incident_location` (a street address) is only visible by inspecting the data.

## 14. Physical design conventions

The model is written in **DBML** (dbdiagram.io). This section records the key and type conventions it follows.

### DBML conventions

- **Relationship:** `[ref: > dim_date.sk_date]` on the column. `>` means many-to-one: many claims on the same day, each day once in the dimension. Every fact-to-dimension foreign key is `>`.
- **Column note:** `[note: '...']` for what describes one column (a formula, an assumption).
- **Table note:** `Note: '...'` inside the block for what describes the whole table (grain, SCD type).
- **Grain as key:** `indexes { (nr_claim, sk_compensation_type) [pk] }` declares a composite key.
- **Groups:** `TableGroup` groups tables in the diagram. A table can belong to only one group, so conformed dimensions have a group of their own.
- **Foreign keys named by role:** `sk_claim_date` and `sk_issue_date` rather than `sk_date`.

### Keys

- **PK (primary key):** identifies each row of its own table. Every dimension has one.
- **FK (foreign key):** holds another table's PK value. A fact has one FK per dimension.
- **Natural key:** the source system's identifier, with business meaning (`policy_number`). Stored in the dimension as an attribute.
- **Surrogate key (SK):** a meaningless key generated by the warehouse (1, 2, 3…). **It is the dimension's PK.**

**At load, the natural key is the bridge:** a source row carries `policy_number = 521585`; the load finds that number in `dim_policy`, gets `sk_policy = 1`, and writes 1 to the fact.

**Why the natural key is not the PK:**

| Reason | Example |
| --- | --- |
| History (SCD2) | A second version of an insured would share the first version's PK, which is impossible. With SKs, each version has its own, and each fact points to the version valid at its event date |
| Source independence | If the source renumbers policies, only the dimension changes; the fact is untouched |
| Multiple sources | Two merged insurers may both have policy "1001" |
| Unknown member | Row `-1` does not exist in the source; only a warehouse-controlled key allows it |
| Performance | `int` is smaller and faster to join |

### Data types

| Key or column | Type | Reason |
| --- | --- | --- |
| SK | `int` | Compact and fast; no business meaning |
| Date SK | `int` YYYYMMDD | Accepted exception for dates: readable and needs no lookup at load |
| Natural key (`policy_number`) | `varchar`, even if numeric | Identifiers are not calculated with; leading zeros would be lost in `int` |
| `policy_csl` | `varchar` | Arrives as text (`250/500`) |
| Money | `decimal` | `float` has rounding errors |
| Flag in a fact | `int` 0/1 | Can be summed and turned into rates |
| Flag in a dimension | `varchar` (Yes / No / Unknown) | A filter label that must represent `?` |

An implementation alternative is **hash SKs** (MD5 of the natural key, `varchar(32)`), common in dbt and Data Vault because they can be computed without a dimension lookup.

**FK and PK types must match** across every relationship.

## 15. Final model

The model has **3 facts and 9 dimensions**: two claim facts (header and line), one policy fact, and four conformed dimensions that link the two processes.

&#91;embedded content: final model · 3 facts, 9 dimensions\]

The conformed dimensions, at the bottom, are where claims and policy meet; the dimensions on the left belong to claims only, and `dim_compensation_type` links to the line fact alone.

| Table | Type | Grain or role |
| --- | --- | --- |
| `fct_claim` | Transactional fact (header) | 1 row per incident |
| `fct_claim_compensation` | Transactional fact (line) | 1 row per incident × compensation type |
| `fct_policy` | Transactional fact | 1 row per policy issuance |
| `dim_date` | Conformed, role-playing | 1 row per day; incident date and issue date |
| `dim_insured` | Conformed, SCD2 | 1 row per insured version |
| `dim_auto` | Conformed | 1 row per car model |
| `dim_policy` | Conformed | 1 row per policy |
| `dim_hour` | Claims only | 1 row per hour (0 to 23) |
| `dim_geography` | Claims only | 1 row per city |
| `dim_incident_type` | Claims only | combination of type, collision and severity |
| `dim_report` | Junk dimension | combination of report flags |
| `dim_compensation_type` | Line fact only | 3 rows, one per compensation type |

### Full DBML

Paste into dbdiagram.io:

```
Project insurance_dw {
  database_type: 'Generic'
  Note: '''
  Dimensional model (Kimball) for an auto insurance company.
  Source: Kaggle "Auto Insurance Claims" (insurance_claims.csv), a single flat table.
  Business processes: (1) claims, modeled as header + line facts; (2) policy issuance.
  Key assumptions:
  - No claim ID in source: nr_claim = policy_number + incident_date + incident_hour_of_the_day.
  - No insured ID in source: policy_number is used as a proxy (1 insured per policy).
  - Insured attributes (age, tenure, occupation...) were observed at incident date.
  '''
}

// CONFORMED DIMENSIONS

Table dim_date {
  sk_date int [pk, note: 'Smart key YYYYMMDD; -1 = unknown']
  date date
  year int
  quarter int
  month int
  day int
  week_day varchar
  flag_weekend int
  Note: 'Generated, not extracted. Role-playing: incident date (claims) and issue date (policy).'
}

Table dim_insured {
  sk_insured int [pk]
  policy_number varchar [note: 'Natural key (proxy for insured ID: none in source)']
  insured_zip varchar
  insured_sex varchar
  insured_education_level varchar
  insured_occupation varchar
  insured_hobbies varchar
  insured_relationship varchar
  capital_gains decimal
  capital_loss decimal
  capital_gains_range varchar
  capital_loss_range varchar
  birth_year int [note: 'Derived: YEAR(incident_date) - age (±1 year)']
  start_date_as_client date [note: 'Derived: incident_date - months_as_customer']
  valid_from date
  valid_to date [note: '9999-12-31 for the current version']
  flag_is_current int
  Note: 'SCD Type 2. Current dataset holds one version per insured, observed at incident date.'
}

Table dim_auto {
  sk_auto int [pk]
  auto_make varchar
  auto_model varchar
  auto_year int
  Note: 'Grain: 1 row per car model (make + model + year). Describes the model, not the specific vehicle: no plate or VIN in source.'
}

Table dim_policy {
  sk_policy int [pk]
  policy_number varchar [note: 'Natural key']
  policy_state varchar [note: 'Kept as attribute (state-level only in source)']
  policy_csl varchar [note: 'Combined single limit, e.g. 250/500']
  policy_deductable decimal
  umbrella_limit decimal
}

// CLAIMS PROCESS

Table dim_hour {
  sk_hour int [pk, note: '0-23; -1 = unknown']
  hour int
  day_period varchar
}

Table dim_geography {
  sk_geography int [pk]
  city varchar
  state varchar
  Note: 'Grain: city. Street address kept out on purpose (degenerate attribute in fct_claim) so this dimension stays reusable.'
}

Table dim_incident_type {
  sk_incident_type int [pk]
  incident_type varchar
  collision_type varchar
  incident_severity varchar
}

Table dim_report {
  sk_report int [pk]
  authorities_contacted varchar
  police_report_available varchar [note: 'Yes / No / Unknown (? in source)']
  property_damage varchar [note: 'Yes / No / Unknown (? in source)']
  Note: 'Junk dimension: low-cardinality flags and categories of the incident report.'
}

Table dim_compensation_type {
  sk_compensation_type int [pk]
  compensation_type varchar [note: 'Bodily injury / Property damage / Vehicle']
  source_column varchar [note: 'injury_claim / property_claim / vehicle_claim']
  Note: '3 rows. Created by unpivoting the three compensation columns of the source.'
}

Table fct_claim {
  nr_claim varchar [note: 'Degenerate dimension: policy_number + incident_date + incident_hour_of_the_day']
  sk_claim_date int [ref: > dim_date.sk_date]
  sk_claim_hour int [ref: > dim_hour.sk_hour]
  sk_insured int [ref: > dim_insured.sk_insured]
  sk_auto int [ref: > dim_auto.sk_auto]
  sk_policy int [ref: > dim_policy.sk_policy]
  sk_geography int [ref: > dim_geography.sk_geography]
  sk_incident_type int [ref: > dim_incident_type.sk_incident_type]
  sk_report int [ref: > dim_report.sk_report]
  incident_location varchar [note: 'Degenerate attribute (street address)']
  number_of_vehicles_involved int
  witnesses int
  bodily_injuries int
  fraud_flag int [note: '0/1, comes as Y/N in source']
  age_in_claim int [note: 'YEAR(incident_date) - dim_insured.birth_year (±1 year)']
  time_as_customer_months int [note: 'DATEDIFF(month, dim_insured.start_date_as_client, incident_date)']
  indexes {
    (nr_claim) [pk]
  }
  Note: 'Header fact. Grain: 1 row per incident. total_claim_amount is not stored: it is SUM(fct_claim_compensation.compensation_amount).'
}

Table fct_claim_compensation {
  nr_claim varchar [note: 'Degenerate dimension, same value as fct_claim']
  sk_compensation_type int [ref: > dim_compensation_type.sk_compensation_type]
  sk_claim_date int [ref: > dim_date.sk_date]
  sk_claim_hour int [ref: > dim_hour.sk_hour]
  sk_insured int [ref: > dim_insured.sk_insured]
  sk_auto int [ref: > dim_auto.sk_auto]
  sk_policy int [ref: > dim_policy.sk_policy]
  sk_geography int [ref: > dim_geography.sk_geography]
  sk_incident_type int [ref: > dim_incident_type.sk_incident_type]
  sk_report int [ref: > dim_report.sk_report]
  compensation_amount decimal
  indexes {
    (nr_claim, sk_compensation_type) [pk]
  }
  Note: '''
  Line fact. Grain: 1 row per incident x compensation type.
  Inherits all header dimensions. Header measures (witnesses, vehicles...) are NOT inherited:
  they would be repeated 3x and break any SUM (fan-out).
  '''
}

// POLICY PROCESS

Table fct_policy {
  sk_issue_date int [ref: > dim_date.sk_date, note: 'policy_bind_date']
  sk_policy int [ref: > dim_policy.sk_policy]
  sk_insured int [ref: > dim_insured.sk_insured, note: 'Insured version valid at issue date (SCD2)']
  sk_auto int [ref: > dim_auto.sk_auto]
  annual_premium_written decimal [note: 'Written premium (policy_annual_premium)']
  age_at_issue int [note: 'YEAR(policy_bind_date) - dim_insured.birth_year (±1 year)']
  time_as_customer_months_at_issue int [note: 'DATEDIFF(month, dim_insured.start_date_as_client, policy_bind_date)']
  indexes {
    (sk_policy, sk_issue_date) [pk]
  }
  Note: '''
  Transactional fact. Grain: 1 row per policy issuance (new business or renewal).
  Source has no renewal history, so today it holds 1 row per policy.
  Future evolution: periodic snapshot of earned premium (policy x month).
  '''
}

// GROUPS (a table can belong to only one group)

TableGroup conformed_dimensions {
  dim_date
  dim_insured
  dim_auto
  dim_policy
}

TableGroup claims_process {
  fct_claim
  fct_claim_compensation
  dim_hour
  dim_geography
  dim_incident_type
  dim_report
  dim_compensation_type
}

TableGroup policy_process {
  fct_policy
}
```

## 16. Example queries

Two query patterns cover the model: with **one fact**, join, group and sum; with **two facts**, aggregate each by a conformed dimension, then join the results.

### Loss ratio by `policy_state` (two facts)

Loss ratio is a portfolio metric: per insured it is almost always 0% or an extreme value, so it is computed over groups of policies.

```latex
\text{loss ratio} = \frac{\text{total compensation}}{\text{total premium}}
```

| Component | Location |
| --- | --- |
| Numerator | `fct_claim_compensation.compensation_amount` |
| Denominator | `fct_policy.annual_premium_written` |
| `policy_state` | `dim_policy`, conformed across both facts |

1. Aggregate compensation by state through `dim_policy`: one row per state.
2. Aggregate premium by state through `dim_policy`: one row per state.
3. Join both results on `policy_state` and divide. The join is 1:1, with no fan-out.

```sql
WITH claims AS (
  SELECT p.policy_state, SUM(c.compensation_amount) AS total_compensation
  FROM fct_claim_compensation c JOIN dim_policy p ON p.sk_policy = c.sk_policy
  GROUP BY p.policy_state
),
premium AS (
  SELECT p.policy_state, SUM(f.annual_premium_written) AS total_premium
  FROM fct_policy f JOIN dim_policy p ON p.sk_policy = f.sk_policy
  GROUP BY p.policy_state
)
SELECT c.policy_state, c.total_compensation / pr.total_premium AS loss_ratio
FROM claims c JOIN premium pr ON pr.policy_state = c.policy_state;
```

### Why not group by `sk_geography`

1. **Different state, different meaning.** `dim_geography` holds where the incident happened; `policy_state` is where the policy was written. A policy from one state can have an incident in another.
2. **`fct_policy` is not linked to `dim_geography`.** Combining two facts requires a dimension shared by both, and incident geography fails the issuance test.
3. **Group by attributes, never by SKs.** Grouping by `sk_geography` yields one group per city (the dimension's grain), labeled with meaningless numbers. SKs are for joins; attributes go in `GROUP BY`.

### Compensation by incident state (one fact)

A single fact needs no drill-across: join `fct_claim_compensation` to `dim_geography` on the SK, group by state and sum.

```sql
SELECT g.state, SUM(c.compensation_amount) AS total_compensation
FROM fct_claim_compensation c
JOIN dim_geography g ON g.sk_geography = c.sk_geography
GROUP BY g.state;
```

## 17. Assumptions and limitations

| Topic | Assumption or limitation | Verification or evolution |
| --- | --- | --- |
| Claim ID | `nr_claim` = policy + date + hour; uniqueness is not guaranteed | Claim number from a claims system |
| Insured ID | `policy_number` as proxy: one insured per policy | National ID (CPF), hashed for LGPD |
| Age | Estimated birth year, ±1 year precision | Actual date of birth |
| Customer tenure | `months_as_customer` measured at incident date | Check that start date precedes issue date |
| Insured attributes | Single snapshot at incident date; SCD2 by design only | Customer history |
| Capital gains and losses | Period undocumented; signs of synthetic data | Documented source |
| Premium | Written premium (option A); no renewals or cancellations | Periodic snapshot of earned premium (option B) |
| Vehicle | `dim_auto` describes the model, not the specific car | Plate or VIN |
| Total compensation | `total_claim_amount` = sum of the three components | Verify during profiling |
| Loss ratio | Every policy in the dataset has a claim, so the ratio is far above a real portfolio | Data including policies without claims |
| States | Policy and incident states may not overlap | Verify during profiling |

## 18. Glossary

| Term | Definition | Where it appears |
| --- | --- | --- |
| Accumulating snapshot | Fact with one row per process, updated at each milestone | Reference (section 10) |
| Additive measure | Can be summed across any dimension (compensation amount) | Section 5 |
| Atomic grain | The most detailed level the data allows | Section 4 |
| Bus architecture | The set of conformed dimensions that integrates processes in a Kimball warehouse | Section 7 |
| Conformed dimension | Dimension with the same meaning and values, used by more than one fact | `dim_policy`, `dim_date` (section 7) |
| Data profiling | Exploring data to learn values, nulls and uniqueness before modeling | Section 13 |
| Degenerate attribute | Descriptive attribute kept in the fact with no table, as it is nearly unique per row | `incident_location` (section 13) |
| Degenerate dimension | Dimension key kept in the fact with no table; usually the transaction ID | `nr_claim` (section 9) |
| Dimension | Descriptive table giving the event context: who, what, when, where, how | Section 6 |
| Drill-across | Combining two facts by aggregating each by a conformed dimension, then joining the results | Loss ratio (sections 7 and 16) |
| Earned premium | The share of premium belonging to each coverage period | Option B (section 10) |
| Fact | Table with a process's numeric measures and FKs to dimensions | Section 1 |
| Fan-out | Row multiplication when joining tables of different grains, inflating sums | Section 5 |
| FK (foreign key) | Column holding another table's PK | Section 14 |
| Grain | Exactly what one fact row represents, in business terms | Sections 4 and 10 |
| Header/line pattern | Two facts for one process: one at transaction grain, one at item grain | `fct_claim` and `fct_claim_compensation` (section 5) |
| Junk dimension | Dimension grouping low-cardinality flags and categories | `dim_report` (section 6) |
| Loss ratio | Compensation ÷ premium; an insurer's main KPI | Sections 7 and 16 |
| Natural key | Source-system identifier with business meaning | `policy_number` (section 14) |
| Non-additive measure | Cannot be summed (a ratio); store numerator and denominator | Loss ratio (section 16) |
| OBT (One Big Table) | Single flat table with no joins and mixed grains | Source file (section 2) |
| OLTP / OLAP | Transactional vs analytical systems | Section 1 |
| Periodic snapshot | Fact with one row per entity per period | Option B (section 10) |
| PK (primary key) | Column that identifies each row of its own table | Section 14 |
| Rollup (shrunken) dimension | A dimension at a higher grain sharing the same values | `dim_state` alternative (section 13) |
| Role-playing | One dimension used in several roles, through distinctly named FKs | `dim_date` (sections 11 and 13) |
| SCD (Slowly Changing Dimension) | Strategy for changing dimension attributes; Type 2 keeps history in new rows | `dim_insured` (section 12) |
| Semi-additive measure | Summable across some dimensions but not time (balance, stock) | Reference |
| Snowflake schema | Dimensions normalized into sub-tables | Section 1 |
| Star schema | Each dimension is a single table joined directly to the fact | Section 1 |
| Surrogate key (SK) | Meaningless key generated by the warehouse; the dimension's PK | Section 14 |
| Unknown member | Row `-1` in each dimension for unmatched FKs | Section 6 |
| Unpivot | Turning columns into rows: names become dimension values, values become one measure | `dim_compensation_type` (section 8) |
| Written premium | Premium recorded in full on the date the policy is sold | Option A (section 10) |

## 19. Next steps

- [ ] Profile the source: policy vs incident states, `?` values, uniqueness of `policy_number` and `nr_claim`, and whether the total equals the sum of the three components.
- [ ] Validate that `start_date_as_client` is on or before `policy_bind_date` for every row.
- [ ] Add the diagram, grain statements, decision log and section 17 to the project README.
- [ ] Implement the model in Microsoft Fabric.
