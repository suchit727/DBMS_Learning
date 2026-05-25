# Chapter 07 — Cardinality & Participation Constraints

*Senior DBMS lecture notes — IIT-style treatment*

**Prerequisites:**

- [Chapter 05 — ER modeling](./chapter-05-er-modeling-university.md)
- [Chapter 06 — Drawing ER diagrams](./chapter-06-er-diagrams-notation.md)

**Data modeling track:** [Ch 05](./chapter-05-er-modeling-university.md) · [Ch 06](./chapter-06-er-diagrams-notation.md) · [Ch 07](./chapter-07-cardinality-participation.md) · [Ch 08](./chapter-08-er-to-relational-mapping.md)

---

## Two dimensions — do not merge them

Every binary relationship in an ER diagram needs **two independent answers**:

| Dimension | Question | Example (ConnectHub) |
|-----------|----------|----------------------|
| **Cardinality (ratio)** | How many instances of **B** can one **A** relate to? | One user → **many** posts |
| **Participation** | Must **every** instance of **A** participate in **at least one** link? | Not every user must post → USER **partial** |

**Exam trap:** “Total participation” does **not** mean “exactly one.” It means “**at least one**” (mandatory membership in the relationship set).

**Notation recap (Chapter 06):**

- Chen: `1`, `N`, `M` on edges; **double line** = total participation  
- Crow’s foot / UML: `0..1`, `1..1`, `0..*`, `1..*` on each end  

---

## Part A — Cardinality ratios (structural)

### 1. One-to-one (1:1)

**Definition:** Each entity on side A relates to **at most one** entity on side B, and vice versa.

**Real examples:**

| Domain | Relationship | Business rule |
|--------|--------------|---------------|
| University | STUDENT — HOSTEL_ROOM | Each student at most one room; each room at most one student (single occupancy) |
| HR | EMPLOYEE — PARKING_PASS | One pass per employee; one employee per pass |
| Passport | CITIZEN — PASSPORT (active) | One valid passport per citizen in this model |

**ER sketch (Chen):**

```text
STUDENT ──1──◇ Occupies ◇──1── HOSTEL_ROOM
```

**Relational mapping:** FK on either side (choose where “many” side is absent — either table works). Put `room_id` in `student` **or** `student_roll` in `room` — pick the side that is **total** in participation if asymmetric.

**1:1 with optional side (0..1 : 1):** Student **may** have no room (commuter) → student side partial, room side total if every room must have someone (or partial if vacant rooms allowed).

---

### 2. One-to-many (1:N)

**Definition:** One entity on the “one” side relates to **many** on the “many” side; each “many” instance relates to **exactly one** “one” (in the usual directed reading).

**Real examples:**

| Domain | One | Many | Rule |
|--------|-----|------|------|
| E-commerce | CUSTOMER | ORDER | One customer, many orders; each order one customer |
| University | DEPARTMENT | PROFESSOR | Dept has many faculty; each professor primary dept one |
| Social | USER | POST | Author issues many posts; each post one author |

**ER sketch:**

```text
CUSTOMER ──1──◇ Places ◇──N── ORDER
(double line on ORDER side if every order must have a customer)
```

**Relational mapping:** FK on **many** side: `orders.customer_id NOT NULL`.

---

### 3. Many-to-many (M:N)

**Definition:** One A may link to many B; one B may link to many A.

**Real examples:**

| Domain | Entities | Relationship |
|--------|----------|--------------|
| University | STUDENT — COURSE | Enrollment |
| Retail | PRODUCT — SUPPLIER | Supplies |
| Social | USER — USER | Follows (directed: still M:N out-degree / in-degree) |

**ER sketch:**

```text
STUDENT ──M──◇ Enrolls ◇──N── COURSE
```

**Relational mapping:** **Junction table** `enrollment(student_id, course_id, grade, …)` with composite PK or surrogate + UNIQUE constraint.

**You cannot** implement M:N with a single FK on one table without **violating** 1NF or duplicating rows.

---

### Comparison table

| Ratio | Max on A→B | Max on B→A | Junction table? |
|-------|------------|------------|-----------------|
| 1:1 | 1 | 1 | No (FK one side) |
| 1:N | N (or 1) | 1 | No (FK on many side) |
| M:N | M | N | **Yes** |

---

## Part B — How to identify cardinality from a business rule

Use this **four-step recipe** (say it in viva):

### Step 1 — Name the relationship in active voice

“**Student enrolls in course section**” → binary `ENROLLS` between `STUDENT` and `SECTION`.

### Step 2 — Ask two directed questions

| Question | Answer for CampusDB enroll |
|----------|---------------------------|
| Q1: One student — how many sections **at once** in this model? | Many (full timetable) → **N** from STUDENT |
| Q2: One section — how many students? | Many → **M** from SECTION |

