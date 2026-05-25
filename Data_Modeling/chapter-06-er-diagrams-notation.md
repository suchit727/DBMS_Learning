# Chapter 06 — Drawing Complete ER Diagrams

*Senior DBMS lecture notes — IIT-style treatment*

**Prerequisites:**

- [Chapter 05 — ER modeling concepts](./chapter-05-er-modeling-university.md)

**Series (Foundation):** [Ch 01](../Foundation/chapter-01-data-database-dbms-rdbms.md) · … · [Ch 04](../Foundation/chapter-04-relational-model-codd.md)

**Data modeling track:** [Ch 05](./chapter-05-er-modeling-university.md) · [Ch 06](./chapter-06-er-diagrams-notation.md) · [Ch 07](./chapter-07-cardinality-participation.md)

---

## What you will learn

Chapter 05 defined **entities, attributes, relationships**. This chapter teaches **how to draw** them correctly in:

1. **Chen notation** (exam / theory standard)  
2. **Crow’s foot notation** (industry / tools standard)  

Then you apply both to a full **social media app** (`ConnectHub`) step by step.

---

## Part A — Chen notation (Peter Chen, 1976)

### Symbols cheat sheet

| Symbol | Shape | Represents |
|--------|-------|------------|
| **Entity type** | Rectangle | `USER`, `POST`, … |
| **Weak entity type** | Double rectangle | `COMMENT` owned by `POST` (when modeled weak) |
| **Relationship type** | Diamond | `Posts`, `Follows`, … |
| **Identifying relationship** | Double diamond | Links weak entity to owner |
| **Attribute** | Ellipse (oval) | `username`, `created_at` |
| **Key attribute** | Underlined oval | `user_id` |
| **Multivalued attribute** | Double oval | `hashtags` |
| **Derived attribute** | Dashed oval | `follower_count` |
| **Composite attribute** | Oval tree | `name` → first, last |
| **Link** | Line | Connects entity ↔ attribute, entity ↔ diamond |

### Conceptual diagram description — minimal Chen fragment

**To draw one strong entity with key and simple attribute:**

1. Draw a **rectangle** centered, label `USER`.  
2. Draw an **oval** below the rectangle, label `user_id`, **underline** the text (key).  
3. Connect rectangle bottom to oval top with a **straight line**.  
4. Draw another oval `username` (no underline), connect to rectangle.  

```text
              ┌──────────┐
    username ─┤          ├─ user_id   (underline user_id in your drawing)
              │   USER   │
              └──────────┘
```

### Lines — rules

| Connection | Allowed? |
|------------|----------|
| Entity rectangle ↔ attribute oval | Yes |
| Entity rectangle ↔ relationship diamond | Yes |
| Attribute oval ↔ relationship diamond | Yes (attributes of relationship) |
| Entity rectangle ↔ entity rectangle directly | **No** — must use diamond |
| Attribute oval ↔ attribute oval (except composite tree) | Only for composite components |

**Cardinality in Chen:** Often written as **1**, **N**, **M** near the lines between entity and diamond (Chapter 05 preview). **Participation** uses **double lines** (see Part E).

---

## Part B — Crow’s foot notation (IE / Barker / industry)

Used in MySQL Workbench, dbdiagram.io, ER/Studio. **Entities are rectangles**; **relationships are lines** between rectangles (no diamond). **Cardinality and participation** are marks on the **ends** of lines.

### Cardinality symbols (end of line)

| Symbol | Meaning |
|--------|---------|
| **Single line (perpendicular tick)** | Exactly one |
| **Crow’s foot (three prongs)** | Many (zero or more, or one or more — read with circle) |
| **Open circle (O)** on line | Zero allowed (optional) |
| **Filled circle (●)** or **tick on line** | One required (some tools differ — learn your tool’s legend) |

**Common pairings (read from USER toward POST):**

| Notation at POST end | Meaning |
|----------------------|---------|
| Crow’s foot only | Many (0..* or 1..* depending on other end) |
| Crow’s foot + circle | Zero or many (**optional** many) |
| Single line + tick | Exactly one |
| Crow’s foot without circle on mandatory side | At least one / many |

**Text alternative (clearer in exams):** Label ends `0..1`, `1..1`, `0..*`, `1..*` directly on the diagram.

### Chen vs Crow’s foot — when to use

| Context | Prefer |
|---------|--------|
| IIT exams, GATE, classic DBMS courses | **Chen** |
| Startups, data modeling interviews with tools | **Crow’s foot** or **UML** |
| This chapter’s social media walkthrough | **Both** — Chen first, then crow’s foot summary |

