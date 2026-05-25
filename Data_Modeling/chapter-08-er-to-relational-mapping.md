# Chapter 08 — ER Diagram → Relational Schema (Complete Mapping)

*Senior DBMS lecture notes — IIT-style treatment*

**Prerequisites:**

- [Chapter 05 — ER modeling](./chapter-05-er-modeling-university.md)
- [Chapter 07 — Cardinality & participation](./chapter-07-cardinality-participation.md)

**Data modeling track:** [Ch 05](./chapter-05-er-modeling-university.md) · … · [Ch 07](./chapter-07-cardinality-participation.md) · [Ch 08](./chapter-08-er-to-relational-mapping.md) *(this chapter)*

---

## Overview — the systematic 7-step process

Converting an ER diagram to a relational schema is **not** guesswork. Follow this **fixed order** (standard textbook algorithm; names vary slightly by author):

| Step | ER construct | Relational output |
|------|--------------|-------------------|
| **1** | Strong entity types | One table per strong entity; PK = key attribute(s) |
| **2** | Simple attributes | Columns on that entity’s table |
| **3** | Composite attributes | **Flatten** to simple columns (no nested struct in classical 1NF) |
| **4** | Multivalued attributes | **New table** + FK to owner; composite PK |
| **5** | Weak entity types | Table with **composite PK** = owner FK + partial key |
| **6** | Binary relationships | 1:1 / 1:N / M:N rules (FK or junction) |
| **7** | Higher-degree (ternary+) | Table with **FK per participating entity** (+ relationship attrs) |

**After all steps:** Merge redundant tables from relationship mapping, add `REFERENCES`, `CHECK`, document participation as `NOT NULL`.

**Worked example throughout:** **Apollo Hospital `ApolloDB`** (links to [Chapter 02](../Foundation/chapter-02-file-systems-inadequacy.md)).

---

## ApolloDB — ER diagram (conceptual, before mapping)

**Strong entities:**

- `PATIENT(id, …)` — key `id` (e.g. P-1042)  
- `DOCTOR(emp_id, …)`  
- `DEPARTMENT(dept_code, name, …)`  
- `WARD(ward_id, name, …)`  
- `DRUG(drug_id, name, …)`

**Composite attribute on PATIENT:** `address` → `{street, city, pincode}`

**Multivalued on PATIENT:** `phone_numbers`

**Weak entity:** `BED(bed_no, …)` — owner `WARD`; partial key `bed_no`

**Binary relationships:**

| Relationship | Type | Notes |
|--------------|------|-------|
| `Works_In` | DOCTOR — DEPARTMENT | N:1 (many doctors, one dept each) |
| `Admits` | PATIENT — WARD | M:N via `ADMISSION` (dates, bed) |
| `Prescribes` | DOCTOR — DRUG | M:N with `dosage`, `date` on relationship |
| `Assigned_Physician` | PATIENT — DOCTOR | 1:1 optional (attending doctor) |
| `Occupies` | ADMISSION — BED | 1:1 during stay (see mapping) |

**Ternary:** `TREATS(DOCTOR, PATIENT, TREATMENT)` — doctor performs treatment on patient; attr `treatment_date`, `notes`

**Diagram description (exam sketch):**

Center: `PATIENT`. Left: `DOCTOR` — `Works_In` → `DEPARTMENT`. `Assigned_Physician` 1:1 between PATIENT and DOCTOR. Below: `WARD` — weak `BED`. M:N `Admits` to `ADMISSION` associative entity (or relationship with attrs). `Prescribes` M:N to `DRUG`. Ternary diamond `TREATS` touching DOCTOR, PATIENT, and entity `TREATMENT` (procedure code).

---

## Step 1 — Map strong entity types

**Rule:** Each **strong entity type** → **one relation** (table) named after the entity (singular or plural — be consistent).

**Primary key:** Key attribute(s) from ER → `PRIMARY KEY`.

### ApolloDB — Step 1 output

| ER entity | Table | Primary key |
|-----------|-------|-------------|
| PATIENT | `patients` | `id` |
| DOCTOR | `doctors` | `emp_id` |
| DEPARTMENT | `departments` | `dept_code` |
| WARD | `wards` | `ward_id` |
| DRUG | `drugs` | `drug_id` |
| TREATMENT | `treatments` | `treatment_code` |

