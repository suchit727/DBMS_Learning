# Chapter 02 — Why File Systems Fail at Data Management

*Senior DBMS lecture notes — IIT-style treatment*

**Prerequisites:** [Chapter 01 — Data, Database, DBMS, RDBMS](./chapter-01-data-database-dbms-rdbms.md)

**Series:** [Ch 01](./chapter-01-data-database-dbms-rdbms.md) · [Ch 02](./chapter-02-file-systems-inadequacy.md) · [Ch 03](./chapter-03-dbms-architecture.md) · [Ch 04](./chapter-04-relational-model-codd.md)

---

## Setting the scene: Apollo Multispeciality Hospital (file-based era)

Imagine **Apollo Multispeciality Hospital** (fictional teaching example) before a central DBMS. Each department runs its own programs and owns its own files on a shared NFS server:

| Department | File(s) | Application |
|------------|---------|-------------|
| Registration | `patients_master.csv` | Desk app A |
| OPD | `opd_visits_2026.log` | App B |
| Lab | `lab_orders/` (one file per test) | App C |
| Pharmacy | `dispense_records.dat` | App D |
| Billing | `invoices.txt` | App E |
| Ward nursing | `ward_3_beds.json` | App F |

Patient **P-1042** (Mrs. Meera Sharma, DOB 1978-04-12, blood group B+, allergy: Penicillin) appears in **multiple files** with **different field names** and **no central enforcement**.

The rest of this chapter walks through **six classic problems** of such a file-based system, illustrates each with Meera’s case, and shows **exactly** what a DBMS does instead.

---

## Overview: six failures of the file-system approach

```text
File-based hospital IS
        │
        ├── 1. Data redundancy      (same fact stored many times)
        ├── 2. Data inconsistency   (copies disagree)
        ├── 3. Data isolation       (formats & programs incompatible)
        ├── 4. Atomicity failures   (partial updates after crash/error)
        ├── 5. Concurrent anomalies (lost update, dirty read, …)
        └── 6. Security issues      (no fine-grained, auditable control)
```

A **DBMS** is not “a fancier folder.” It is a **control layer** that owns concurrency, recovery, integrity, and authorization so applications do not reimplement them per file.

---

## 1. Data redundancy

### Definition

**Data redundancy** is the unnecessary repetition of the same data in multiple places. Redundancy wastes storage, increases update cost, and is the root cause of many inconsistency bugs.

### Hospital example (concrete)

Mrs. Meera Sharma (P-1042) is stored as:

**`patients_master.csv`**
```text
P-1042,Meera Sharma,1978-04-12,B+,Penicillin,9876500100,12 Rose Lane
```

**`opd_visits_2026.log`** (every visit repeats demographics)
```text
2026-05-10|P-1042|Meera Sharma|1978-04-12|Cardiology|Dr Rao|...
2026-05-18|P-1042|Meera Sharma|1978-04-12|Lab follow-up|...
```

**`lab_orders/L-8891.json`**
```json
{ "patient_id": "P-1042", "name": "Meera Sharma", "dob": "1978-04-12", "test": "HbA1c" }
```

**`dispense_records.dat`** again copies name, allergy, phone for labeling.

If Meera has **40 OPD visits**, **15 lab files**, and **200 pharmacy lines**, her name, DOB, and allergy may be written **hundreds of times**.

**Quantified pain:**

- Storage: ~2 KB × 250 copies ≈ 500 KB for one patient’s repeated demographics alone (scales to TBs hospital-wide).
- Change address once → hunt every file format that embedded address (if anyone remembered to store it).

### How a DBMS solves it

| Mechanism | What it does for Apollo |
|-----------|-------------------------|
| **Normalization** | `patients` table holds demographics **once**; `visits`, `lab_orders`, `prescriptions` store only `patient_id` (FK). |
| **Single logical schema** | All apps read patient facts via join or view, not by copying into local files. |
| **Controlled redundancy (optional)** | Materialized views / denormalization for performance — **declared and maintained** by the DBMS, not accidental. |