```text
Chen:     [USER]────◇ Posts ◇────[POST]

Crow's:   [USER]───────<──────[POST]
               (line with crow's foot at POST side = many posts per user)
```

---

## Part C — Representing every attribute type visually

### 1. Simple attribute

| Notation | Draw |
|----------|------|
| **Chen** | Single oval, single line to entity |
| **Crow’s foot** | List inside rectangle: `username` (often all attrs in box) |

**ConnectHub:** `USER.email` — single oval in Chen; column inside `USER` box in crow’s foot.

---

### 2. Composite attribute

| Notation | Draw |
|----------|------|
| **Chen** | Oval `name` connected to sub-ovals `first_name`, `last_name` (tree) |
| **Crow’s foot** | Flatten: `first_name`, `last_name` inside rectangle (composite concept implicit) |

**Diagram description:** Main oval `display_name` hangs from `USER`; branch lines split to `first` and `last` ovals.

---

### 3. Multivalued attribute

| Notation | Draw |
|----------|------|
| **Chen** | **Double oval** around attribute name |
| **Crow’s foot** | Separate entity `USER_PHONE(user_id, phone)` — crow’s foot rarely uses double ovals; model explicitly |

**ConnectHub:** `POST.hashtags` — double oval in Chen; or weak entity `HASHTAG` + M:N `Tagged_With` in advanced designs.

---

### 4. Derived attribute

| Notation | Draw |
|----------|------|
| **Chen** | **Dashed oval** |
| **Crow’s foot** | Omit from physical diagram or mark `(derived)` in notes |

**ConnectHub:** `USER.follower_count` dashed oval — computed from `Follows` relationship.

---

### 5. Key attribute

| Notation | Draw |
|----------|------|
| **Chen** | **Underline** attribute name in oval |
| **Crow’s foot** | **Bold** or `PK` or 🔑 in rectangle; composite PK underlined together |

**ConnectHub:** `user_id`, `post_id` underlined.

---

### Quick reference table (draw this in notes)

| Attribute type | Chen | Crow’s foot (typical) |
|----------------|------|------------------------|
| Simple | Oval | In entity box |
| Composite | Oval tree | Split columns in box |
| Multivalued | Double oval | Separate table/entity |
| Derived | Dashed oval | Note / view only |
| Key | Underline | PK marker |

---

## Part D — Weak entities and identifying relationships

### Chen notation

| Element | Draw |
|---------|------|
| Weak entity | **Double rectangle** |
| Partial key (discriminator) | **Dashed underline** on oval (e.g. `comment_seq`) |
| Identifying relationship | **Double diamond** between owner and weak entity |
| Owner (strong) | Single rectangle |

**Diagram description — `COMMENT` depends on `POST`:**

1. Rectangle `POST` with underlined `post_id`.  
2. Double rectangle `COMMENT` with ovals `comment_seq` (dashed underline) and `body`, `created_at`.  
3. Double diamond `Contains` from `POST` to `COMMENT`.  
4. Full key in mapping: `(post_id, comment_seq)`.

**Alternative design (also valid):** Strong `COMMENT` with surrogate `comment_id` PK — single diamond `Contains`, no double box. Exams may require weak form when discriminator is natural.

### Crow’s foot notation

Weak entities often appear as normal tables with **composite PK** including FK:

```text
[POST] 1 ───────< * [COMMENT]
      (PK post_id)     (PK post_id + comment_seq, FK post_id)
```

Identifying relationship = **identifying** crow’s foot (child cannot exist without parent) — line from `POST` to `COMMENT` with **mandatory participation on COMMENT side** (cannot have comment without post).

---

## Part E — Participation constraints (total vs partial)

### Definitions

| Constraint | Meaning | Business reading |
|------------|---------|------------------|
| **Total participation** (mandatory) | Every entity in **this entity type** must participate in **at least one** instance of this relationship | “Every student must belong to at least one department” |
| **Partial participation** (optional) | Some entities may **not** participate | “Some employees are not assigned to any project” |

**Do not confuse** with **cardinality** (how many on the other side). Participation = **must you join at all?** Cardinality = **how many times if you do?**

### How to show in Chen notation

| Participation | Line style from entity to diamond |
|---------------|-----------------------------------|
| **Total** | **Double line** |
| **Partial** | **Single line** |

**Example — ConnectHub `USER` creates `POST`:**

- Every `POST` must have an author → **POST** side **total** participation in `Posts` (double line from POST to diamond).  
- Not every `USER` must post → **USER** side **partial** (single line from USER to diamond).

```text
USER ────────◇ Posts ◇════════ POST
(single)              (double)
 partial              total
```

### How to show in Crow’s foot notation