If both answers are “many” → **M:N**.

### Step 3 — Check “at most one” phrases

| Phrase in requirement | Cardinality hint |
|-----------------------|------------------|
| “exactly one”, “only one”, “unique” | **1** on that side |
| “at most one”, “zero or one”, “optional one” | **0..1** |
| “many”, “several”, “list of”, “history of” | **N** or **\*** |
| “each X belongs to one Y” | FK side is **many** toward Y |

### Step 4 — Separate participation (next section)

“Every order must have a customer” → **total** participation of ORDER in `Places`, not 1:N by itself.

### Worked micro-example

**Rule:** “A department offers many courses; each course is offered by exactly one department.”

- Q1: One department → many courses → **1 : N**  
- Q2: One course → one department → **1** on course side  
- Participation: If every course must have a department → COURSE **total** in `Offered_By`. If a department may exist before any course is listed → DEPARTMENT **partial**.

---

## Part C — Participation constraints (total vs partial)

### Definitions

| Constraint | Formal | Business language |
|------------|--------|-------------------|
| **Total** (mandatory) | Every entity instance in set E appears in **≥ 1** relationship instance | “No orphans allowed on this side” |
| **Partial** (optional) | ∃ entity in E that appears in **0** relationship instances | “Allowed to exist without joining” |

### Business-term examples

| Scenario | Side | Participation | Why |
|----------|------|---------------|-----|
| Every post has an author | POST in `Posts` | **Total** | Orphan post is meaningless |
| User may never post | USER in `Posts` | **Partial** | Lurker / new signup |
| Every payment belongs to an order | PAYMENT in `Pays_For` | **Total** | No floating payment |
| Order may exist before payment | ORDER in `Pays_For` | **Partial** | Cart not paid yet |
| Every section belongs to a course | SECTION (weak) in `Offered_As` | **Total** | Section cannot float |
| Course may have no sections this semester | COURSE in `Offered_As` | **Partial** | Not offered this term |

### Visual (Chen)

- **Total:** double line from entity rectangle to diamond  
- **Partial:** single line  

### Mapping to SQL

| Participation | Typical SQL |
|---------------|-------------|
| Total on many side of 1:N | `FK NOT NULL` |
| Partial on many side | `FK NULL` allowed |
| Total on one side of 1:1 | `FK NOT NULL` on FK-holding table |
| M:N total on both | Junction row required for each? — usually **partial** on entities (student may not enroll yet) |

---

## Part D — Wrong cardinality → real bugs

Cardinality mistakes are not “diagram errors” — they become **schema bugs** in production.

### Bug 1 — Modeled 1:1 but reality is 1:N

**Wrong model:** `CUSTOMER` —1— `ORDER` (one order per customer lifetime).

**Schema:** `orders.customer_id UNIQUE` or only one row per customer.

**Production failure:**

- Returning customer places second order → **UNIQUE violation** or silent overwrite.  
- E-commerce loses order history; revenue reports wrong.

**Fix:** `CUSTOMER` 1 — `ORDER` N; drop UNIQUE on `customer_id` in orders (keep FK).

---

### Bug 2 — Modeled 1:N but reality is M:N

**Wrong model:** `STUDENT` — N — `COURSE` with FK `course_id` on `student` (each student one course).

**Production failure:**

- Student Arjun takes DBMS **and** OS → second course needs **second row** duplicating student demographics or illegal second `student` row.  
- Timetable and GPA logic break.

**Fix:** Junction `enrollment(student_id, course_id)` — M:N.

---

### Bug 3 — Modeled M:N but reality is 1:N

**Wrong model:** Junction table for `EMPLOYEE` — `DEPARTMENT` when rule is “each employee has exactly one primary department.”

**Production failure:**

- Duplicate `(emp_id, dept_A)` and `(emp_id, dept_B)` both “primary” — app must guess; payroll tax jurisdiction wrong.  
- Extra join on every HR query.

**Fix:** `employee.dept_id FK NOT NULL` — simple 1:N.

---

### Bug 4 — Confused participation with cardinality

**Wrong reading:** “Every post has one author” → draw **1:1** between USER and POST.

**Reality:** One author, **many** posts → **1:N** (USER one, POST many), POST **total** in `Posts`.

**Production failure:** If implemented as 1:1, second post by same user rejected.

---

### Bug 5 — Partial vs total ignored (nullable FK when business forbids orphan)

**Wrong model:** `post.author_id` **NULL** allowed (partial on wrong side).

**Production failure:**

- API bug inserts post without author → **orphan posts** in feed, moderation crashes.  
- Analytics: “unknown author” bucket explodes.

**Fix:** POST **total** participation → `author_id NOT NULL` + app validation.

---