```sql
-- Demographics stored once
CREATE TABLE patients (
  id           VARCHAR(10) PRIMARY KEY,
  full_name    VARCHAR(100) NOT NULL,
  dob          DATE NOT NULL,
  blood_group  CHAR(3),
  allergy_note TEXT
);

-- Visit references patient; no repeated name/dob
CREATE TABLE opd_visits (
  visit_id    SERIAL PRIMARY KEY,
  patient_id  VARCHAR(10) NOT NULL REFERENCES patients(id),
  visit_date  DATE NOT NULL,
  department  VARCHAR(50) NOT NULL
);
```

**Principle:** redundancy is a **design choice** under the DBMS, not an **accident** of each application’s file layout.

---

## 2. Data inconsistency

### Definition

**Data inconsistency** occurs when multiple stored representations of the same real-world fact **disagree**, so different departments or reports show different “truths.”

Inconsistency often **follows** redundancy: update one copy, forget another.

### Hospital example (concrete)

On **2026-05-20**, Meera’s allergy is upgraded in Registration after a reaction:

- `patients_master.csv` → `Penicillin,Sulfa drugs`
- OPD clerk still has old rows in `opd_visits_2026.log` → `Penicillin` only
- Pharmacy `dispense_records.dat` from May 19 still says `Penicillin` — nurse uses that for labeling
- Lab JSON files unchanged

**2026-05-21 — dangerous scenario:**

1. Doctor in OPD record (stale log) sees: allergy = Penicillin only → prescribes amoxicillin (penicillin family) — **contraindicated**.
2. Registration (updated CSV) shows Sulfa — **correct latest**.
3. Billing prints invoice with phone `9876500100` but Registration changed phone to `9876500199` yesterday — **SMS payment link fails**.

Two clinicians looking at “the same patient” see **different allergies**. That is not a user error; it is an **architecture failure**.

### How a DBMS solves it

| Mechanism | Effect |
|-----------|--------|
| **Single source of truth** | One row in `patients`; all modules query it. |
| **Integrity constraints** | `NOT NULL`, `CHECK`, domain types (e.g. `blood_group IN ('A+','B+',...)`). |
| **Transactions** | `UPDATE patients SET allergy_note = ... WHERE id = 'P-1042'` commits atomically; all subsequent reads see new value (under correct isolation). |
| **Triggers / audit** | Optional `patient_audit` log who changed allergy and when — for medico-legal trace. |

```sql
UPDATE patients
SET allergy_note = 'Penicillin; Sulfa drugs',
    phone        = '9876500199'
WHERE id = 'P-1042';
-- After COMMIT, every app using DB sees the same values
```

**Principle:** consistency is **logical** (constraints + one canonical store), not “hope every team re-copied the CSV.”

---

## 3. Data isolation (program–data dependence)

### Definition

**Data isolation** (in the classic DBMS textbook sense) means each application is **tied to a particular physical/logical file structure**. Changing file layout forces **rewriting all programs** that use that file. Related term: **program–data dependence**.

Secondary effect: **data are isolated between departments** — no shared vocabulary (PatientID vs `pid` vs `MRN`).

### Hospital example (concrete)

| App | Patient key field | Date format | Allergy encoding |
|-----|-------------------|-------------|------------------|
| Registration | `P-1042` | `YYYY-MM-DD` | plain text |
| Lab | `patient_id` | ISO string | not stored |
| Ward JSON | `mrn` | Unix timestamp | `allergies: ["penicillin"]` array |
| Billing | fixed-width, cols 1–10 | `DD/MM/YYYY` | not present |

**Change request:** Add mandatory field `abha_id` (India’s health ID) for every patient.

- Registration: edit CSV header → rewrite import/export.
- Lab: change JSON schema → migrate 50,000 files.
- Ward: different JSON shape → mobile app update.
- Billing: recalculate record width → **recompile** COBOL-style billing program.

**Six months later:** IT wants to merge `opd_visits_2026.log` into a single `visits` table with indexing. Every report script that `grep`’s pipe-delimited logs breaks.

Meera’s lab result cannot be linked to ward bed assignment because **Lab uses `L-8891`** and **Ward uses visit number from a different namespace** — integration requires brittle ad hoc scripts.

### How a DBMS solves it