| Participation | Common mark |
|---------------|-------------|
| **Total (mandatory)** | **Filled circle (●)** or perpendicular bar on **that entity’s end** of the connector |
| **Partial (optional)** | **Open circle (○)** on that end |

Plus crow’s foot for “many” on the other end.

**Read example:** `USER` ○———<—— `POST`  
- Circle at USER: user may exist with zero posts (partial).  
- Crow’s foot at POST side + mandatory bar at POST: each post has exactly one author (total from POST toward relationship).

*Tool legends vary — always label `0..*` / `1..1` in exam answers if unsure.*

### Combined cardinality + participation table (ConnectHub)

| Relationship | USER participation | POST participation | Cardinality (typical) |
|--------------|-------------------|--------------------|------------------------|
| **Posts** | Partial | Total | USER 1 : POST N |
| **Follows** | Partial both | Partial both | M : N |
| **Likes** | Partial | Partial | M : N (via associative) |

---

## Part F — Step-by-step: complete ER for ConnectHub (social media)

**Requirements:**

- **Users** register with `user_id`, `username`, `email`, composite `display_name`, optional `bio`.  
- **Posts** have `post_id`, `content`, `created_at`; every post has **exactly one** author; users may post zero or many.  
- **Comments** on posts; comment identified by post + sequence number `comment_seq` (weak under post) OR use global `comment_id` — we draw **weak** form for teaching.  
- **Likes:** user may like many posts; post may have many likes; same user likes same post at most once; store `liked_at`.  
- **Followers:** directed follow (A follows B); not every user follows anyone; store `followed_at`.  
- **Derived:** `follower_count` on user (optional dashed).  
- **Multivalued:** hashtags on post (double oval).

---

### Step 1 — List entity types and keys

| Entity | Type | Key |
|--------|------|-----|
| USER | Strong | `user_id` |
| POST | Strong | `post_id` |
| COMMENT | Weak (under POST) | `post_id` + `comment_seq` |
| LIKE | Associative (relationship with attrs) | `(user_id, post_id)` |
| FOLLOW | Associative (directed) | `(follower_id, followee_id)` |

*Hashtag:* multivalued on POST in Chen; optional entity `HASHTAG` for normalization.

---

### Step 2 — List relationship types, degree, cardinality, participation

| # | Relationship | Degree | Cardinality | Participation notes |
|---|--------------|--------|-------------|---------------------|
| 1 | **Posts** | Binary USER–POST | 1 : N | POST total in Posts; USER partial |
| 2 | **Contains** | Binary POST–COMMENT (identifying) | 1 : N | COMMENT total; POST partial (post may have zero comments) |
| 3 | **Likes** | Binary USER–POST | M : N | Both partial |
| 4 | **Follows** | Binary USER–USER (unary roles) | M : N directed | Both partial |

---

### Step 3 — Draw strong entities (Chen)

1. Place **USER** rectangle left, **POST** rectangle right (horizontal layout).  
2. Attach ovals: USER — `user_id` (underline), `username`, `email`, composite `display_name` → `first`, `last`, `bio` (partial: bio optional — note in prose, not a special line).  
3. POST — `post_id` (underline), `content`, `created_at`.  
4. Double oval `hashtags` → POST.  
5. Dashed oval `follower_count` → USER.

**Diagram description after step 3:**

Left third of page: USER box with five attribute branches (name tree + key + simple + derived dashed). Right third: POST box with key, content, time, double oval hashtags.

---

### Step 4 — Draw weak COMMENT (Chen)

1. **Double rectangle** `COMMENT` below POST.  
2. Ovals: `comment_seq` (**dashed underline**), `body`, `commented_at`.  
3. **Double diamond** `Contains` between POST and COMMENT.  
4. **Double line** from COMMENT to `Contains` (total: every comment belongs to a post).  
5. **Single line** from POST to `Contains` (partial: post may have zero comments).

---

### Step 5 — Draw binary relationships Posts, Likes, Follows (Chen)

**Posts (USER–POST):**

- Diamond `Posts` between USER and POST.  
- Single USER–`Posts`; double `Posts`–POST.  
- Label near USER: `1`, near POST: `N`.

**Likes (M:N with attribute):**

- Diamond `Likes` between USER and POST (separate from Posts).  
- Oval `liked_at` attached to diamond.  
- Single lines both sides (partial).  
- Labels `M` and `N`.

**Follows (unary on USER):**

- Diamond `Follows` below USER.  
- Two lines back to USER with roles **`follower`** and **`followee`**.  
- Single lines (partial).  
- Cardinality `M` : `N` (many followers, many following).

