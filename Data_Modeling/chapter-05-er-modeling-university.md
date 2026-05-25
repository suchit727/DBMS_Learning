# Chapter 05 — Entity-Relationship (ER) Modeling

*Senior DBMS lecture notes — IIT-style treatment*

**Prerequisites:**

- [Chapter 01 — Data, Database, DBMS, RDBMS](../Foundation/chapter-01-data-database-dbms-rdbms.md)
- [Chapter 04 — Relational model (Codd)](../Foundation/chapter-04-relational-model-codd.md)

**Series (Foundation):** [Ch 01](../Foundation/chapter-01-data-database-dbms-rdbms.md) · [Ch 02](../Foundation/chapter-02-file-systems-inadequacy.md) · [Ch 03](../Foundation/chapter-03-dbms-architecture.md) · [Ch 04](../Foundation/chapter-04-relational-model-codd.md)

**Data modeling track:** [Ch 05](./chapter-05-er-modeling-university.md) · [Ch 06 — Drawing ER diagrams](./chapter-06-er-diagrams-notation.md)

---

## Why ER modeling exists

Before you write `CREATE TABLE`, you must answer: **what things exist in the real world, how they connect, and what facts must be stored?**

The **Entity-Relationship (ER) model** (Chen, 1976; extended in textbooks as **EER**) is a **conceptual design** notation — independent of SQL — used to communicate with stakeholders and to map later to a **relational schema**.

**Running example — IIT-style university `CampusDB`:**

| Real-world focus | Examples |
|------------------|----------|
| **Students** | roll number, name, program, hostel room |
| **Courses** | course code, title, credits |
| **Professors** | employee id, name, department |
| **Teaching** | who teaches which course in which semester |
| **Enrollment** | which student takes which course, grade |

All diagrams below are described in words so you can draw them on paper or in tools (draw.io, Lucidchart, pgModeler).

---

## Part A — Entities and entity types

### Entity (instance)

An **entity** is a **distinct object in the mini-world** we are modeling — something that exists and can be uniquely identified, about which we want to store data.

- **Instance level:** one particular thing  
- **CampusDB:** the student *Arjun Mehta, roll CS21B0847* is one entity. Professor *Dr. S. Rao, emp_id P-1042* is another.

### Entity type (schema / set)

An **entity type** is a **collection of similar entities** sharing the same properties (attributes) and the same role in the organization.

- **Schema level:** the pattern / class  
- **CampusDB:** entity type **STUDENT** groups all students; **COURSE** groups all courses; **PROFESSOR** groups all faculty.

| | **Entity (instance)** | **Entity type** |
|---|----------------------|-----------------|
| **Analogy** | One specific chair in the lecture hall | The concept “Chair” |
| **CampusDB** | Arjun Mehta | STUDENT |
| **In relational mapping** | One row | One table (usually) |
| **Notation** | Lowercase in prose: “entity student Arjun” | Rectangle in ER diagram: `STUDENT` |

### Conceptual diagram description — entity type

**Draw one rectangle** labeled `STUDENT`. Inside or beside the rectangle, list attribute names (ovals in Chen notation) such as `roll_no`, `name`, `program`. Each **real student** you enroll is a **dot** you would place inside the set mentally — in practice we draw the **type** once, not every instance.

```text
┌─────────────────────┐
│      STUDENT        │
│  roll_no (key)      │
│  name               │
│  program            │
└─────────────────────┘
```

**Exam sound bite:** Entity type = intension (definition); entity = extension (member of the set).

---

## Part B — Strong vs weak entities

### Strong entity

A **strong entity** has a **key attribute** (or composite key) that uniquely identifies each instance **without depending on another entity**.

- **CampusDB:** `STUDENT(roll_no, …)` — `roll_no` alone identifies Arjun globally in the university.

### Weak entity

A **weak entity** **cannot be uniquely identified** by its own attributes alone; it depends on a **owner (strong) entity** and needs the owner’s key as part of its identifier.

- **Partial key** (discriminator): attribute(s) of the weak entity that distinguish instances **under the same owner**  
- **Identifying relationship** (double diamond in Chen notation): links weak entity to owner