| Mechanism | Effect |
|-----------|--------|
| **Three-level architecture (preview)** | External views for each app; logical schema stable; physical storage can move (index, partition) without app rewrites. |
| **Catalog / data dictionary** | System tables document column names, types — not scattered README per folder. |
| **SQL / API abstraction** | Apps say `SELECT allergy_note FROM patients WHERE id = ?`; not “byte offset 240 in `.dat`”. |
| **Logical independence** | Add `abha_id` column; old apps use view without that column until upgraded. |

```sql
-- Ward app view: only fields it needs, stable names
CREATE VIEW ward_patient_summary AS
SELECT id AS mrn, full_name, dob, allergy_note
FROM patients;

-- Add column without breaking legacy apps immediately
ALTER TABLE patients ADD COLUMN abha_id VARCHAR(20);
```

**Principle:** applications depend on **logical names and constraints**, not on **file byte layout**.

---

## 4. Atomicity failures

### Definition

**Atomicity** means a transaction’s effects are **all-or-nothing**: either every sub-step succeeds and is durably committed, or none of them remain (rollback after failure).

**Atomicity failure** in file systems: a multi-step business operation **partially completes** (crash, power loss, bug), leaving data in an impossible state.

### Hospital example (concrete)

**Operation:** Admit Meera to Ward 3, Bed 12 — one business action, three file updates:

1. Append assignment to `ward_3_beds.json` → Bed 12 = occupied, P-1042  
2. Append line to `admissions.log`  
3. Deduct “available beds” counter in `ward_stats.txt` (used by dashboard)

**Timeline:**

| Time | Event |
|------|--------|
| T1 | Step 1 succeeds — JSON written |
| T2 | Step 2 succeeds — log appended |
| T3 | Power failure before Step 3 |

**State after reboot:**

- Ward JSON: Bed 12 **occupied** by P-1042  
- Admissions log: Meera **admitted**  
- `ward_stats.txt`: still shows **1 free bed** (stale)  
- Emergency desk reads stats → assigns **another patient to Bed 12**  

**Second failure mode (billing):** Transfer ₹50,000 from advance deposit file, append invoice, append payment receipt. Crash after deducting deposit but before invoice → **money vanished from ledger, no invoice**.

File systems offer `fsync` on individual files, not **cross-file atomic business units**.

### How a DBMS solves it

| Mechanism | Effect |
|-----------|--------|
| **ACID transactions** | `BEGIN … COMMIT` wraps bed assignment + admission row + stats update. |
| **Write-ahead logging (WAL)** | Intent logged before pages change; crash → redo committed, undo incomplete. |
| **Rollback** | Error in step 3 → `ROLLBACK` restores prior bed status and counts. |

```sql
BEGIN;

UPDATE beds SET status = 'occupied', patient_id = 'P-1042'
WHERE ward_id = 3 AND bed_no = 12 AND status = 'free';

INSERT INTO admissions (patient_id, ward_id, bed_no, admitted_at)
VALUES ('P-1042', 3, 12, NOW());

UPDATE ward_stats SET free_beds = free_beds - 1 WHERE ward_id = 3;

COMMIT;  -- all three visible together, or none if any step fails
```

**Principle:** atomicity is defined on **business transactions**, not on **single `write()` system calls**.

---

## 5. Concurrent access anomalies

### Definition

When **multiple users or processes** access the same data at the same time without a DBMS-style scheduler, interleavings cause **anomalies**:

| Anomaly | Idea |
|---------|------|
| **Lost update** | Two writes; one overwrites the other unnoticed |
| **Dirty read** | Read uncommitted data that gets rolled back |
| **Non-repeatable read** | Same query twice; row changed in between |
| **Phantom read** | Same range query; new rows appear |

File locking (if used at all) is often **coarse, ad hoc, or forgotten**.

### Hospital example (concrete)

**Scenario A — Lost update (two nurses, one bed file)**

Ward 3 `ward_3_beds.json` for Bed 12: `status: free`

| Time | Nurse A process | Nurse B process |
|------|-----------------|-----------------|
| T1 | Read file: Bed 12 free | |
| T2 | | Read file: Bed 12 free |
| T3 | Write: Bed 12 → P-1042 (Meera) | |
| T4 | | Write: Bed 12 → P-2088 (Mr. Khan) |

**Result:** Last writer wins; **both think they assigned Bed 12**. One patient arrives to an occupied bed — operational and medico-legal risk.

**Scenario B — Pharmacy stock file (lost update on quantity)**

