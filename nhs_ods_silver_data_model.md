# NHS ODS Silver Data Model – `nhs_ods_silver`

## 1. Purpose & high-level design

The `nhs_ods_silver` database holds **cleaned (Silver-layer) reference data from the NHS Organisation Data Service (ODS)**.

ODS is the master list of NHS organisation and site codes used across health and social care. This Silver layer standardises the raw ODS CSVs into a set of tables that are easy to join from analytical datasets (GP workforce, GP appointments, Fingertips, etc.) and that can also be used as a knowledge base for LangChain / NLP.

### Tables

Current tables:

- `gp_practice_dim` – curated dimension for GP practices, enriched with geography and commissioning info.
- `orgs` – master dimension of all ODS organisations (trusts, ICBs, practices, PCNs, sites, etc.).
- `org_dates` – legal/operational date ranges by organisation and date type.
- `org_roles` – roles held by organisations (e.g. GP practice, ICB, trust, prescribing cost centre).
- `org_rels` – organisation-to-organisation relationships (e.g. commissioning, partnership, operational relationships).
- `org_succs` – succession relationships (predecessor / successor links when organisations are reconfigured).

### Core keys

- `orgid` is the **main key** that identifies an ODS organisation or site in almost all tables.
- `roleid` identifies an ODS role type (e.g. GP practice, ICB, trust).
- Geo fields like `areacode_icb`, `geographic local authority code`, `geographic government office region code` connect organisations to health and local-government geographies.

For **NLP / LangChain**, describe this database as:

> A Silver-layer ODS reference model centred around the `orgs` and `gp_practice_dim` tables for organisation details, with supporting tables that describe roles, operational/legal dates, relationships, and predecessors/successors. Most analytics will join fact tables to ODS using `orgid` or a GP practice code and then use the geographic and commissioning fields to roll up to ICB, local authority, region, or country.

---

## 2. `gp_practice_dim` – GP Practice Dimension

### Business description

Curated dimension table containing **one row per GP practice**, enriched with address, commissioning relationships, roles and geography codes.

This is the **preferred table** for joining primary care datasets (GP appointments, GP workforce, patients registered at a practice, GP Patient Survey) when you want practice-level information plus ICB / LA / region context.

### Technical info

- **Table name:** `nhs_ods_silver.gp_practice_dim`
- **Grain:** 1 row = 1 GP practice (ODS prescribing cost centre / GP practice code).
- **Primary key:** `orgid` (ODS practice code such as `Y04386`, `G84022`).

### Column groups and meanings

Below we group the many columns into logical blocks rather than list every single column separately.

#### 2.1 Identity & status

- `orgid` – ODS code for the GP practice.
- `name` – GP practice name.
- `status` – textual status (e.g. `Active`, `Inactive`).
- `isactive` – boolean flag (true/false) indicating if the practice is currently active.
- `orgrecordclass` – record class code (e.g. `RC1`).
- `orgrecordclasslabel` – descriptive label for record class (e.g. `HSCOrg`).
- `lastchangedate` – date that this record was last updated in ODS.

These fields are used for basic filtering such as “only active practices”, or seeing if an organisation has changed recently.

#### 2.2 Address & contact

- `addrline1`–`addrline4` – address lines.
- `towncity` – town or city.
- `county` – county.
- `country` – country (e.g. `ENGLAND`).
- `postcode` – ODS compact postcode (no space).
- `postcodespaced` – standard UK postcode with a space.
- `addressfull` – full concatenated address.
- `contact telephone number` – practice phone number (where available).

These allow display of practice addresses and could be used for geocoding or distance calculations together with latitude/longitude.

#### 2.3 Commissioning and relationship “flattened” columns

ODS expresses many relationships between organisations (who commissions whom, who operates what, etc.). In `gp_practice_dim`, some of these have been flattened into descriptive sets of columns, each starting with a pattern such as **`is commissioned by - ...`**.

For each relationship type, there is a set of columns that describe the target organisation, its primary role, and the legal date range:

- `is commissioned by - code`
- `is commissioned by - primary role id`
- `is commissioned by - legal start date`
- `is commissioned by - legal end date`
- `is commissioned by - name`
- `is commissioned by - primary role name`

Similarly for other relationship prefixes:

- `is partner to - ...`
- `is a sub-division of - ...`
- `is directed by - ...`
- `is operated by - ...`
- `is nominated payee for - ...`
- `is constituent of - ...`

These columns tell you, for each GP practice, **which ICB, PCN or other body it is related to and in what capacity**, and from/to which dates those relationships apply.