**CampusDB classic weak entity — `SECTION` (or `CLASS_MEETING`):**

A course `DBMS` may have **multiple sections** in one semester: Section 1 (Mon 9am), Section 2 (Wed 2pm). Section number `1` is **not unique globally** — only unique **for course DBMS in Sem 2026-I**.

- Owner: **COURSE** (strong), key `course_code`  
- Weak: **SECTION**, partial key `sec_no`  
- Full key: `(course_code, sec_no, semester)`

### Conceptual diagram description — weak entity

**Draw:**

1. Rectangle `COURSE` with key `course_code`  
2. Rectangle `SECTION` with partial key `sec_no` (underline with **dashed** underline in Chen style for partial key)  
3. **Double diamond** `Offered_As` (identifying relationship) between `COURSE` and `SECTION`  
4. **Double rectangle** around `SECTION` (optional in some notations) to mark weak entity  

```text
┌─────────┐       ╔═══════════╗       ┌──────────────┐
│ COURSE  │───────║ Offered_As ║───────│   SECTION    │
│course_code│     ╚═══════════╝       │ sec_no (partial)│
└─────────┘   (double diamond)         │ room, slot      │
                                       └──────────────┘
                                        (double box = weak)
```

**Another CampusDB weak entity — `DEPENDENT` of `PROFESSOR`:**

Insurance may store dependents; dependent name “Anita” is not unique — only unique per professor `P-1042`.

### Comparison table

| | **Strong** | **Weak** |
|---|------------|----------|
| **Key** | Own key suffices | Owner key + partial key |
| **Exists without owner?** | Yes | No (meaningless orphan) |
| **Relationship to owner** | Ordinary | **Identifying** (double diamond) |
| **Relational mapping** | Own table with PK | Table with **composite PK** including FK to owner |

---

## Part C — Types of attributes

Attributes describe **entity types** (and relationship types). Classify each attribute carefully — wrong type → bad relational schema.

### 1. Simple (atomic) attribute

Cannot be meaningfully divided **for this application**.

- **CampusDB:** `roll_no`, `credits`, `emp_id`

### 2. Composite attribute

Can be split into **components** that still have meaning.

- **CampusDB:** `name` → `{first_name, middle_name, last_name}`  
- **CampusDB:** `address` → `{street, city, pincode}`

**Diagram:** One oval `name` connected to sub-ovals `first_name`, `last_name` (tree structure).

**Relational mapping:** Often **flatten** to columns `first_name`, `last_name` unless you embed JSON (not classical ER).

### 3. Multivalued attribute

May hold **more than one value** for a single entity instance.

- **CampusDB:** `STUDENT.phone_numbers` — home + mobile  
- **CampusDB:** `COURSE.prerequisites` — list of course codes  

**Notation:** **Double oval** around the attribute name.

**Diagram description:** Oval `phone_numbers` drawn with **double border** branching from `STUDENT`.

**Relational mapping:** Separate table `student_phone(roll_no, phone)` — never comma-separated list in one cell (violates 1NF).

### 4. Derived attribute

Computed from **other attributes** or related data — may or may not be stored.

- **CampusDB:** `STUDENT.age` derived from `date_of_birth`  
- **CampusDB:** `SECTION.enrollment_count` derived from counting enrollment records  

**Notation:** **Dashed oval** (Chen).

**Relational mapping:** Usually **do not store**; compute in view/query. If stored for performance, maintain with triggers and document as denormalization.

### 5. Key attribute

Uniquely identifies an entity instance within its entity type (for strong entities).

- **CampusDB:** `roll_no` for `STUDENT`, `course_code` for `COURSE`, `emp_id` for `PROFESSOR`

**Notation:** **Underlined** attribute name.

**Candidate key:** Minimal unique set. **Primary key:** chosen candidate key for implementation.

### Attribute summary diagram (CampusDB STUDENT)

```text
                    ┌── first_name
         ┌── name ──┼── last_name        (composite: tree of ovals)
         │          └── middle_name
STUDENT ─┼── roll_no  (single underline = key)
         ├── date_of_birth
         ├── age        (dashed oval = derived)
         └── phone_numbers (double oval = multivalued)
```

---