`stock_penicillin.txt` contains `qty=100`.

- Pharmacist A sells 30 → reads 100, writes 70  
- Pharmacist B sells 25 → reads 100 (before A’s write visible), writes 75  

**True sold:** 55. **File shows:** 75. **Physical stock:** 45. **Inventory drift** until manual count.

**Scenario C — Dirty read (billing draft)**

Billing app writes temporary `invoice_draft.tmp` with amount ₹1,20,000 while calculating. Report generator reads draft, prints summary for CFO. Calculation fails, app deletes draft — CFO saw **₹1,20,000 revenue that never existed**.

**Scenario D — Non-repeatable read (allergy update during consultation)**

Dr. Rao opens Meera’s record from `patients_master.csv` at **10:00** — allergy = `Penicillin`. Registration updates the same CSV at **10:05** to `Penicillin; Sulfa drugs` after a new reaction. Dr. Rao re-reads the **same file handle / cached copy** at **10:10** during prescription — still sees `Penicillin` only (stale read), or if the file was re-read, sees new value while her **first decision was based on old data** within one logical “session.” Without transaction isolation, one clinician’s workflow spans **inconsistent snapshots**.

**Scenario E — Phantom read (ward census report)**

Night supervisor runs a script: count lines in `admissions.log` where `ward_id=3` and `discharged_at` empty → **18 patients**. While the report is generating, a new admission for patient P-3301 is appended. Supervisor re-runs the **same script** → **19 patients**. No row was “updated” — a **new tuple appeared** in the range. Bed allocation dashboard and staffing decisions disagree within minutes.

### How a DBMS solves it

| Mechanism | Effect |
|-----------|--------|
| **Concurrency control** | Locking, MVCC (multi-version concurrency control), or optimistic validation |
| **Isolation levels** | `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE` — trade safety vs performance |
| **Row-level locks** | Two nurses updating **different beds** do not block entire `beds` table file |
| **Atomic SQL updates** | `UPDATE stock SET qty = qty - 30 WHERE drug_id = ?` is read-modify-write **inside** engine |
| **Snapshot / repeatable read** | Dr. Rao’s transaction sees one consistent allergy value for its duration (`REPEATABLE READ`) |
| **Predicate / range locking** | Prevents phantom admissions in ward census under `SERIALIZABLE` or equivalent |

```sql
-- Lost update prevented: row lock + atomic decrement
BEGIN;
UPDATE drug_stock
SET quantity = quantity - 30
WHERE drug_id = 'PEN-250' AND quantity >= 30;
-- If row changed concurrently, isolation + predicate lock handle conflict
COMMIT;
```

**Serializable schedule** for bed assignment:

```sql
SELECT * FROM beds
WHERE ward_id = 3 AND bed_no = 12 FOR UPDATE;  -- lock row first
-- then assign if still free
```

**Principle:** interleaving is **managed by a scheduler** with formal isolation guarantees, not “whoever saves last.”

---

## 6. Security issues

### Definition

In file-based systems, security is usually **OS-level** (read/write permission on files/directories). That is insufficient for databases because:

- Many users need **partial** access (see allergy, not salary of staff)  
- **Role-based** rules differ per application  
- **Audit** of who read which patient record is required (HIPAA-like / India DPDP context)  
- **Backup copies** and **copies emailed between departments** multiply leak surface  

### Hospital example (concrete)

**Unix permissions:** `chmod 660` on `patients_master.csv` for group `hospital_staff`.

| Problem | Concrete incident |
|---------|-------------------|
| **Too coarse** | Junior billing intern has read access to entire CSV — sees **HIV status, psychiatric notes** in same file as phone number |
| **No row-level control** | Meera’s record readable because intern can open **all** patients |
| **Copy leakage** | Registration emails `patients_master.csv` snapshot to OPD weekly — now in **12 mailboxes + laptops** |
| **No audit trail** | Someone changes allergy field in file; **no log** of who/when — dispute after adverse drug event |
| **Tampering** | Attacker with write on NFS replaces `lab_orders/L-8891.json` HbA1c from `6.2` to `4.1` → wrong treatment |
| **Repudiation** | Doctor denies changing discharge summary in shared Word-on-NFS workflow |

### How a DBMS solves it