### Summary — bug matrix

| Mistake | Symptom in DB/app |
|---------|-------------------|
| 1:N modeled as 1:1 | Second child insert fails |
| M:N modeled as 1:N | Duplicate entity rows or lost associations |
| 1:N modeled as M:N | Redundant junction; duplicate links |
| 1:1 vs 1:N confused | Power users blocked |
| Total/participation wrong | Orphan or NULL FK rows |

---

## Part E — Five business scenarios (full analysis)

For each: **business rule → cardinality → participation → relational hint**.

---

### Scenario 1 — Food delivery (`QuickBite`)

**Rule:** A customer can place many orders over time. Each order is placed by exactly one customer. An order must exist with a customer; a newly registered customer may have placed no orders yet.

| Analysis | Value |
|----------|-------|
| **Cardinality** | CUSTOMER **1** : **N** ORDER |
| **Participation** | CUSTOMER **partial** in `Places`; ORDER **total** in `Places` |
| **Notation** | CUSTOMER `0..*` — ORDER `1..1` (from order’s perspective toward customer) |
| **SQL** | `orders.customer_id BIGINT NOT NULL REFERENCES customers` |

**Wrong choice:** 1:1 → repeat customers break. M:N → unnecessary `order_customer` junction.

---

### Scenario 2 — Hospital appointment (`Apollo` style)

**Rule:** A doctor can have many appointments per day. Each appointment is with exactly one doctor. An appointment cannot exist without a doctor; a doctor on leave may have zero appointments that day.

| Analysis | Value |
|----------|-------|
| **Cardinality** | DOCTOR **1** : **N** APPOINTMENT |
| **Participation** | DOCTOR **partial**; APPOINTMENT **total** |
| **SQL** | `appointments.doctor_id NOT NULL` |

**Wrong choice:** M:N (doctor–appointment junction) — overkill unless appointment has **multiple attending** physicians (then ternary or M:N with role).

---

### Scenario 3 — University enrollment (`CampusDB`)

**Rule:** A student may enroll in many course sections per semester. Each enrollment links one student to one section. A student may be registered but not yet enrolled in any section (fresh admit). A section may exist with zero students (cancelled if under-enrolled).

| Analysis | Value |
|----------|-------|
| **Cardinality** | STUDENT **M** : **N** SECTION (via relationship **ENROLLS**) |
| **Participation** | STUDENT **partial**; SECTION **partial** (both optional until event occurs) |
| **SQL** | `enrollment(roll_no, section_id, semester, grade, PK(roll_no, section_id, semester))` |

**Wrong choice:** FK `section_id` on `student` only → one section per student.

---

### Scenario 4 — Company car parking

**Rule:** Each employee may be assigned at most one parking slot. Each slot is assigned to at most one employee. Some employees have no slot (remote workers). Some slots are empty (reserved visitor slots).

| Analysis | Value |
|----------|-------|
| **Cardinality** | EMPLOYEE **0..1** : **0..1** PARKING_SLOT (symmetric optional 1:1) |
| **Participation** | Both **partial** |
| **SQL** | `parking_slots.employee_id UNIQUE NULL` or `employees.slot_id UNIQUE NULL` — one FK, nullable both sides |

**Wrong choice:** 1:N (one slot many employees) → collision at gate. M:N → two employees same slot without 1:1 constraint.

---

### Scenario 5 — Social media follow (`ConnectHub`)

**Rule:** A user can follow many users and be followed by many users. A user need not follow anyone. Following is directed (A follows B does not imply B follows A).

| Analysis | Value |
|----------|-------|
| **Cardinality** | USER **M** : **N** USER (recursive / unary relationship **Follows**) |
| **Participation** | USER **partial** on both roles (follower and followee) |
| **SQL** | `follows(follower_id, followee_id, followed_at, PK(follower_id, followee_id))` |

**Wrong choice:** 1:N “user has many followers” only — cannot model mutual follow or user following many. 1:1 — absurd for Twitter-scale product.

---

### Scenarios at a glance

| # | Domain | Ratio | Participation highlight |
|---|--------|-------|-------------------------|
| 1 | QuickBite orders | 1:N | ORDER total toward customer |
| 2 | Doctor appointments | 1:N | APPOINTMENT total toward doctor |
| 3 | Student–section | M:N | Both partial |
| 4 | Employee–parking | 0..1 : 0..1 | Both partial |
| 5 | User follows | M:N (directed) | Both partial |

---

## Part F — Interview questions (with deep answers)

### Q1. “Can a relationship be 1:N from A to B and N:1 from B to A? Is that M:N?”

**Deep answer:**

From **A’s perspective:** one A, many B → **1:N**.  
From **B’s perspective:** one B, one A → **N:1** (many B’s share one A).

