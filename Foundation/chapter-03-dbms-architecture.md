# Chapter 03 — DBMS Architecture: ANSI/SPARC & Engine Components

*Senior DBMS lecture notes — IIT-style treatment*

**Prerequisites:**

- [Chapter 01 — Data, Database, DBMS, RDBMS](./chapter-01-data-database-dbms-rdbms.md)
- [Chapter 02 — Why file systems fail](./chapter-02-file-systems-inadequacy.md)

**Series:** [Ch 01](./chapter-01-data-database-dbms-rdbms.md) · [Ch 02](./chapter-02-file-systems-inadequacy.md) · [Ch 03](./chapter-03-dbms-architecture.md) · [Ch 04](./chapter-04-relational-model-codd.md)

---

## Why architecture matters

Chapter 02 showed **what** breaks without a DBMS. This chapter shows **how** a DBMS is structured so that:

- Many applications see **different views** of the same data  
- The **logical design** can evolve without rewriting every app  
- The **physical storage** can change without apps knowing disk layout  

That separation is the **ANSI/SPARC three-level architecture** (1975). The **engine components** (query processor, storage manager, transaction manager, buffer manager) are the machinery that implements it.

We continue **Apollo Multispeciality Hospital** after migrating to PostgreSQL.

---

## Part A — Schema vs instance (vocabulary first)

Before the three levels, fix two terms used everywhere in exams:

| Term | Definition | Hospital example |
|------|------------|------------------|
| **Schema** | The **structure** — names of tables, columns, types, constraints, views. The “blueprint.” | `patients(id, full_name, dob, allergy_note)` |
| **Instance** | The **actual data** stored at a moment in time — the current rows. | Row `(P-1042, Meera Sharma, 1978-04-12, …)` |

- Changing **schema**: `ALTER TABLE patients ADD COLUMN abha_id VARCHAR(20);`  
- Changing **instance**: `INSERT`, `UPDATE`, `DELETE` on rows  

**Three levels each have their own schema.** The **mapping** between levels is maintained by the DBMS catalog, not by each application.

---

## Part B — ANSI/SPARC three-level architecture

### Historical note

The **ANSI/SPARC Architecture** (also **three-schema architecture**) was proposed by the DBMS Framework Study Group (~1975) to standardize how databases separate user views from physical storage. Almost every commercial RDBMS follows this idea, even if internal module names differ.

### The three schemas

| Level | Schema name | Also called | What it describes |
|-------|-------------|-------------|-------------------|
| **Level 1 (top)** | **External schema** | View schema, subschema | Data **as one user group or application** needs to see it |
| **Level 2 (middle)** | **Conceptual schema** | Logical schema, community schema | **Entire organization’s** integrated logical model (all entities, relationships, constraints) |
| **Level 3 (bottom)** | **Internal schema** | Physical schema, storage schema | How data are **stored on disk** — files, pages, indexes, hashing |

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL LEVEL (views)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │  View: OPD   │  │ View: Nurse  │  │ View: Billing│  │ View: Lab  │ │
│  │  (no psych)  │  │ (ward only)  │  │ (charges)    │  │ (results)  │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
│         │                 │                 │                │          │
│         └─────────────────┴────────┬────────┴────────────────┘          │
│                                      │  External ↔ Conceptual mapping   │
├──────────────────────────────────────┼──────────────────────────────────┤
│                         CONCEPTUAL LEVEL                              │
│         ┌────────────────────────────────────────────────────┐          │
│         │  Tables: patients, admissions, beds, lab_orders, │          │
│         │  prescriptions, billing_lines, staff, …            │          │
│         │  + PK, FK, CHECK, triggers, enterprise rules       │          │
│         └──────────────────────────┬─────────────────────────┘          │
│                                    │  Conceptual ↔ Internal mapping   │
├────────────────────────────────────┼──────────────────────────────────┤
│                         INTERNAL LEVEL                                │
│         ┌────────────────────────────────────────────────────┐          │
│         │  Heap files, B+ tree indexes on (ward_id, bed_no)│          │
│         │  Page size 8 KB, buffer pool, WAL segment files  │          │
│         │  Partition lab_orders by month, compression …    │          │
│         └────────────────────────────────────────────────────┘          │
│                                    │                                    │
│                                    ▼                                    │
│                              DISK / SSD                                 │
└─────────────────────────────────────────────────────────────────────────┘

     ▲                    ▲                         ▲
     │                    │                         │
  App users          DBA / data modeler         DBMS implementer
  (via SQL views)    (logical design)           (physical tuning)