| Mechanism | Effect |
|-----------|--------|
| **Authentication** | DB user / SSO identity, not shared UNIX password |
| **Authorization (GRANT/REVOKE)** | `GRANT SELECT (id, full_name, dob) ON patients TO billing_role` |
| **Row-level security (RLS)** | Nurse sees only patients in assigned ward |
| **Views** | `CREATE VIEW opd_patient_basic AS SELECT id, full_name, dob FROM patients` — hide psychiatric columns |
| **Audit logging** | `AUDIT` or triggers → `patient_access_log` |
| **Encryption** | TDE at rest, TLS in transit — beyond file permission bits |
| **Backup policy** | Centralized backup with same access rules, not 50 USB copies |

```sql
-- Role-based column restriction (conceptual)
GRANT SELECT (id, full_name, visit_date) ON opd_visits TO intern_role;
REVOKE SELECT ON patients FROM intern_role;

-- Optional: row-level policy (PostgreSQL-style sketch)
CREATE POLICY nurse_ward_patients ON admissions
  FOR SELECT TO nurse_role
  USING (ward_id IN (SELECT ward_id FROM nurse_assignments WHERE user = current_user));
```

**Principle:** security matches **organizational roles and legal minimum-necessary access**, not **“can you open the folder?”**

---

## 7. Master comparison table

| Problem | File system @ Apollo | DBMS response |
|---------|----------------------|---------------|
| **Redundancy** | Meera’s demographics in CSV, logs, JSON, `.dat` | Normalized tables + FK; optional controlled denorm |
| **Inconsistency** | Allergy updated in CSV, not in pharmacy file | Single row + transactional update + constraints |
| **Isolation** | Six formats for patient ID; grep scripts | Catalog, SQL, views, logical independence |
| **Atomicity** | Bed assigned in JSON but stats not decremented | `BEGIN/COMMIT`, WAL, rollback |
| **Concurrency** | Two nurses assign same bed; stock qty wrong | Locks, MVCC, isolation levels, atomic `UPDATE` |
| **Security** | Group-readable CSV leaks all fields | Roles, grants, RLS, audit, encryption |

```text
                    FILE SYSTEM                    DBMS
                    ───────────                    ────
Control location    Each application               Central engine
Correctness         Programmer discipline          Constraints + transactions
Concurrency         Ad hoc / none                  Scheduler + isolation
Recovery            Restore files from backup      Point-in-time + WAL redo/undo
Security            File permissions               AuthZ + audit + optional RLS
```

---

## 8. Interview questions (with detailed answers)

### Q1. “Data redundancy is bad, so should we eliminate all redundancy in a database?”

**Deep answer:**

**Accidental redundancy** (copy-paste across files) is bad: update anomalies, inconsistency, wasted space — as in Meera’s case across OPD logs and lab JSON.

**Intentional redundancy** can be justified:

1. **Performance** — denormalized `visit_summary` table avoids heavy joins for a dashboard.  
2. **Snapshots / warehousing** — historical report frozen at month-end.  
3. **Derived attributes** — `total_due` cached if recomputing from line items is expensive (with triggers to keep in sync).

A DBMS lets you declare redundant structures **explicitly** (materialized view, trigger-maintained column) and keep them **consistent**. File systems give you accidental redundancy with **no reconciliation mechanism**.

**Exam sound bite:** Normalize logically for OLTP integrity; denormalize **controlled** for read performance, never by duplicating into six unlinked files.

---

### Q2. “Our hospital uses Excel on a shared drive. Isn’t that a database? Why migrate?”

**Deep answer:**

Excel on a share is a **collaborative spreadsheet**, not a DBMS.

| Requirement | Shared Excel | DBMS |
|-------------|--------------|------|
| Concurrent writers | File lock / last-save-wins | Row-level locking, MVCC |
| Integrity | Weak FK across workbooks | `REFERENCES`, `CHECK`, triggers |
| Atomic multi-sheet update | No true cross-sheet transaction | SQL `BEGIN/COMMIT` |
| Audit | Limited, tamperable | Server audit log |
| Scale | Row limits, slow full load | Indexes, query optimizer |
| Security | File share ACL | Role, column, row policies |

For **low-volume club lists**, Excel is fine. For **bed assignment + allergies + billing** with legal liability, partial updates and concurrent saves are **predictable failure modes** — exactly Sections 4–5.