```text
                    first ─┐
                    last  ─┤ display_name
                           │
    follower_count (--)    │     hashtags (( ))
         ┌─────────────────┴─────────────────┐
         │              USER                 │
         └──────────┬───────────┬────────────┘
                    │           │
              (single)     (single)  follower / followee
                    │           │
                 ◇ Posts ◇   ◇ Follows ◇
                    │           │
              (double)          (single both roles)
                    │           │
         ┌──────────┴───────────┴────────────┐
         │              POST                   │
         └──────────────────┬──────────────────┘
                            │ (double)
                       ╔════◇ Contains ════╗
                            │
                    ┌───────┴────────┐
                    │    COMMENT     │  (double rectangle)
                    │ comment_seq ~~ │
                    └────────────────┘

         USER ────◇ Likes ◇──── POST
              (single)  liked_at  (single)
```

---

### Step 6 — Crow’s foot version (same schema, compact)

Draw four boxes: `USER`, `POST`, `COMMENT`, and either diamond-less links or junction tables:

| Link | Crow’s foot |
|------|-------------|
| USER → POST | `USER` 1 ——○——< `POST` (one user, optional many posts; each post one user) |
| POST → COMMENT | `POST` 1 ——< `COMMENT` (identifying FK `post_id` in COMMENT) |
| USER ↔ POST likes | `USER` >———< `POST` through `LIKE(user_id, post_id, liked_at)` |
| USER follows USER | `USER` >———< `USER` through `FOLLOW(follower_id, followee_id, followed_at)` |

**Diagram description:**

- Place `USER` top-left, `POST` top-right, `COMMENT` under `POST`.  
- Mandatory one-to-many from USER to POST (bar at POST, crow’s foot at USER’s post side).  
- `LIKE` rectangle between USER and POST (associative entity).  
- `FOLLOW` rectangle with two FKs to `USER` (role labels follower / followee).

---

### Step 7 — Relational schema (sanity check after drawing)

```sql
CREATE TABLE users (
  user_id    BIGINT PRIMARY KEY,
  username   VARCHAR(50) UNIQUE NOT NULL,
  email      VARCHAR(255) UNIQUE NOT NULL,
  first_name VARCHAR(100),
  last_name  VARCHAR(100),
  bio        TEXT
);

CREATE TABLE posts (
  post_id     BIGINT PRIMARY KEY,
  user_id     BIGINT NOT NULL REFERENCES users(user_id),
  content     TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL
);

CREATE TABLE post_hashtags (
  post_id BIGINT NOT NULL REFERENCES posts(post_id),
  tag     VARCHAR(100) NOT NULL,
  PRIMARY KEY (post_id, tag)
);

CREATE TABLE comments (
  post_id      BIGINT NOT NULL REFERENCES posts(post_id),
  comment_seq  INT NOT NULL,
  user_id      BIGINT NOT NULL REFERENCES users(user_id),
  body         TEXT NOT NULL,
  commented_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (post_id, comment_seq)
);

CREATE TABLE likes (
  user_id   BIGINT NOT NULL REFERENCES users(user_id),
  post_id   BIGINT NOT NULL REFERENCES posts(post_id),
  liked_at  TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (user_id, post_id)
);

CREATE TABLE follows (
  follower_id BIGINT NOT NULL REFERENCES users(user_id),
  followee_id BIGINT NOT NULL REFERENCES users(user_id),
  followed_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (follower_id, followee_id),
  CHECK (follower_id <> followee_id)
);
```

If your ER matches these tables, the diagram is **complete**.

---

## Part G — Interview questions (with deep answers)

### Q1. “Chen vs crow’s foot — are they different models?”

**Deep answer:**

They depict the **same conceptual model** (entities, relationships, constraints) with **different glyphs**.

| Aspect | Chen | Crow’s foot |
|--------|------|-------------|
| Relationship | Diamond | Line + cardinality marks |
| Attributes | External ovals | Often inside rectangle |
| Weak entity | Double box + double diamond | Composite PK + mandatory FK |
| Participation | Double line | Circle / bar on endpoint |

**Interview move:** Draw ConnectHub `Posts` in Chen (double line on POST side), then redraw as `USER 1—< * POST` with mandatory FK `user_id NOT NULL` — same constraint.

**Trap:** Mixing diamonds and crow’s feet on one diagram without legend — examiners mark down.

---

### Q2. “Total participation vs mandatory cardinality — is ‘every post has an author’ total or 1:N?”

**Deep answer:**

**Both apply — different dimensions.**