For example (from your sample):

- Practice `Y04386` has `is commissioned by - code = 72Q` and name `NHS SOUTH EAST LONDON ICB - 72Q`.
- Practice `Y04386` is also partnered with `BROMLEY HEALTHCARE` with role `RO172` from 2013-08-29.

These values are important when rolling up practice-level metrics to ICB, PCN or other organisational groupings.

#### 2.4 Geography & health system fields

Core geospatial and system mapping fields:

- `national grouping code` – internal region code (e.g. `720`).
- `latitude`, `longitude` – decimal coordinates for the practice address.
- `areacode_lad` – Local Authority District code.
- `areacode_lsoa` – Lower Layer Super Output Area code.
- `areacode_msoa` – Middle Layer Super Output Area code.
- `areacode_icb` – Integrated Care Board code (e.g. `E54000030`).
- `geographic government office region code` – GOR code (e.g. `E12000007`).
- `high level health geography code` – high-level NHS geography code.
- `geographic local authority code` – local authority code as used in geography models.
- `geographic local authority name` – local authority name (e.g. `Bromley`).
- `high level health geography name` – e.g. `NHS South East London Integrated Care Board`.
- `geographic government office region name` – e.g. `London`.
- `national grouping name` – group name associated with `national grouping code`.
- `geographic primary care organisation code` – historical primary care organisation code.
- `geographic primary care organisation name` – name of that organisation.

These fields expose how each practice sits within **local authority, region, ICB and NHS geography hierarchies**. They are key for joining to LA/region-level datasets or for aggregating practice metrics.

#### 2.5 Role information

- `primary role id` – main role for the organisation (often `RO177` for prescribing cost centre / practice-related roles, or `RO76` for GP practice).
- `primary role name` – descriptive label, e.g. `PRESCRIBING COST CENTRE` or `GP PRACTICE`.
- `non primary role id(s)` – pipe-separated list of additional roles (e.g. `RO247|RO72`).
- `non primary role name(s)` – corresponding names for non-primary roles.
- `non primary role legal start date`, `non primary role legal end date` – date range that non-primary roles apply.
- `non primary role operational start date`, `non primary role operational end date` – operational date range for those roles.

#### 2.6 Legal / operational dates and succession

- `legal start date`, `legal end date` – legal existence dates of the organisation.
- `operational start date`, `operational end date` – dates when the organisation was operational.
- `predecessor(s)` – codes of predecessor organisations (if any).
- `successor(s)` – codes of successor organisations (if any).
- `has forward succession?` – boolean flag indicating whether ODS records a forward succession chain.
- `ctry` – country grouping (e.g. `720` series for England, `S92` for Scotland pseudo-code).
- `dt` – snapshot date in the Silver pipeline (date the record was processed).

### Relationships & usage

- Joins to clinical/operational fact tables via `orgid` (GP practice code).
- Can be joined to `org_roles`, `org_dates`, `org_rels` and `org_succs` for more detailed history and role information using `orgid`.
- Acts as a **convenience view** over the more generic `orgs` table where only GP practices are needed.

### NLP / LangChain hints

For natural language questions:

- “Practice”, “GP practice”, “surgery”, “prescribing cost centre” → refer to `gp_practice_dim`.
- “Which ICB is this practice in?” → use `areacode_icb` and `high level health geography name`.
- “Which local authority / region is this practice in?” → use `geographic local authority name` and `geographic government office region name`.
- “Is this practice active?” → check `isactive = true` and `status = 'Active'` and `operational end date` is null.
- “Show all practices commissioned by [ICB]” → filter on `is commissioned by - code` or `is commissioned by - name`.
- For time-bounded questions (“who commissioned this practice in 2020?”), use legal/operational start and end dates plus the `is commissioned by` legal date range.

---

## 3. `orgs` – All Organisations Dimension

### Business description

`orgs` is the **master list of all organisations and sites in ODS**, not just GP practices. It uses the same column structure as `gp_practice_dim`, but includes:

- Trusts, ICBs, PCNs, commissioning bodies.
- Hospitals and community sites.
- Scottish/Welsh pseudo-codes, independent providers, etc.

This table is used when you need to work with **non-GP organisations**, or when you want the most general form of organisation metadata.

### Technical info

- **Table name:** `nhs_ods_silver.orgs`
- **Grain:** 1 row = 1 organisation or site record as defined by ODS.
- **Primary key:** `orgid` (e.g. `S85032`, `R1C81`).

### Column groups

Column pattern is identical to `gp_practice_dim`:

- Identity & status (`orgid`, `name`, `status`, `isactive`, `orgrecordclass`, `orgrecordclasslabel`, `lastchangedate`).
- Address & contact fields (`addrline1`–`addrline4`, `towncity`, `county`, `country`, `postcode`, `postcodespaced`, `addressfull`, `contact telephone number`).
- Relationship columns (`is commissioned by - ...`, `is partner to - ...`, `is a sub-division of - ...`, `is directed by - ...`, `is operated by - ...`, `is nominated payee for - ...`, `is constituent of - ...`).
- Geography & health system mapping (LAD, LSOA, MSOA, ICB, GOR, LA code/name, region name, primary care organisation codes, latitude/longitude).
- Role information (`primary role id`, `primary role name`, `non primary role id(s)`, `non primary role name(s)`, and their date ranges).
- Legal/operational dates and succession (`legal start date`, `legal end date`, `operational start date`, `operational end date`, `predecessor(s)`, `successor(s)`, `has forward succession?`, `ctry`, `dt`).

### NLP / LangChain hints

- When a question is about **trusts**, **ICBs**, **PCNs**, or other organisations, prefer `orgs` rather than `gp_practice_dim`.
- “Organisation”, “trust”, “ICB”, “PCN”, “site” may all map to records in `orgs`.
- To filter for specific role types (e.g. only ICBs), use `primary role id` or `primary role name` in this table or join to `org_roles`.

---

## 4. `org_dates` – Organisation Date Ranges

### Business description

`org_dates` stores **date ranges associated with organisations** for different date types (e.g. Operational, Legal, Prescribing). It provides a more normalised view than the flattened columns on `orgs`/`gp_practice_dim` and can be used for detailed temporal analysis.

### Technical info

- **Table name:** `nhs_ods_silver.org_dates`
- **Grain:** 1 row = 1 (`orgid`, `datetype`) date range, with start and end dates.
- **Columns:**

  - `orgid` – organisation ID (ODS code).
  - `datetype` – the type of date, e.g. `Operational`, `Legal`, `Prescribing` (values depend on ODS).
  - `start` – start date for this date type.
  - `end` – end date for this date type (nullable if ongoing).
  - `dt` – Silver-layer snapshot date.

### Usage & relationships

- Join with `orgs` on `orgid` to see when an organisation was operational or legally in existence.
- Useful for historical analysis, e.g. “which organisations were operational on 2015-01-01?”.

### NLP / LangChain hints

- Questions like “When did this trust become operational?” or “Which organisations were active in 1990?” should consider `org_dates` with `datetype = 'Operational'`.
- “Legal start/end date” questions can either use the flattened columns on `orgs` or this more normalised table.

---

## 5. `org_roles` – Organisation Roles

### Business description

`org_roles` describes **which roles each organisation holds and when**. Roles describe the type of organisation, such as ICB, GP practice, trust, prescribing cost centre, etc.

### Technical info

- **Table name:** `nhs_ods_silver.org_roles`
- **Grain:** 1 row = 1 (`orgid`, `roleid`) role assignment for a specific date range.
- **Columns:**

  - `orgid` – ODS organisation ID.
  - `roleid` – role code (e.g. `RO198`, `RO177`, `RO76`).
  - `primaryrole` – boolean flag; true if this is the organisation’s primary role.
  - `rolestatus` – role status (e.g. `Active`, `Inactive`).
  - `rolename` – descriptive role name (e.g. `GP PRACTICE`, `INTEGRATED CARE BOARD`, `HSCSite`). May be blank in some extracts.
  - `roledatetype` – date type for the role (typically `Operational`).
  - `rolestart` – role start date.
  - `roleend` – role end date (if any).
  - `dt` – Silver snapshot date.

### Usage & relationships

- Join with `orgs` on `orgid` to see all roles an organisation has held over time.
- Filter for specific role types:
  - ICBs: role IDs like `RO198` (example; check actual mapping).
  - GP Practices: usually `RO76` or similar.
- For “point-in-time” role questions, filter with `rolestart <= date < roleend OR roleend IS NULL`.

### NLP / LangChain hints

- Questions such as “What type of organisation is this?” or “Is this organisation an ICB or a trust?” can map to `org_roles` (and/or primary role on `orgs`).
- Questions about when an organisation assumed or lost a role should use `rolestart` and `roleend`.

---

## 6. `org_rels` – Organisation Relationships

### Business description

`org_rels` stores **relationships between organisations**, expressed as edges between a `orgid` and a `targetorgid`. Examples include commissioning relationships, partnership relationships, and other structural relationships defined by ODS.