**Mature line:** “Excel is a UI; a DBMS is a **contract** on concurrent, recoverable, authorized access.”

---

### Q3. “Explain lost update and how a DBMS prevents it. Is file locking enough?”

**Deep answer:**

**Lost update:** Two transactions read the same value \(V\), compute new values \(V_A\) and \(V_B\) from \(V\), write sequentially; **one update is silently lost** because writes were not based on the latest value.

Hospital: two pharmacists read `qty=100`, write 70 and 75; true result should be 45.

**File locking “enough”?**

- **Exclusive lock on entire file** — prevents lost update on one counter but **kills throughput** (entire pharmacy file locked per sale).  
- **Lock forgotten** — bug → anomaly returns.  
- **No atomic read-modify-write** — must lock, read, compute, write, unlock in **user code**; crash mid-way → inconsistency.  
- **Multiple files** — bed JSON + stats txt need **distributed** lock protocol across apps — rarely correct.

**DBMS approach:**

```sql
UPDATE drug_stock SET quantity = quantity - :n WHERE drug_id = :id;
```

Executed under **row lock**; isolation level ensures serializable or acceptable schedules; **undo/redo** on crash.

Optional: `UPDATE ... WHERE quantity = :expected` (optimistic concurrency) — 0 rows updated → retry.

**Sound bite:** File locking can serialize access to **one file**; a DBMS provides **transactional, multi-row, recoverable** schedules with a defined isolation contract.

---

## 9. Exercises (attempt before hints)

### Exercise 1 — Map failures to fixes

Apollo adds an **ICU vitals monitor** that appends one line per second to `icu_vitals_P-1042.csv`. Doctors query a **Python script** that scans the whole file for max heart rate in the last hour. Billing still keeps a copy of `patient_name` in every invoice line.

For **each** of the six problems in this chapter:

1. State whether it **appears** in this new setup (yes/no/partial).  
2. Name **one DBMS feature** (not “use PostgreSQL” generically) that addresses it — e.g. “MVCC”, “FOREIGN KEY”, “WAL”.  
3. Propose **one table** (name + key columns only) that would replace the worst file in this vignette.

<details>
<summary>Hint (after you try)</summary>

Vitals → concurrency + isolation + maybe partitioning; scanning whole file → indexing on `(patient_id, ts)`; invoice name copy → redundancy/FK to `patients`.
</details>

---

### Exercise 2 — Design a transaction

Write pseudocode or SQL for **one atomic business transaction** at Apollo:

**“Discharge patient P-1042 from Ward 3 Bed 12”** must:

- Free the bed  
- Set admission row `discharged_at`  
- Increment `ward_stats.free_beds`  
- Insert billing line for final ward day charge  

1. List **four failure points** if implemented as four separate file writes.  
2. Write a single **`BEGIN … COMMIT`** block (real SQL preferred).  
3. What should happen if billing insert fails — and how does the DBMS enforce that?

<details>
<summary>Hint (after you try)</summary>

Failure points: partial bed free, double free, orphan admission, wrong stats. On billing failure: `ROLLBACK` — bed still occupied, admission open. Enforcement: transaction manager + log.
</details>

---

## 10. Chapter summary

File systems are excellent at **storing named blobs of bytes**. They are **not** a substitute for **shared, correct, concurrent, secure data management** across many applications.

| # | Problem | One-line hospital lesson |
|---|---------|---------------------------|
| 1 | Redundancy | Meera’s allergy copied 250× |
| 2 | Inconsistency | Registration says Sulfa; pharmacy file says Penicillin only |
| 3 | Isolation | `P-1042` vs `mrn` vs grep scripts — one schema change breaks six apps |
| 4 | Atomicity | Bed occupied in JSON, stats say bed free |
| 5 | Concurrency | Two nurses, one bed; two pharmacists, wrong stock; dirty/non-repeatable/phantom reads |
| 6 | Security | Intern reads entire CSV; no audit after allergy change |

The DBMS exists so these are not solved **again, differently, in every department’s code**.

---

*Prev: [Chapter 01](./chapter-01-data-database-dbms-rdbms.md) · Next: [Chapter 03 — DBMS architecture](./chapter-03-dbms-architecture.md)*