## Part D — Relationships

### Relationship (instance)

A **relationship** is an **association among entities** — a fact connecting two or more instances at a moment in time.

- **CampusDB:** *Arjun* **enrolled in** *DBMS section 1* in *Sem 2026-I* is one relationship instance (a fact).

### Relationship type

A **relationship type** is the **pattern** of associations among entity types.

- **CampusDB:** relationship type **ENROLLS** connects entity types `STUDENT` and `SECTION` (or `COURSE`).

| | **Relationship instance** | **Relationship type** |
|---|---------------------------|------------------------|
| **Level** | One link between specific entities | Set of all such links |
| **CampusDB** | Arjun enrolled in DBMS-sec-1 | ENROLLS between STUDENT and SECTION |
| **Notation** | (often not drawn per instance) | **Diamond** `ENROLLS` between rectangles |

### Conceptual diagram description — binary relationship

**Draw two rectangles** `STUDENT` and `COURSE`. **Diamond** `Enrolls_In` between them. Lines connect rectangle sides to the diamond. Label the diamond with the verb phrase (present tense, active voice).

```text
┌─────────┐                    ┌─────────┐
│ STUDENT │───────◇ Enrolls_In ◇───────│ COURSE  │
└─────────┘                    └─────────┘
```

**Attributes on relationships:** Enrollment may have `grade`, `enrollment_date` — attach ovals to the **diamond** `Enrolls_In`, not to one entity only.

```text
         grade
          │
STUDENT ──◇ Enrolls_In ◇── COURSE
          │
    enrollment_date
```

---

## Part E — Degree of a relationship

**Degree** = number of entity types participating in one relationship type.

### Unary (degree 1) — recursive relationship

One entity type relates **to itself**.

**CampusDB examples:**

| Relationship | Meaning |
|--------------|---------|
| **Prerequisite_Of** | Course A is prerequisite for course B |
| **Supervises** | Professor supervises another professor (research) |
| **Mentors** | Senior student mentors junior student (same STUDENT type) |

**Diagram description:** One rectangle `COURSE`. Diamond `Prerequisite_Of` attached to `COURSE` **twice** with two roles labeled `prereq` and `dependent_course` (role names mandatory when same type appears twice).

```text
          prereq
            │
        ┌───▼───┐
        │ COURSE│
        └───┬───┘
            │ dependent_course
            ▼
        (back to COURSE — same box, two edges)
```

**Draw tip:** Use **role names** on each edge: “as prerequisite” vs “as dependent course.”

---

### Binary (degree 2) — most common

Two entity types.

**CampusDB examples:**

| Relationship | Entities | Notes |
|--------------|----------|-------|
| **Teaches** | PROFESSOR, SECTION | Dr. Rao teaches DBMS sec-1 |
| **Enrolls** | STUDENT, SECTION | Arjun in DBMS sec-1 |
| **Offers** | DEPARTMENT, COURSE | CS dept offers DBMS |

**Diagram:** Standard diamond between two rectangles (see Part D).

**Cardinality** (covered in next chapter mapping; preview here):

- One professor **may teach** many sections; one section **has** many students enrolling → **1:N** and **M:N** constraints written near edges as `1`, `N`, `M`.

---

### Ternary (degree 3)

Three entity types participate; the association **cannot be decomposed** into binary relationships **without losing meaning**.

**CampusDB example — `ASSIGNS`:**

Fact: *Professor P* **assigns** *grade component rubric R* for *course section S* — all three together matter. You cannot capture “who grades which component in which section” with only `Professor–Section` and `Section–Rubric` pairs without ambiguity.

- Entities: `PROFESSOR`, `SECTION`, `GRADING_COMPONENT`  
- Relationship type: **ASSIGNS** (ternary diamond)  
- Attributes on diamond: `weight_percent`, `effective_from`

**Diagram description:**

**Three rectangles** arranged in a triangle. **One diamond** `ASSIGNS` in the center with **three lines** to each rectangle (no direction arrow in conceptual ER).

```text
    PROFESSOR
        ╲
         ╲
          ◇ ASSIGNS ◇
         ╱         ╲
        ╱           ╲
   SECTION      GRADING_COMPONENT
```