```sql
CREATE TABLE patients (
  id VARCHAR(10) PRIMARY KEY
  -- more columns in Step 2–4
);

CREATE TABLE doctors (
  emp_id VARCHAR(10) PRIMARY KEY
);

CREATE TABLE departments (
  dept_code VARCHAR(10) PRIMARY KEY,
  name    VARCHAR(100) NOT NULL
);

CREATE TABLE wards (
  ward_id INT PRIMARY KEY,
  name    VARCHAR(50) NOT NULL
);

CREATE TABLE drugs (
  drug_id VARCHAR(20) PRIMARY KEY,
  name    VARCHAR(100) NOT NULL
);

CREATE TABLE treatments (
  treatment_code VARCHAR(20) PRIMARY KEY,
  description    VARCHAR(255) NOT NULL
);
```

**Do not** create tables yet for weak `BED` (Step 5) or pure M:N diamonds without attributes until Step 6.

---

## Step 2 — Map simple attributes

**Rule:** Each **simple attribute** of an entity → **column** in that entity’s table with appropriate SQL type.

### ApolloDB — additions

| Entity | Simple attributes → columns |
|--------|----------------------------|
| PATIENT | `full_name`, `dob`, `blood_group`, `allergy_note` |
| DOCTOR | `full_name`, `specialization`, `license_no` |
| DRUG | `unit_price`, `stock_qty` |

```sql
-- Extend patients (shown as full CREATE later in one block)
-- full_name VARCHAR(100) NOT NULL, dob DATE NOT NULL, blood_group CHAR(3), allergy_note TEXT
```

**Relationship attributes** (e.g. `dosage` on `Prescribes`) are mapped in **Step 6**, not on entity tables.

---

## Step 3 — Map composite attributes

**Rule:** **Do not** create a column `address` as opaque blob for classical relational design. **Flatten** each component into its own column.

| ER | Relational |
|----|------------|
| `address {street, city, pincode}` | `street`, `city`, `pincode` |

**Optional:** Keep `address_line` computed in app or add `CHECK` on `pincode` length.

### ApolloDB

`PATIENT.address` → `street VARCHAR(200)`, `city VARCHAR(80)`, `pincode CHAR(6)` on `patients`.

**Exam note:** Composite keys (e.g. `{ward_id, bed_no}`) are **not** composite *attributes* — they are keys for weak entities (Step 5).

---

## Step 4 — Map multivalued attributes

**Rule:** Multivalued attribute `A` of entity `E` → **new table** `E_A` or descriptive name:

- Columns: FK to `E`’s PK + multivalued value (or decomposed attrs)  
- **Primary key:** `(fk_to_E, value)` or `(fk_to_E, seq)` if duplicates forbidden per value

### ApolloDB

`PATIENT.phone_numbers` (multivalued) → table `patient_phones`:

| Column | Role |
|--------|------|
| `patient_id` | FK → `patients(id)` |
| `phone` | phone number |
| `phone_type` | optional: mobile / home / emergency |

```sql
CREATE TABLE patient_phones (
  patient_id VARCHAR(10) NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
  phone      VARCHAR(15) NOT NULL,
  phone_type VARCHAR(20) DEFAULT 'mobile',
  PRIMARY KEY (patient_id, phone)
);
```

**Never** store `phone1, phone2, phone3` on `patients` unless you accept 1NF violation.

---

## Step 5 — Map weak entity types

**Rule:** Weak entity `W` owned by strong `S` via identifying relationship:

- Table `w` with columns: **all simple attributes of W**  
- **FK** to `s` (owner PK), `NOT NULL`  
- **PK** = `(owner_fk, partial_key)`  

### ApolloDB

`BED` weak under `WARD`, partial key `bed_no`:

```sql
CREATE TABLE beds (
  ward_id INT NOT NULL REFERENCES wards(ward_id),
  bed_no  INT NOT NULL,
  status  VARCHAR(20) NOT NULL DEFAULT 'free'
    CHECK (status IN ('free', 'occupied', 'maintenance')),
  PRIMARY KEY (ward_id, bed_no)
);
```

Identifying relationship `Has_Bed` is **embedded** in composite PK — **no separate** `has_bed` table unless you also need relationship attributes (e.g. `installed_date` → add column on `beds`).

---

## Step 6 — Map binary relationships

### 6A — One-to-one (1:1)

**Rule:** Merge FK into **one** side (choose by participation and query frequency):

| Participation | Typical FK placement |
|---------------|------------------------|
| Both partial | Either side, `UNIQUE` nullable FK |
| One total, one partial | FK on **total** side, `NOT NULL` |
| Both total | FK either side, `NOT NULL` |

### ApolloDB — `Assigned_Physician` (PATIENT — DOCTOR, 0..1 : 0..1)

Each patient **at most one** attending doctor; each doctor **at most one** attending patient in this simplified model (teaching example):

→ `patients.attending_doctor_id UNIQUE NULL REFERENCES doctors(emp_id)`