```

### Conceptual diagram description (for exams)

**Picture three horizontal slabs stacked vertically:**

1. **Top slab — External:** Several “windows” (views), each a subset or reshaping of the hospital data. OPD clerks see one window; billing another. Windows do not see each other’s hidden columns.

2. **Middle slab — Conceptual:** One complete ER-like relational model of the whole hospital — single truth for entities and integrity rules.

3. **Bottom slab — Internal:** Storage structures — which table is a heap file, which index exists, how pages are laid out on disk.

**Arrows:** Mappings connect external → conceptual → internal. The DBMS query processor uses the catalog to translate `SELECT` on a view into access paths on internal structures.

**Users on the left:** Application programs and end-users touch **only** external schemas (or conceptual via SQL if no view). **DBA** designs conceptual. **Internal** is tuned by DBA / performance engineer with `CREATE INDEX`, tablespaces, partitioning — applications remain unaware if **physical data independence** holds.

---

## 1. External schema (Level 1)

### What it represents

An **external schema** is the logical description of data for **one class of users** or **one application**:

- Subset of tables/columns  
- Renamed columns for legacy apps  
- Joined/simplified structures exposed as **views**  
- Security boundaries (hide salary, psychiatric notes)

There are typically **many** external schemas per database; **one** conceptual schema.

### Who interacts

| Actor | Interaction |
|-------|-------------|
| **Application programmers** | Write SQL against views or limited table grants |
| **End users** (via apps) | OPD desk, nurse tablet, billing — never see full `patients` |
| **Report tools** | BI dashboards bound to sanitized views |
| **Intern / role accounts** | `GRANT SELECT` on `opd_patient_basic` only |

**Not** the primary home of the DBA’s full enterprise model — that is conceptual.

### Apollo example

```sql
-- External schema for OPD clerks: no psychiatric_flag, no full address
CREATE VIEW opd_patient_basic AS
SELECT id, full_name, dob, blood_group, allergy_note
FROM patients;

-- External schema for ward nurses: only their ward’s admissions
CREATE VIEW nurse_ward_admissions AS
SELECT a.admission_id, a.patient_id, p.full_name, a.ward_id, a.bed_no
FROM admissions a
JOIN patients p ON p.id = a.patient_id
WHERE a.discharged_at IS NULL;
-- (+ row-level security policy in production)
```

OPD app SQL:

```sql
SELECT * FROM opd_patient_basic WHERE id = 'P-1042';
```

The app **believes** the database *is* `opd_patient_basic` — five columns — even though conceptual `patients` has twenty columns.

---

## 2. Conceptual schema (Level 2)

### What it represents

The **conceptual schema** is the **organization-wide integrated logical database**:

- All base tables, relationships, constraints  
- Enterprise rules: “discharged patient cannot occupy a bed”  
- Normalization choices, entity integrity  

**One conceptual schema per database** (in classical textbook model). It is the DBA’s “source of truth” for what exists in the hospital universe.

### Who interacts

| Actor | Interaction |
|-------|-------------|
| **Database administrator (DBA)** | Creates/alters tables, FKs, policies |
| **Data architect / modeler** | ER → relational design, normalization |
| **Senior backend engineers** | Migrations, `ALTER TABLE`, new modules |
| **DBMS catalog** | Stores metadata used to compile all queries |

End-users **usually should not** depend on raw conceptual tables if views enforce policy — but in practice many apps query tables directly when discipline slips.

### Apollo example (fragment)

```sql
CREATE TABLE patients (
  id              VARCHAR(10) PRIMARY KEY,
  full_name       VARCHAR(100) NOT NULL,
  dob             DATE NOT NULL,
  blood_group     CHAR(3),
  allergy_note    TEXT,
  psychiatric_flag BOOLEAN DEFAULT FALSE,
  abha_id         VARCHAR(20) UNIQUE,
  address         TEXT
);