**Relational mapping:** Table with **three foreign keys** `(emp_id, section_id, component_id)` as composite key (if many-to-many-many) plus relationship attributes.

**When ternary is necessary:** The fact only makes sense with **all three** participants simultaneously (sponsor–project–researcher classic example). If two binaries suffice, prefer simpler design — but exam questions often test ternary recognition.

---

## Part F — CampusDB integrated conceptual schema (overview)

**Entity types (strong):**

- `STUDENT(roll_no, name, program, dob, …)`  
- `PROFESSOR(emp_id, name, dept, …)`  
- `COURSE(course_code, title, credits, …)`  
- `DEPARTMENT(dept_code, name, …)`

**Weak:**

- `SECTION(sec_no, room, slot, …)` owned by `COURSE` via identifying `Offered_As`

**Key relationship types:**

| Type | Degree | Connects |
|------|--------|----------|
| `Prerequisite_Of` | Unary | COURSE – COURSE |
| `Teaches` | Binary | PROFESSOR – SECTION |
| `Enrolls` | Binary | STUDENT – SECTION |
| `Works_In` | Binary | PROFESSOR – DEPARTMENT |
| `Offered_As` | Binary identifying | COURSE – SECTION (weak) |
| `ASSIGNS` | Ternary | PROFESSOR – SECTION – GRADING_COMPONENT |

**Full diagram (verbal walkthrough for exams):**

Place `DEPARTMENT` top-center. `PROFESSOR` left, `COURSE` right, connected by `Works_In` and `Offers`. Below `COURSE`, double-diamond to weak `SECTION`. `STUDENT` bottom-left connects to `SECTION` via `Enrolls` (M:N). `PROFESSOR` connects to `SECTION` via `Teaches`. Optional ternary `ASSIGNS` in center among professor, section, grading component.

---

## Part G — ER → relational mapping (preview)

You will formalize in the next chapter; memorize these defaults:

| ER construct | Relational table |
|--------------|------------------|
| Strong entity type | Table; PK = key attribute(s) |
| Weak entity type | Table; PK = owner FK + partial key |
| Multivalued attribute | Separate table + FK |
| Composite attribute | Flatten to columns (usually) |
| Derived attribute | Omit or view |
| Binary M:N relationship | Junction table with both FKs |
| Ternary relationship | Table with three FKs (often composite PK) |

**CampusDB:** `Enrolls(roll_no, course_code, sec_no, semester, grade)` with FKs to student and section keys.

---

## Part H — Interview questions (with deep answers)

### Q1. “What is the difference between an entity and an entity type? Give a university example.”

**Deep answer:**

An **entity** is a **particular** object in the universe of discourse — *this* student Arjun with roll CS21B0847. An **entity type** is the **abstraction** describing all students that share structure and semantics — the category **STUDENT**.

Database tables implement **entity types**; rows implement **entities** (instances). Confusing them leads to design errors: you might create a table named `Arjun` instead of `STUDENT`.

**Weak vs strong at type level:** `SECTION` is a weak **entity type** because its instances are identified only with owning course. Arjun is a strong **entity** of type STUDENT.

**Sound bite:** “Type = schema rectangle; entity = row in the real world.”

---

### Q2. “When do you model a weak entity vs a strong entity with a foreign key?”

**Deep answer:**

Use a **weak entity** when the dependent object **has no global meaning** without its owner and its discriminator is only unique **in owner’s scope**.

- **Weak entity fit:** `SECTION` under `COURSE` — section 1 exists for many courses.  
- **Strong + FK fit:** `ENROLLMENT` linking student and section — enrollment id `E-991` can be globally unique even though it references student and section.

**Rule of thumb:**

| Question | Weak entity | Strong + FK |
|----------|-------------|-------------|
| Can ID exist without owner? | No | Often yes |
| Is partial key natural? | Yes (`sec_no` per course) | Use surrogate `enrollment_id` instead |
| Identifying relationship? | Required (double diamond) | Regular relationship |

**Over-modeling trap:** Making every child table a weak entity. If you assign `enrollment_id` as surrogate PK, model **ENROLLMENT** as strong entity type related by binary `Enrolls`.