Or reverse: `doctors.attending_patient_id UNIQUE NULL` — pick one; **not both** (redundant update anomalies).

```sql
-- On patients table:
-- attending_doctor_id VARCHAR(10) UNIQUE REFERENCES doctors(emp_id)
```

---

### 6B — One-to-many (1:N)

**Rule:** Place **FK on the many side** referencing the one side. Relationship attributes → columns on **many side** table (or on FK table if many side is weak/associative).

| Participation on many side | FK |
|----------------------------|-----|
| Total | `NOT NULL` |
| Partial | `NULL` allowed |

### ApolloDB — `Works_In` (DOCTOR many — DEPARTMENT one)

Many doctors, one department each:

```sql
-- On doctors:
-- dept_code VARCHAR(10) NOT NULL REFERENCES departments(dept_code)
```

---

### 6C — Many-to-many (M:N)

**Rule:** Create **junction table** (associative relation):

- PK: composite `(fk1, fk2)` or surrogate `id` + `UNIQUE(fk1, fk2)`  
- Include **relationship attributes** as columns

### ApolloDB — `Prescribes` (DOCTOR — DRUG)

```sql
CREATE TABLE prescribes (
  emp_id   VARCHAR(10) NOT NULL REFERENCES doctors(emp_id),
  drug_id  VARCHAR(20) NOT NULL REFERENCES drugs(drug_id),
  prescribed_on DATE NOT NULL,
  dosage        VARCHAR(50) NOT NULL,
  PRIMARY KEY (emp_id, drug_id, prescribed_on)
);
```

### ApolloDB — `Admits` (PATIENT — WARD) with admission facts

M:N with attributes `admitted_at`, `discharged_at`, bed reference → **strong associative entity** `ADMISSION` is cleaner than bare diamond:

| Design | Tables |
|--------|--------|
| ER | `ADMISSION` entity between PATIENT and WARD |
| Relational | `admissions(admission_id PK, patient_id FK, ward_id FK, admitted_at, discharged_at, ward_id, bed_no FK)` |

```sql
CREATE TABLE admissions (
  admission_id  SERIAL PRIMARY KEY,
  patient_id    VARCHAR(10) NOT NULL REFERENCES patients(id),
  ward_id       INT NOT NULL,
  bed_no        INT NOT NULL,
  admitted_at   TIMESTAMPTZ NOT NULL,
  discharged_at TIMESTAMPTZ,
  FOREIGN KEY (ward_id, bed_no) REFERENCES beds(ward_id, bed_no),
  CHECK (discharged_at IS NULL OR discharged_at >= admitted_at)
);
```

`Admits` M:N is **realized** by `admissions` linking patient + ward + bed + time.

**1:N sub-case inside admission:** Each admission **occupies** one bed (1:1 during stay) → FK `(ward_id, bed_no)` on `admissions`, not separate `occupies` table.

---

### Step 6 summary table

| Ratio | Relational pattern |
|-------|-------------------|
| 1:1 | FK + `UNIQUE` on one table |
| 1:N | FK on many side |
| M:N | Junction / associative table |

---

## Step 7 — Map higher-degree relationships (ternary+)

**Rule:** Relationship type `R` involving entity types `E1, E2, E3, …` → table `r` with:

- One **FK column (or set)** per participating entity  
- **PK** often composite `(fk1, fk2, fk3)` if identifying tuple is unique  
- All **relationship attributes** on this table  

**Do not** decompose ternary into three binary tables unless you can prove **lossless join** — usually **lossy** for ternary facts.

### ApolloDB — `TREATS(DOCTOR, PATIENT, TREATMENT)`

Fact: doctor D performed treatment T on patient P on a date.

```sql
CREATE TABLE treats (
  emp_id         VARCHAR(10) NOT NULL REFERENCES doctors(emp_id),
  patient_id     VARCHAR(10) NOT NULL REFERENCES patients(id),
  treatment_code VARCHAR(20) NOT NULL REFERENCES treatments(treatment_code),
  treatment_date DATE NOT NULL,
  notes          TEXT,
  PRIMARY KEY (emp_id, patient_id, treatment_code, treatment_date)
);
```

**Binary decomposition trap:** Tables `doctor_treatment`, `patient_treatment`, `doctor_patient` cannot recover **which doctor did which treatment on which patient** without recombining incorrectly.

---

## Complete ApolloDB — final SQL (all steps merged)