### Technical info

- **Table name:** `nhs_ods_silver.org_rels`
- **Grain:** 1 row = 1 relationship between two organisations for a given date range.
- **Columns:**

  - `orgid` – source organisation ID.
  - `relid` – relationship type code (e.g. `RE5`, `RE6`).
  - `relstatus` – status of the relationship (e.g. `Active`, `Inactive`).
  - `targetorgid` – target organisation ID.
  - `targetprimaryroleid` – primary role of the target organisation.
  - `reldatetype` – date type for the relationship (e.g. `Operational`).
  - `relstart` – start date for the relationship.
  - `relend` – end date for the relationship (null if ongoing).
  - `dt` – Silver snapshot date.

### Usage & relationships

- Join with `orgs` on `orgid` and `targetorgid` to see names and details of both sides of the relationship.
- Useful for:
  - Understanding which organisations commission or partner with which others.
  - Building graphs/network views of NHS structures.

### NLP / LangChain hints

- Questions like “Which organisations does this ICB commission?” or “Which trust is this site part of?” can map to `org_rels`.
- The model may need an internal mapping of `relid` codes to English labels (e.g. “commissioned by”, “partner to”) – this can be added as a small static lookup in your app or prompt.

---

## 7. `org_succs` – Predecessor/Successor Links

### Business description

`org_succs` describes **succession relationships between organisations** – who replaced whom in legal or operational terms when organisations are merged, split or reconfigured.

### Technical info

- **Table name:** `nhs_ods_silver.org_succs`
- **Grain:** 1 row = 1 succession link from an organisation to a target organisation.
- **Columns:**

  - `orgid` – source organisation ID (can be predecessor or successor depending on `succtype`).
  - `succtype` – type of succession, such as `Predecessor` or `Successor`.
  - `targetorgid` – target organisation ID (the other side of the succession).
  - `targetprimaryroleid` – primary role ID of the target organisation.
  - `succdatetype` – date type for the succession (e.g. `Legal`).
  - `succstart` – start date of the succession.
  - `succend` – end date (rare; often null).
  - `dt` – Silver snapshot date.

### Usage & relationships

- To trace history when organisations change:
  - “What is the successor of organisation RTE15?” → look up rows with `orgid = 'RTE15'` and `succtype = 'Predecessor'` and read `targetorgid`.
  - Then join `targetorgid` to `orgs` to get the successor name.
- Can be combined with `org_dates` to reason about when the transition occurred.

### NLP / LangChain hints

- Questions like “Which organisation replaced this trust?”, “What did this organisation become?”, or “What were the predecessors of this ICB?” should use `org_succs`.
- The `has forward succession?` flag on `orgs`/`gp_practice_dim` can serve as a quick hint that an organisation has successor records in `org_succs`.

---

## 8. Suggested context text for LLMs (LangChain)

You can embed the following summary as a context document for agents that need to generate SQL or reason about NHS organisations:

> The `nhs_ods_silver` database is the Silver-layer reference for NHS Organisation Data Service (ODS). The key tables are:
>
> - `orgs`: one row per ODS organisation or site (trusts, ICBs, GP practices, PCNs, sites, etc.) with identity, address, geography, roles, and flattened relationship fields.
> - `gp_practice_dim`: one row per GP practice, derived from ODS and enriched with commissioning and geographic fields. This is the preferred table for GP practice analytics.
> - `org_dates`: date ranges by organisation and date type (operational, legal, etc.).
> - `org_roles`: roles held by each organisation (e.g. GP practice, ICB, trust) with start/end dates.
> - `org_rels`: relationships between organisations (who commissions/partners/directs whom) with start/end dates.
> - `org_succs`: predecessor/successor links between organisations for organisational change.
>
> Most analytical datasets will join to ODS using `orgid` (organisation code) or, for GP practice datasets, via `gp_practice_dim.orgid`. To roll metrics up to wider geographies:
>
> - Use `areacode_icb` and `high level health geography name` for ICBs.
> - Use `geographic local authority code` / `geographic local authority name` for local authorities.
> - Use `geographic government office region code` / name for regions.
>
> When answering questions about where a practice sits in the system (ICB, LA, region), use `gp_practice_dim`. Questions about non-GP organisations (trusts, ICBs, PCNs, sites) should use `orgs` and can be further broken down by `primary role id` / `primary role name` or by looking up roles in `org_roles`. Historical questions about when organisations or roles were active use `org_dates`, `org_roles`, and the date ranges in `org_rels` and `org_succs`.