CREATE TABLE admissions (
  admission_id   SERIAL PRIMARY KEY,
  patient_id     VARCHAR(10) NOT NULL REFERENCES patients(id),
  ward_id        INT NOT NULL,
  bed_no         INT NOT NULL,
  admitted_at    TIMESTAMPTZ NOT NULL,
  discharged_at  TIMESTAMPTZ,
  CHECK (discharged_at IS NULL OR discharged_at >= admitted_at)
);
```

This is the **full hospital model** — not tailored to one department.

---

## 3. Internal schema (Level 3)

### What it represents

The **internal schema** describes **physical storage** and access structures:

- File organization (heap, clustered, columnar)  
- Index types (B+, hash, GiST) and which columns indexed  
- Page size, buffer pool configuration  
- Partitioning, tablespaces, compression, WAL layout  

Applications and external views **must not** reference disk addresses or page numbers.

### Who interacts

| Actor | Interaction |
|-------|-------------|
| **DBA / performance engineer** | `CREATE INDEX`, `EXPLAIN`, partition strategy |
| **DBMS kernel** | Chooses access paths, maintains statistics |
| **Storage admin** | Disk arrays, backup at volume level |
| **OS / file system** | Ultimately holds `.db` files or tablespace files |

Application programmers **should not** code to internal schema; exams still ask you to **name** what lives here.

### Apollo example

```sql
-- Physical tuning: not part of app's logical world
CREATE INDEX idx_admissions_ward_bed
  ON admissions (ward_id, bed_no)
  WHERE discharged_at IS NULL;

-- Table stored in fast SSD tablespace (implementation-specific)
-- CREATE TABLE admissions (...) TABLESPACE ward_hot;
```

**Internal change:** DBA adds `idx_admissions_patient` on `(patient_id)` because ward queries are slow. OPD view and conceptual `admissions` **unchanged** — only query **plans** change (physical independence).

---

## Part C — Mappings between levels

| Mapping | Direction | Maintained by |
|---------|-----------|---------------|
| **External ↔ Conceptual** | View definition, column projection, joins, RLS | `CREATE VIEW`, grants, policies |
| **Conceptual ↔ Internal** | Table → heap file; index → B+ tree; tuple → page/slot | Catalog + storage engine |

When you run:

```sql
SELECT full_name FROM opd_patient_basic WHERE id = 'P-1042';
```

the **query processor**:

1. Resolves `opd_patient_basic` → definition over `patients` (external → conceptual)  
2. Plans access → heap or index on `patients.id` (conceptual → internal)  
3. **Buffer manager** brings pages into memory; **transaction manager** handles locks/isolation  

---

## Part D — Data independence

**Data independence** means the capacity to change schema at one level without forcing changes at adjacent higher levels (or without breaking applications).

### Type 1: Logical data independence

**Definition:** Ability to change the **conceptual schema** without changing **external schemas** (applications).

**Formal:** External schemas remain valid when conceptual tables/columns are added, removed, or reorganized — if mappings (views) absorb the change.

#### Concrete example 1 — Add column (Apollo)

**Change at conceptual level:**

```sql
ALTER TABLE patients ADD COLUMN emergency_contact VARCHAR(20);
```

**OPD app** still runs:

```sql
SELECT id, full_name, dob, blood_group, allergy_note
FROM opd_patient_basic;
```

View definition unchanged → **five columns** returned as before → **logical independence achieved.**

**Without a view:** If OPD app used `SELECT * FROM patients` and parsed column positions, adding a column in the middle could break brittle clients — independence violated at application discipline level.

#### Concrete example 2 — Split table (restructuring)

**Conceptual change:** Split `patients` into `patients_core` + `patients_extended` for normalization.

**Independence mechanism:**

```sql
CREATE VIEW patients AS
SELECT c.id, c.full_name, c.dob, e.psychiatric_flag, e.address, ...
FROM patients_core c
JOIN patients_extended e ON e.id = c.id;
```

Apps still query `patients` view name → external/conceptual mapping hides split.

**Limits:** Dropping a column that a view exposes, or changing types incompatibly, **breaks** independence — DBA must alter views too.

---

### Type 2: Physical data independence

**Definition:** Ability to change the **internal schema** without changing the **conceptual schema** (and thus without changing applications/external views).

**Formal:** Storage structures and access paths change; logical table definitions stay the same.

#### Concrete example 1 — New index (Apollo)

**Physical change:**

```sql
CREATE INDEX idx_patients_name ON patients (full_name);
```

**Conceptual:** `patients` table definition identical.  
**External:** `opd_patient_basic` identical.  
**OPD app code:** unchanged.  
**Effect:** Same SQL, faster execution plan — classic **physical independence**.

#### Concrete example 2 — Storage reorganization

**Physical changes (no SQL visible to app):**

- Move `lab_orders` to partitioned monthly tablespaces  
- Switch `admissions` from heap to cluster on `ward_id`  
- Increase page size on new tablespace during hardware refresh  

Billing still:

```sql
INSERT INTO billing_lines (admission_id, amount, ...) VALUES (...);
```

Table and column names at conceptual level **unchanged** — only **performance and admin** change.

#### Concrete example 3 — What physical independence does NOT mean

If DBA **drops** the only index that made a uniqueness check fast, apps still work but timeouts may occur — performance, not correctness. Independence is about **logical interface stability**, not SLA.

---

### Comparison table (exam-ready)

| | **Logical independence** | **Physical independence** |
|---|--------------------------|---------------------------|
| **What changes** | Conceptual (tables, columns, splits) | Internal (indexes, files, partitions) |
| **What stays stable** | External views / app SQL against views | Conceptual + external |
| **Typical tool** | Views, synonyms, migrations with compatibility views | `CREATE INDEX`, tablespace move, recluster |
| **Who benefits** | App teams, multiple departments | DBA performance tuning |
| **Chapter 2 link** | Fixes **data isolation** | Apps don’t depend on file byte layout |

---

## Part E — Components of the DBMS engine

The three-level architecture is the **logical layering**. The **engine** is the runtime that executes queries and manages data on disk.

```text
                    ┌─────────────────────────────────────┐
                    │           Client / SQL API           │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │         QUERY PROCESSOR              │
                    │  Parser → Validator → Optimizer      │
                    │  → Execution plan (relational ops)   │
                    └──────────────────┬──────────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
         ▼                             ▼                             ▼
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│ TRANSACTION     │◄────────►│ BUFFER MANAGER  │◄────────►│ STORAGE MANAGER │
│ MANAGER         │          │ (buffer pool)   │          │ (files, indexes)│
│ ACID, locks,    │          │ page fetch/     │          │ heap, B+, WAL   │
│ log, schedule   │          │ flush, pin/unpin│          │ on disk         │
└────────┬────────┘          └────────┬────────┘          └────────┬────────┘
         │                            │                            │
         └────────────────────────────┴────────────────────────────┘
                                      │
                                      ▼
                               DISK / SSD (+ WAL)