That is **the same binary relationship**, not M:N. **M:N** requires: ∃ instances where one A links to **≥2** B **and** one B links to **≥2** A.

**CampusDB:** Department–Professor primary affiliation is **1:N** (one dept, many profs; each prof one primary dept). Not M:N unless a professor can have **two primary departments** simultaneously (unlikely — that would be M:N or a ternary with role).

**Viva diagram:** Draw one department, three professors, each professor edge only to that one department → 1:N.

---

### Q2. “Every employee must belong to a department’ — total participation on which side? What SQL?”

**Deep answer:**

**Total participation** on **EMPLOYEE** in `Works_In` (every employee must appear in at least one `Works_In` tuple).

If each employee has **exactly one** department at a time, cardinality is **N:1** from employee to department (many employees, one dept each) → FK `employee.dept_id NOT NULL`.

If employees could belong to **multiple** departments, M:N junction with **total participation on EMPLOYEE** means every employee has ≥1 row in junction — still `NOT NULL` on `emp_id` in junction, but no single `dept_id` column suffices.

**Trap:** Total on DEPARTMENT would mean “every department has at least one employee” — **different rule** (no empty departments). Read English carefully.

---

### Q3. “Why not store M:N enrollment by adding multiple nullable FK columns course1, course2, …?”

**Deep answer:**

That is a **repeating group** — violates **1NF**, fixed arbitrary limit (what if 11th course?), hard to query (“all students in DBMS”), wastes space, breaks integrity (course7 FK orphan).

**Correct:** `enrollment(student_id, course_id)` with M:N cardinality in ER.

**Real bug:** Report “count enrollments per student” becomes `CASE WHEN course1 IS NOT NULL THEN 1 ELSE 0 END + …` instead of `COUNT(*)` from junction.

**When nullable FKs appear:** Rare 1:N optional relationship (student optional advisor `advisor_id NULL`) — not M:N in disguise.

---

## Part G — Exercises

### Exercise 1 — Cardinality from English rules

For each rule, state **1:1, 1:N, or M:N**, **participation on each side**, and **where the FK or junction goes**.

1. A blog **post** has exactly one **author** (user). A user may publish zero or many posts.  
2. A **product** may appear in many **orders**; an **order** contains many **products** (with quantity).  
3. A **country** has exactly one **capital city**; a city is capital of at most one country. Some countries change capital (model current snapshot only).  
4. A **train** has many **coaches**; each coach belongs to exactly one train. A train cannot run with zero coaches.  
5. A **mentor** can mentor many **interns**; an intern has exactly one mentor at a time; every intern must have a mentor.

<details>
<summary>Hint (after you try)</summary>

1. 1:N USER:POST, POST total. 2. M:N via order_line. 3. 1:1 or 0..1:1 COUNTRY:CITY. 4. 1:N TRAIN:COACH, COACH total. 5. 1:N MENTOR:INTERN, INTERN total.
</details>

---

### Exercise 2 — Find the bug (schema + cardinality)

Each snippet encodes a **wrong** cardinality or participation choice. Explain the **production bug** and redraw the correct ER constraint.

1. `CREATE TABLE users (id PK, current_post_id FK UNIQUE REFERENCES posts(id));` — “users and posts like Instagram.”  
2. `CREATE TABLE enrollments (student_id FK, course_id FK);` plus `ALTER TABLE students ADD course_id FK;` — only one column on student.  
3. `CREATE TABLE orders (id PK, customer_id FK NULL);` — business rule: “every order must identify the buyer.”  
4. `CREATE TABLE marriages (husband_id FK, wife_id FK);` plus `UNIQUE(husband_id)` and `UNIQUE(wife_id)` — rule: “a person may marry many times over life (serial monogamy model).”

<details>
<summary>Hint (after you try)</summary>

1. 1:1 not 1:N for author-posts. 2. M:N not FK on student. 3. total participation → NOT NULL. 4. 1:N history or M:N with date, not dual UNIQUE 1:1.
</details>

---

## Part H — Chapter summary

| Idea | Takeaway |
|------|----------|
| **1:1** | At most one each way; rare; often optional on one side |
| **1:N** | FK on many side; most common |
| **M:N** | Junction table; both sides can repeat |
| **Identify cardinality** | Two directed “how many” questions + watch “exactly one” |
| **Participation** | Must every instance join? — separate from 1 vs N |
| **Wrong cardinality** | UNIQUE violations, orphans, duplicate people rows, lost history |

Get cardinality and participation from **stated business rules**, not from “what feels simple in SQL.”

---

*Prev: [Chapter 06 — Drawing ER diagrams](./chapter-06-er-diagrams-notation.md) · Next: [Chapter 08 — ER → relational mapping](./chapter-08-er-to-relational-mapping.md)*