**CampusDB:** Dependent of professor → weak. Student’s optional club membership with global membership number → strong `MEMBERSHIP(membership_id, roll_no, club_code)`.

---

### Q3. “Explain multivalued and derived attributes. How do you map them to relations?”

**Deep answer:**

**Multivalued:** Attribute may hold a **set of values** per entity — `STUDENT.phone_numbers = {mobile, home}`. ER: **double oval**. Relational: **never** store as `phone1, phone2` without limit; create `student_phone(roll_no, phone, type)` with FK to `STUDENT`. Violating this breaks **1NF** (non-atomic cell).

**Derived:** Value computed from others — `age` from `dob`, `enrollment_count` from enrollments. ER: **dashed oval**. Relational: prefer **view** `student_age AS SELECT roll_no, EXTRACT(YEAR FROM AGE(dob)) …` or compute in app. Storing derived data requires **trigger/maintenance** (controlled redundancy, Chapter 02).

**Composite:** `name` → components; map to multiple columns or keep composite only in ER documentation stage.

**Interview twist:** “Can `gpa` be derived?” — Yes from grades + credits; if stored for transcripts, document as **materialized** derived attribute with refresh policy.

---

## Part I — Exercises (attempt before hints)

### Exercise 1 — Draw and classify (CampusDB extension)

A library module is added:

- Each **book** has ISBN (unique), title, and **multiple authors** (author names can repeat across books).  
- A **copy** of a book on shelf (barcode unique globally) may be **borrowed** by at most one student at a time; loan has `due_date`.  
- Student has `roll_no`, composite `name`, multivalued `email_addresses`.

1. List **entity types** (strong vs weak) with justification.  
2. Classify each mentioned attribute (simple / composite / multivalued / derived / key).  
3. Write a **conceptual diagram description** (rectangles, diamonds, double ovals, double diamonds) — no need to submit a graphic, prose is enough.  
4. Name relationship types and their **degree** (unary/binary/ternary).

<details>
<summary>Hint (after you try)</summary>

BOOK strong (ISBN); COPY weak under BOOK? Often strong with barcode PK; or weak with partial copy_no per ISBN — both arguable if barcode global. BORROWS binary STUDENT–COPY with due_date on diamond. AUTHORS multivalued → book_author table or AUTHOR entity + M:N. EMAIL multivalued table.
</details>

---

### Exercise 2 — Unary, binary, ternary decisions

For each scenario at CampusDB, choose **unary, binary, or ternary** and sketch who participates:

1. Professor **collaborates with** professor on research (same department or not).  
2. Student **rates** course **and** professor after semester (one rating ties all three).  
3. Course **is prerequisite for** another course.  
4. Professor **teaches** section; section **belongs to** course.

Explain why (2) should not be split into only `Student–Course` and `Student–Professor` if the rating is “for that professor in that course.”

<details>
<summary>Hint (after you try)</summary>

(1) unary Collaborates on PROFESSOR. (2) ternary RATES (STUDENT, COURSE, PROFESSOR) or binary RATING with three FKs. (3) unary Prerequisite on COURSE. (4) two binaries or Teaches + identifying Offered_As for section-course.
</details>

---

## Part J — Chapter summary

| Concept | CampusDB anchor |
|---------|-----------------|
| **Entity vs type** | Arjun vs rectangle STUDENT |
| **Strong entity** | STUDENT keyed by `roll_no` |
| **Weak entity** | SECTION under COURSE, partial `sec_no` |
| **Simple / composite / multivalued / derived / key** | phone (double oval), age (dashed), name (tree) |
| **Relationship vs type** | Arjun enrolled once vs diamond ENROLLS |
| **Unary** | Course prerequisite on course |
| **Binary** | Professor teaches section |
| **Ternary** | Professor assigns grading component in section |

ER diagrams are **contracts with reality** before SQL. Draw types, mark keys, separate multivalued facts — and relational schemas become straightforward.

---

*Prev: [Chapter 04 — Relational model](../Foundation/chapter-04-relational-model-codd.md) · Next: [Chapter 06 — Drawing ER diagrams](./chapter-06-er-diagrams-notation.md)*