```sql
-- ========== Step 1–3: Strong entities + simple + composite ==========
-- Run in dependency order (parents before children).

CREATE TABLE departments (
  dept_code VARCHAR(10) PRIMARY KEY,
  name      VARCHAR(100) NOT NULL
);

CREATE TABLE wards (
  ward_id INT PRIMARY KEY,
  name    VARCHAR(50) NOT NULL
);

CREATE TABLE drugs (
  drug_id    VARCHAR(20) PRIMARY KEY,
  name       VARCHAR(100) NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL CHECK (unit_price >= 0),
  stock_qty  INT NOT NULL CHECK (stock_qty >= 0)
);

CREATE TABLE treatments (
  treatment_code VARCHAR(20) PRIMARY KEY,
  description    VARCHAR(255) NOT NULL
);

CREATE TABLE doctors (
  emp_id          VARCHAR(10) PRIMARY KEY,
  full_name       VARCHAR(100) NOT NULL,
  specialization  VARCHAR(80),
  license_no      VARCHAR(30) UNIQUE NOT NULL,
  dept_code       VARCHAR(10) NOT NULL REFERENCES departments(dept_code)  -- Step 6B
);

CREATE TABLE patients (
  id            VARCHAR(10) PRIMARY KEY,
  full_name     VARCHAR(100) NOT NULL,
  dob           DATE NOT NULL,
  blood_group   CHAR(3),
  allergy_note  TEXT,
  street        VARCHAR(200),   -- Step 3: composite address flattened
  city          VARCHAR(80),
  pincode       CHAR(6),
  attending_doctor_id VARCHAR(10) UNIQUE
    REFERENCES doctors(emp_id)   -- Step 6A: 1:1 Assigned_Physician
);

-- ========== Step 4: Multivalued ==========

CREATE TABLE patient_phones (
  patient_id VARCHAR(10) NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
  phone      VARCHAR(15) NOT NULL,
  phone_type VARCHAR(20) DEFAULT 'mobile',
  PRIMARY KEY (patient_id, phone)
);

-- ========== Step 5: Weak entity ==========

CREATE TABLE beds (
  ward_id INT NOT NULL REFERENCES wards(ward_id),
  bed_no  INT NOT NULL,
  status  VARCHAR(20) NOT NULL DEFAULT 'free'
    CHECK (status IN ('free', 'occupied', 'maintenance')),
  PRIMARY KEY (ward_id, bed_no)
);

-- ========== Step 6: Binary M:N and associative ==========

CREATE TABLE admissions (
  admission_id  SERIAL PRIMARY KEY,
  patient_id    VARCHAR(10) NOT NULL REFERENCES patients(id),
  ward_id       INT NOT NULL,
  bed_no        INT NOT NULL,
  admitted_at   TIMESTAMPTZ NOT NULL,
  discharged_at TIMESTAMPTZ,
  FOREIGN KEY (ward_id, bed_no) REFERENCES beds(ward_id, bed_no),
  CHECK (discharged_at IS NULL OR discharged_at >= admitted_at)
);

CREATE TABLE prescribes (
  emp_id        VARCHAR(10) NOT NULL REFERENCES doctors(emp_id),
  drug_id       VARCHAR(20) NOT NULL REFERENCES drugs(drug_id),
  prescribed_on DATE NOT NULL,
  dosage        VARCHAR(50) NOT NULL,
  PRIMARY KEY (emp_id, drug_id, prescribed_on)
);

-- ========== Step 7: Ternary ==========

CREATE TABLE treats (
  emp_id         VARCHAR(10) NOT NULL REFERENCES doctors(emp_id),
  patient_id     VARCHAR(10) NOT NULL REFERENCES patients(id),
  treatment_code VARCHAR(20) NOT NULL REFERENCES treatments(treatment_code),
  treatment_date DATE NOT NULL,
  notes          TEXT,
  PRIMARY KEY (emp_id, patient_id, treatment_code, treatment_date)
);
```

### Deployment order

1. `departments`, `wards`, `drugs`, `treatments`  
2. `doctors` → `patients`  
3. `patient_phones`, `beds`, `admissions`, `prescribes`, `treats`

---

## Mapping checklist (use in exams)

```text
[ ] Step 1: Every strong entity → table + PK
[ ] Step 2: Simple attrs → columns
[ ] Step 3: Composite → flattened columns
[ ] Step 4: Each multivalued → new table + FK
[ ] Step 5: Each weak → table, PK = owner_FK + partial_key
[ ] Step 6a: Each 1:1 → FK + UNIQUE on one side
[ ] Step 6b: Each 1:N → FK on many side
[ ] Step 6c: Each M:N → junction + relationship attrs
[ ] Step 7: Each ternary+ → single table with all FKs
[ ] Participation → NOT NULL on FK where total
[ ] No duplicate tables for same fact
```