- **Cardinality 1:N (from POST to USER):** Each post links to **exactly one** user (not two authors in this model).  
- **Total participation of POST in Posts:** Every post **must** participate in `Posts` — no post without author. Shown as **double line** (Chen) from POST to diamond.  
- **Partial participation of USER:** Some users never post — allowed. **Single line** from USER to diamond.

**Mandatory cardinality** (`1..1` on POST side) ≈ total participation + max 1.  
**Optional many** (`0..*` on USER side) = partial participation + crow’s foot.

**Sound bite:** “Participation = must you play? Cardinality = how many partners if you do?”

---

### Q3. “Should Likes be a diamond or a table in the ER diagram?”

**Deep answer:**

**Conceptual ER:** `Likes` is a **relationship type** with attribute `liked_at` — diamond with oval (Chen) or **associative entity** rectangle (crow’s foot / Barker when relationship has complex growth).

**Relational:** Always maps to table `likes(user_id, post_id, liked_at)` with composite PK.

**When to elevate to associative entity in diagram:**

- M:N with **payload attributes** (`liked_at`, `reaction_type`)  
- Relationship may have **independent lifecycle** or own identifier in extensions  

**Not a weak entity:** Like is not identified only by partial key under one owner — composite `(user_id, post_id)` is global in scope.

**ConnectHub:** Diamond `Likes` (Chen) or rectangle `LIKE` between USER and POST (crow’s foot) — both correct; pick one notation and stay consistent.

---

## Part H — Exercises (model these systems)

Draw **full ER diagrams** in **both Chen and crow’s foot** (or Chen + cardinality labels `0..1` / `1..*` if time-limited). Include keys, one multivalued or composite attribute, at least one M:N, and mark **total vs partial** participation on every relationship.

### System 1 — Food delivery platform (`QuickBite`)

Model:

- **Customer**, **Restaurant**, **DeliveryPartner**, **Order**, **MenuItem**.  
- Customer places orders; order has status, total, timestamp; must reference exactly one customer and one restaurant.  
- Order contains many menu items (quantity, line price) — same item can appear in many orders.  
- Delivery partner assigned to at most one active order at a time; order may exist before partner assigned (preparing).  
- Restaurant has multiple **cuisine tags** (multivalued).  
- Customer has composite **address** for delivery.

**Deliverable checklist:**

1. Entity list + strong/weak choice  
2. Chen diagram with participation lines  
3. Crow’s foot diagram (or labeled cardinalities)  
4. Bullet list mapping to table names  

<details>
<summary>Hint (after you try)</summary>

ORDER strong; ORDER_ITEM weak or associative (order_id, line_no); M:N Order–MenuItem via ORDER_ITEM; Assigns binary Order–DeliveryPartner partial on both until assigned; Restaurant total in Offers? partial.
</details>

---

### System 2 — University club management (`CampusClub`)

Extend [Chapter 05 CampusDB](../Foundation/chapter-05-er-modeling-university.md) with:

- **Club**, **Student**, **FacultyAdvisor** (professor).  
- Club has president (exactly one student at a time); student may lead zero or one club.  
- Many students **join** many clubs (`joined_on`, `role`).  
- Club hosts **Event** (event_id, title, date); event belongs to one club; club may host zero events.  
- **BudgetExpense** for club: weak under club with `expense_seq`, amount, description.  
- Derived: `member_count` on club.

**Deliverable checklist:**

1. Unary/binary/ternary count per relationship  
2. Weak entity + identifying relationship drawn correctly  
3. Total vs partial on `President_Of` and `Hosts`  

<details>
<summary>Hint (after you try)</summary>

President_Of: STUDENT 1 — 0..1 CLUB (partial on student, total on club for president slot — careful: club may lack president temporarily → partial on club). Joins M:N. BudgetExpense weak. Events binary Club–Event 1:N total on Event side.
</details>

---

## Part I — Chapter summary

| Topic | Remember |
|-------|----------|
| **Chen** | Rectangle, oval, diamond; double/dashed for special attrs; double lines = total |
| **Crow’s foot** | Lines + ○ ● + crow’s foot; PK in box |
| **Weak + identifying** | Double rectangle + double diamond; dashed underline partial key |
| **Participation** | Total = must participate; partial = optional — **not** same as 1 vs N |
| **ConnectHub** | USER–POST–COMMENT weak–LIKE M:N–FOLLOW unary roles |

A complete ER diagram is **readable without verbal explanation** — every entity has a key, every relationship has cardinality and participation, and multivalued/derived attributes use the correct visual grammar.

---

*Prev: [Chapter 05 — ER modeling](./chapter-05-er-modeling-university.md) · Next: [Chapter 07 — Cardinality & participation](./chapter-07-cardinality-participation.md)*