```

### 1. Query processor

**Role:** Turn a declarative SQL statement into an efficient **procedure** over internal structures.

| Stage | Function | Apollo example |
|-------|----------|----------------|
| **Parser** | Syntax tree from `SELECT …` | Detect typo in `SELCT` |
| **Validator** | Check against **catalog** (conceptual + external): tables exist, types match, permissions | Intern cannot `SELECT psychiatric_flag` if not granted |
| **Optimizer** | Choose plan: nested-loop vs hash join; index vs seq scan | `WHERE ward_id = 3 AND bed_no = 12` → use `idx_admissions_ward_bed` |
| **Execution engine** | Run plan operators: scan, join, sort, aggregate | Return Meera’s row to OPD app |

**Interaction with architecture:** Uses **external → conceptual** mapping (view expansion), then **conceptual → internal** (statistics, indexes).

**Exam point:** Optimizer cost model uses **logical** properties (cardinality, selectivity) and **physical** metadata (index pages, sequential I/O cost).

---

### 2. Storage manager

**Role:** Manage **persistent** representation: files, pages, indexes, free space, and interaction with OS.

| Responsibility | Detail |
|----------------|--------|
| **File allocation** | Table → heap file or partition files |
| **Index structures** | B+ tree for `(patient_id)`, hash for point lookups |
| **Record placement** | Tuple format, slot directory on page |
| **WAL / log records** | Physical log entries for recovery (with transaction manager) |
| **Space reclamation** | Vacuum, defragmentation (PostgreSQL `VACUUM`) |

**Apollo:** Table `lab_orders` with 50M rows — storage manager stores pages on disk; **does not** parse SQL.

**Boundary:** Storage manager asks buffer manager for a **page** in memory; does not implement isolation semantics alone.

---

### 3. Transaction manager

**Role:** Ensure **ACID** execution and **correct interleaving** of concurrent transactions.

| Responsibility | Detail |
|----------------|--------|
| **Transaction boundaries** | `BEGIN`, `COMMIT`, `ROLLBACK` |
| **Concurrency control** | Locks, latches, or MVCC snapshot rules |
| **Logging** | Write-ahead log: undo/redo for crash recovery |
| **Scheduling** | Serializable or weaker isolation per session setting |
| **Deadlock detection** | Detect wait cycles, abort victim transaction |

**Apollo discharge transaction** (Chapter 02): bed free + admission update + billing line — transaction manager guarantees **all commit or none**, and coordinates with log for crash recovery.

**Interaction:** On `UPDATE beds`, transaction manager acquires lock; buffer manager loads page; storage manager eventually persists dirty page **after** log flush (WAL protocol).

---

### 4. Buffer manager

**Role:** Cache **disk pages in main memory** (buffer pool) because disk I/O dominates cost.

| Responsibility | Detail |
|----------------|--------|
| **Page fetch** | On miss, read page from storage manager into frame |
| **Replacement policy** | LRU, clock, or DBMS-specific eviction |
| **Pin / unpin** | Pin while operator uses page; prevent eviction mid-read |
| **Dirty page tracking** | Modified in memory → flush to disk on checkpoint or eviction |
| **Coherency with log** | **WAL rule:** log record on disk before dirty data page written |

**Apollo:** Ten nurses query bed status — same **index root page** stays in buffer pool; repeated disk reads avoided.

**Exam trap:** Buffer manager ≠ transaction manager. Buffer = **performance** cache; transaction = **correctness** and recovery semantics.

---

### Component interaction walkthrough

**Query:** `UPDATE beds SET status = 'free' WHERE ward_id = 3 AND bed_no = 12;`

```text
1. Query processor: parse, validate, plan (index seek on admissions/beds)
2. Transaction manager: begin txn, acquire row lock on bed 12
3. Buffer manager: fix page containing row in pool (fetch if miss)
4. Execution: update tuple in memory page → mark dirty
5. Transaction manager: write WAL record, later commit
6. Buffer manager: eventually flush dirty page (after WAL safe)
7. Storage manager: page written to table file on SSD
```

---

## Part F — Who sits where (summary matrix)

| Level / component | Primary human actors | Primary software |
|-------------------|---------------------|------------------|
| External schema | App devs, end users, analysts | Views, grants, ORM models |
| Conceptual schema | DBA, architects | `CREATE TABLE`, FKs, triggers |
| Internal schema | DBA performance, sysadmin | Indexes, partitions, tablespaces |
| Query processor | (automatic) | Parser, optimizer, executor |
| Storage manager | (automatic) | File/index layer |
| Transaction manager | (automatic) | Lock, log, commit |
| Buffer manager | (automatic) | Buffer pool |

---

## Part G — Interview questions (with deep answers)

### Q1. “How many schemas can a database have? Is there one external schema or many?”

**Deep answer:**

In the **classical ANSI/SPARC model**:

- **Exactly one conceptual schema** per database (integrated logical model of the enterprise).  
- **Exactly one internal schema** per database (one physical storage design at a time — though it may include many files/indexes).  
- **Many external schemas** — one per user group, application family, or security domain.

Modern systems blur the wording:

- PostgreSQL “schema” (`CREATE SCHEMA billing`) is a **namespace** within one database — closer to a **module of the conceptual model**, not an ANSI external schema.  
- Each **view** + **role grants** implements an **external schema** in the ANSI sense.

**Interview closure:** “Many views/roles at the top; one logical database in the middle; one physical design controlled by the DBA at the bottom.”

---

### Q2. “Explain logical vs physical data independence. Can you have one without the other?”

**Deep answer:**

| | Logical | Physical |
|---|---------|----------|
| **Changes** | Tables split, column added, entity renamed via view | Index added, partition, file moved |
| **Protects** | Applications from conceptual redesign | Applications from storage tuning |
| **Mechanism** | Views, compatibility layers | Catalog + optimizer replanning |

**Independent in principle:**

- **Physical without logical change:** Add index daily — common in production.  
- **Logical without physical change:** Add column + update view — apps unchanged; no index change required.

**Neither is automatic:**

- Logical: `DROP COLUMN allergy_note` breaks `opd_patient_basic` until view/ app fixed — **independence lost**.  
- Physical: Rare rewrites (rewrite table to columnar) might require maintenance window — apps still run but ops impact.

**Yes, you can have one without the other** — they address different layers. Good DBAs maintain **both**: stable APIs (views) and freedom to tune disk.

**Link to Chapter 02:** Logical independence fixes **program–data dependence** at the logical layer; physical independence fixes it at the storage layer.

---

### Q3. “What is the difference between buffer manager and storage manager?”

**Deep answer:**

| | **Buffer manager** | **Storage manager** |
|---|-------------------|---------------------|
| **Abstraction** | **Pages in RAM** (buffer pool frames) | **Files/pages on disk** (persistent) |
| **Goal** | Minimize latency via caching | Organize durable data structures |
| **Lifetime** | Volatile — lost on crash (except logged changes) | Persistent — survives restart |
| **Policy** | Replacement (LRU/clock), pin/unpin | Allocation, index split, free space map |
| **Cooperates with** | Transaction manager (WAL before flush) | OS block I/O |

**Analogy (hospital pharmacy):**

- **Storage manager** = warehouse shelving system (where each drug box lives on shelves).  
- **Buffer manager** = cart of boxes at the dispensing counter (fast access; only a subset of warehouse on the cart).  
- **Transaction manager** = rules for “don’t sell same vial twice” and ledger of pending sales.

**Common mistake:** Saying buffer manager “stores the database.” It stores **copies of disk pages** temporarily. Authoritative copy is on disk under storage manager control, with WAL for recovery.

---

## Part H — Exercises (attempt before hints)

### Exercise 1 — Classify changes

For each change below at Apollo, state:

1. **Level** (external / conceptual / internal)  
2. **Independence type** affected (logical, physical, or neither)  
3. **Who must be notified** (OPD app team, DBA only, nobody if view absorbs)

| # | Change |
|---|--------|
| (a) `CREATE VIEW lab_results_readonly AS SELECT …` | |
| (b) `CREATE INDEX ON lab_orders (patient_id, ordered_at)` | |
| (c) `ALTER TABLE patients RENAME COLUMN phone TO mobile` — OPD uses view listing `phone` | |
| (d) Drop table `legacy_admissions_2010` from conceptual schema | |
| (e) Move `lab_orders` partition May 2026 to cheaper disk tablespace | |

<details>
<summary>Hint (after you try)</summary>

(a) external, neither independence “change” — new view. (b) internal, physical. (c) conceptual + view mapping — logical unless view updated. (d) conceptual — may break apps. (e) internal, physical.
</details>

---

### Exercise 2 — Trace a query through the engine

Given:

```sql
SELECT full_name, allergy_note
FROM opd_patient_basic
WHERE id = 'P-1042';
```

1. Draw or narrate the path through **query processor → transaction manager → buffer manager → storage manager** (at least one sentence per component).  
2. Where does **external → conceptual** mapping happen?  
3. DBA adds `CREATE INDEX ON patients (id)` — which component’s behavior changes first, and do OPD apps need a redeploy?

<details>
<summary>Hint (after you try)</summary>

View expansion in parser/validator; plan uses index in optimizer/executor; buffer fetches index leaf + heap page; txn may take shared lock under isolation. Physical independence → optimizer only.
</details>

---

## Part I — Chapter summary

| Idea | One sentence |
|------|----------------|
| **External schema** | Per-group view of data; apps see this |
| **Conceptual schema** | Whole hospital logical model; one integrated design |
| **Internal schema** | Disk/index/page layout; DBA tunes performance |
| **Logical independence** | Change conceptual DB without breaking apps (views) |
| **Physical independence** | Change storage without changing tables/apps |
| **Query processor** | SQL → validated, optimized plan → execution |
| **Storage manager** | Durable files, indexes, pages on disk |
| **Transaction manager** | ACID, locks/MVCC, log, recovery |
| **Buffer manager** | RAM cache of pages; WAL-coordinated flush |

Architecture exists so **many apps, one truth, tunable disk** — without returning to the file-system chaos of Chapter 02.

---

*Prev: [Chapter 02](./chapter-02-file-systems-inadequacy.md) · Next: [Chapter 04 — Relational model](./chapter-04-relational-model-codd.md)*