---

## Part I — Interview questions (with deep answers)

### Q1. “Why can’t we map M:N Prescribes by adding drug_id to doctors?”

**Deep answer:**

Adding `drug_id` to `doctors` allows **only one drug per doctor** (or repeated doctor rows — entity integrity violation). M:N means **one doctor, many drugs** and **one drug, many doctors**.

**Correct:** `prescribes(emp_id, drug_id, …)` junction.

**Symmetric mistake:** `drug_id` on `patients` for prescriptions confuses **who prescribed** (doctor) with **who receives** (patient) — need doctor and patient roles, often `prescribes` or `prescription` header with lines.

**Sound bite:** “FK on one side encodes 1:N; M:N always needs a separate table or duplicate entity rows.”

---

### Q2. “When do you merge a 1:1 relationship into one table instead of two with FK?”

**Deep answer:**

**Merge** when:

- Participation is **total on both sides** (every patient has exactly one medical_record row)  
- Entities are **always queried together**  
- No growth asymmetry (both sides similar column count)

**Example:** `employees` + `employee_parking` 1:1 total total → optional columns on `employees` (`slot_id`).

**Keep separate** when:

- One side is optional (partial) and large nullable columns rare on main table  
- Security boundary (HR vs facilities)  
- Different lifecycles (patient vs organ_donor_card 0..1)

**ApolloDB:** `Assigned_Physician` 0..1:0..1 → **separate** `patients.attending_doctor_id` FK, not merge patient into doctor row.

**Lossless join:** Merged table must reconstruct both entity types without NULL confusion.

---

### Q3. “Map weak entity BED. Why is PK (ward_id, bed_no) not just bed_no?”

**Deep answer:**

`bed_no` is **partial key** — unique **only within a ward**. Ward 3 bed 12 and Ward 5 bed 12 are different beds.

Weak entity table **must** include **owner’s key** in PK: `(ward_id, bed_no)`.

**Identifying relationship** `Has_Bed` does not need its own table unless relationship attributes exist (e.g. `last_sanitized_at` → column on `beds`).

**Contrast strong entity:** `PATIENT.id` is globally unique — single-column PK.

**Exam trap:** Surrogate `bed_id` serial PK is **allowed** in implementation (strong-style surrogate) but ER weak form still documents logical `(ward_id, bed_no)`.

---

## Part J — Exercises

### Exercise 1 — Map ConnectHub ER to SQL

Using [Chapter 06 ConnectHub ER](./chapter-06-er-diagrams-notation.md):

1. Apply **steps 1–7** in writing (table list + PK/FK per step, no need to duplicate full SQL).  
2. Produce final `CREATE TABLE` for: `users`, `posts`, `comments` (weak), `likes`, `follows`, `post_hashtags` (multivalued).  
3. State **creation order** to satisfy FK dependencies.

<details>
<summary>Hint (after you try)</summary>

users, posts (user_id FK), comments (post_id, comment_seq PK), likes junction, follows junction, post_hashtags multivalued. Order: users → posts → comments, likes, hashtags, follows.
</details>

---

### Exercise 2 — ApolloDB extension (ternary + M:N)

Add to ApolloDB:

- **NURSE** strong entity; **M:N** `Monitors` between NURSE and WARD with `shift` attribute.  
- **Ternary** `Administers(NURSE, PATIENT, DRUG)` with `administered_at`, `dose_ml`.

1. Which **step** handles each construct?  
2. Write SQL for new tables only.  
3. Can `Administers` be replaced by two binary tables `Nurse_Drug` and `Patient_Drug` without losing information? Argue briefly.

<details>
<summary>Hint (after you try)</summary>

Monitors: Step 6c junction. Administers: Step 7 ternary table. Two binaries lossy — cannot tell which nurse gave which drug to which patient.
</details>

---

## Part K — Chapter summary

| Step | ApolloDB artifact |
|------|-------------------|
| 1 | `patients`, `doctors`, `wards`, … |
| 2 | `full_name`, `dob`, `stock_qty`, … |
| 3 | `street`, `city`, `pincode` |
| 4 | `patient_phones` |
| 5 | `beds(ward_id, bed_no)` |
| 6 | `doctors.dept_code`, `patients.attending_doctor_id`, `prescribes`, `admissions` |
| 7 | `treats(emp_id, patient_id, treatment_code, …)` |

ER drawing is the **spec**; the seven steps are the **compiler** from conceptual model to `CREATE TABLE`.

---

*Prev: [Chapter 07 — Cardinality & participation](./chapter-07-cardinality-participation.md) · Next: Chapter 09 (normalization — suggested)*
