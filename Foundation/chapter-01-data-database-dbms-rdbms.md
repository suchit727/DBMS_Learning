# Chapter 01 — Foundations: Data, Database, DBMS, and RDBMS

*Senior DBMS lecture notes — IIT-style treatment*

**Series:** [Ch 01](./chapter-01-data-database-dbms-rdbms.md) · [Ch 02](./chapter-02-file-systems-inadequacy.md) · [Ch 03](./chapter-03-dbms-architecture.md) · [Ch 04](./chapter-04-relational-model-codd.md)

---

## 1. What is Data?

### Precise definition

**Data** are raw, unorganized facts represented as symbols (numbers, characters, bits, images, audio) that describe some aspect of the real world. Data alone carry no intrinsic meaning until they are interpreted in context.

Formally: if \(D\) is a set of atomic values (e.g., `50000`, `"Rahul"`, `2026-05-23`), then each element is *data* until it is assigned semantics (what entity, what attribute, what unit).

### Real-world analogy (banking)

When an ATM prints:

```
50000
Rahul
20260523
```

you see three isolated values. You do not yet know whether `50000` is a balance in rupees, a transaction ID, or a PIN attempt count. That is **data** — facts without agreed meaning.

When the bank’s system labels them as `balance_inr`, `account_holder_name`, and `last_transaction_date`, the same symbols become **information**.

### Why data exists (what problem it solves)

Organizations must record reality: who bought what, who owes whom, which seat is booked. Without recording facts, you cannot operate, audit, or decide. Data is the *input* to every business process.

The problem data alone does **not** solve: **consistency of meaning**. Two departments can store `50000` with different interpretations unless rules exist.

### What came before

| Era | Approach | Limitation |
|-----|----------|------------|
| Oral / ledger books | Clerks wrote amounts in registers | No concurrent access; errors; no search at scale |
| Flat files (`.txt`, `.csv`) | Programs read/write files directly | Duplication; no shared schema; application owns structure |

**Data** is the atomic layer. Everything above (database, DBMS) exists to give data **structure, meaning, and safe access**.

---

## 2. What is a Database?

### Precise definition

A **database** is an organized, persistent collection of related data, stored so that it can be accessed efficiently and maintained with integrity constraints. It is typically abstracted from physical storage details.

Key properties:

- **Organized** — grouped by schema (tables, documents, graphs, etc.)
- **Persistent** — survives program termination and power loss
- **Related** — entities are linked (customer ↔ orders ↔ payments)
- **Shared** — multiple applications/users access the same logical store

### Real-world analogy (e-commerce)

Think of an e-commerce company’s **single warehouse inventory system**, not scattered Excel sheets per team.

| Without a database | With a database |
|--------------------|-----------------|
| Marketing keeps `products.csv` | One `products` table |
| Warehouse keeps `stock.xls` | One `inventory` table linked to `products` |
| Finance keeps `revenue.txt` | One `orders` / `payments` schema |

The **database** is the unified “warehouse ledger” where every department sees consistent product IDs, prices, and stock — one source of truth.

### Why a database exists (what problem it solves)

1. **Data redundancy** — same customer stored 10 times in 10 files  
2. **Inconsistency** — price updated in one file, stale in another  
3. **Isolation** — each app invents its own file format  
4. **Durability** — business cannot afford to lose orders on crash  

A database centralizes data under a **schema** so many programs read/write **one logical model**.

### What came before

- **File-based systems (1960s–70s):** Each application had its own files (`CUSTOMER.dat`, `ORDER.dat`). No central catalog; no standard query language.
- **Hierarchical / network models:** Data in trees or graphs (IMS, CODASYL). Hard to change structure; programmer navigated pointers.

**Database** (as a *concept*) answers: *“Where do we keep all related facts together, with rules?”*

---

## 3. What is a DBMS?

### Precise definition

A **Database Management System (DBMS)** is system software that provides an interface to define, create, query, update, and administer databases, while handling storage, concurrency, recovery, security, and integrity **on behalf of applications**.

The DBMS sits between users/applications and the physical data:

```
[ Users / Apps ]  →  [ DBMS ]  →  [ Database (stored data) ]
```

Core responsibilities:

| Function | Meaning |
|----------|---------|
| **Data definition** | DDL: create/alter schema |
| **Data manipulation** | DML: insert, update, delete, query |
| **Concurrency control** | Many users; avoid lost updates |
| **Recovery** | Crash → consistent state (logging, checkpoints) |
| **Security** | Authentication, authorization |
| **Integrity** | Enforce constraints (UNIQUE, FK, CHECK) |

Examples (broad family): MySQL, PostgreSQL, MongoDB, Redis, Oracle, SQLite.

### Real-world analogy (banking)

The **DBMS** is the entire **core banking platform** — not the vault (data), not the teller alone (app).

- Tellers (apps) never open the vault directly.
- They request: “transfer ₹10,000 from A to B.”
- The platform (DBMS) checks balance, locks rows, writes audit logs, commits or rolls back.

Without it, two tellers could both debit the same ₹10,000 from a ₹10,000 balance (race condition).

### Why a DBMS exists (what problem it solves)

Applications should **not** implement:

- Disk block layout  
- Transaction atomicity (ACID)  
- Locking when 10,000 users checkout simultaneously  
- Backup/restore after disk failure  

**Separation of concerns:** programmers express *what* data they need; the DBMS handles *how* it is stored and kept correct.

### What came before

| Before DBMS | With DBMS |
|-------------|-----------|
| Application code opens files, seeks records | Apps issue SQL/API calls |
| Each app implements locking | Central scheduler |
| Crash → corrupted files | WAL, redo/undo |
| Ad hoc backup scripts | Integrated backup tools |

**Database** = the organized data. **DBMS** = the engine that manages it.

---

## 4. What is an RDBMS?

### Precise definition

A **Relational Database Management System (RDBMS)** is a DBMS that stores data in **relations** (tables) consisting of **rows** (tuples) and **columns** (attributes), and exposes a **declarative query language** (typically SQL) based on **relational algebra / calculus**.

Codd’s relational model (1970) requires:

- Data as relations, not physical pointers exposed to users  
- **Keys** identify rows; **foreign keys** express relationships  
- **Normalization** reduces redundancy  
- **ACID transactions** on multi-row updates  

Examples: PostgreSQL, MySQL (InnoDB), Oracle, SQL Server, SQLite.

### Real-world analogy (e-commerce)

An e-commerce RDBMS models:

```text
customers(id, name, email)
orders(id, customer_id, order_date, total)
order_items(order_id, product_id, qty, price)
products(id, name, sku, price)
```

- `customer_id` in `orders` is a **foreign key** → “this order belongs to this customer.”
- You can **JOIN** tables to answer: “Total revenue per customer in May 2026” without writing nested file loops.

The **relation** is the spreadsheet-like table; **integrity** is enforced by the engine, not by hope.

### Why an RDBMS exists (what problem it solves)

Early hierarchical/network DBMS forced programs to navigate physical links. Changing storage layout broke applications.

The relational model solves:

1. **Data independence** — logical tables stable; physical layout can change  
2. **Declarative queries** — say *what* you want (`SELECT ... WHERE`), not *how* to traverse pointers  
3. **Mathematical foundation** — joins, projections, selections are well-defined  
4. **Integrity** — keys and constraints are first-class  

### What came before

| Model | Example | Issue |
|-------|---------|-------|
| Hierarchical | IBM IMS | Tree-only; awkward many-to-many |
| Network | CODASYL | Complex pointers; tight coupling to code |
| Relational | System R → SQL | Table + logic separation; became dominant for OLTP |

**Not every DBMS is relational.** MongoDB (document), Neo4j (graph), Redis (key-value) are DBMSs but **not** RDBMSs.

---

## 5. DBMS vs RDBMS — Concrete Example

### Scenario: Flipkart-style order tracking

**Requirement:** Store customers, orders, and products; ensure an order cannot reference a non-existent customer; report total sales per product.

---

### Approach A — Generic DBMS (e.g., document store, no enforced relations)

```json
// Collection: orders
{
  "order_id": "O1001",
  "customer": { "name": "Asha", "email": "asha@mail.com" },
  "items": [
    { "sku": "PHONE-9", "qty": 1, "price": 49999 }
  ]
}
```

**Problems:**

- Customer email duplicated in every order document → update email in 50 places if she changes it  
- No DB-enforced rule: `customer_id` must exist in `customers`  
- “Revenue per product” needs application code to scan all JSON documents  

The **DBMS** still gives persistence and concurrency, but **structure is imposed by the application**.

---

### Approach B — RDBMS (PostgreSQL / MySQL)

```sql
CREATE TABLE customers (
  id         INT PRIMARY KEY,
  name       VARCHAR(100) NOT NULL,
  email      VARCHAR(255) UNIQUE NOT NULL
);

CREATE TABLE products (
  id    INT PRIMARY KEY,
  sku   VARCHAR(50) UNIQUE NOT NULL,
  price DECIMAL(12,2) NOT NULL
);

CREATE TABLE orders (
  id          INT PRIMARY KEY,
  customer_id INT NOT NULL REFERENCES customers(id),
  order_date  DATE NOT NULL
);

CREATE TABLE order_items (
  order_id   INT NOT NULL REFERENCES orders(id),
  product_id INT NOT NULL REFERENCES products(id),
  qty        INT NOT NULL CHECK (qty > 0),
  PRIMARY KEY (order_id, product_id)
);

-- Revenue per product (declarative, one query)
SELECT p.sku, SUM(oi.qty * pr.price) AS revenue
FROM order_items oi
JOIN products pr ON pr.id = oi.product_id
JOIN orders o ON o.id = oi.order_id
GROUP BY p.sku;
```

**What the RDBMS adds:**

| Feature | Benefit in this example |
|---------|-------------------------|
| `REFERENCES customers(id)` | Cannot insert orphan order |
| Normalized tables | Email stored once in `customers` |
| SQL `JOIN` + `GROUP BY` | Analytics without custom scanners |
| ACID transaction | Debit stock + insert order + charge payment — all or nothing |

### One-line distinction

> **DBMS** = software that manages any database model.  
> **RDBMS** = DBMS where the model is **relations (tables)** with **SQL** and **relational integrity**.

Every RDBMS is a DBMS; not every DBMS is an RDBMS.

---

## 6. Concept Map (quick revision)

```text
Data          → raw facts
     ↓
Database      → organized, persistent, related collection
     ↓
DBMS          → software that manages the database (any model)
     ↓
RDBMS         → DBMS based on relational tables + SQL + keys/FKs
```

---

## 7. Interview Questions (with deep answers)

### Q1. “Is a spreadsheet (Excel) a database? Is Excel a DBMS?”

**Deep answer:**

A single Excel workbook can hold **organized, persistent, related data** — so colloquially people call it a “database.” For a strict DBMS course:

- **As a database (loose sense):** Yes, if it is the authoritative store for a small domain (e.g., club membership list).
- **As a database (strict / enterprise sense):** Usually **no** — weak concurrent write control (file locking), limited integrity (no real foreign keys across sheets in older Excel), no ACID multi-user transactions, size limits, and security model geared to documents not multi-app access.

- **Excel is not a DBMS** in the Codd/Date sense: it is a **spreadsheet application** with tabular UI, not a dedicated storage engine with query optimizer, WAL recovery, and fine-grained authorization for thousands of concurrent clients.

**Follow-up insight:** Google Sheets + Apps Script closer to “shared data,” but production systems use PostgreSQL/MySQL because of **concurrency, integrity, and scale**.

---

### Q2. “Why did relational databases win over hierarchical and network models?”

**Deep answer:**

1. **Programmer productivity** — SQL is declarative; network models required navigational code tied to physical structure. Schema migration broke apps.

2. **Data independence** — Logical view (tables) decoupled from storage. IMS programmers knew exact record paths.

3. **Theory** — Relational algebra gave optimizers a basis to reorder joins, use indexes, and improve plans without changing queries.

4. **Ad hoc reporting** — Business questions (“sales by region last quarter”) are JOIN/aggregate, not rewrites of traversal loops.

5. **Normalization** — Reduced update anomalies systematically.

**Caveat (mature answer):** Relational did not “end” other models. JSON columns, graph DBs, and warehouses exist for semi-structured data, graphs, and analytics — but **OLTP banking/e-commerce** still overwhelmingly uses RDBMS for ACID and integrity.

---

### Q3. “What is the difference between data and information? Give an example from banking.”

**Deep answer:**

- **Data:** symbols without agreed context — `9876543210`, `150000`, `DEBIT`.
- **Information:** data + semantics + (often) timeliness for a decision — “Account 9876543210 was debited ₹1,50,000 on 2026-05-23 for NEFT to merchant X; available balance ₹2,00,000.”

The transformation requires:

- **Schema** (which column is what)  
- **Metadata** (currency INR, timestamp UTC)  
- **Relationships** (this debit belongs to this account)  
- Sometimes **derived values** (running balance)

**Interview trap:** “Big Data” is still mostly **data** until analytics assigns meaning (KPIs, fraud scores = **information/knowledge** in DIKW pyramid).

**DIKW (bonus):** Data → Information → Knowledge → Wisdom. DBMS sits at the layer that turns raw data into reliable, queryable information at scale.

---

## 8. Exercises (attempt before peeking at hints)

### Exercise 1 — Modeling & terminology

A startup stores user profiles in `users.json` (one file per user on disk). A second service appends orders to `orders.log`. Marketing copies email addresses into a separate `mailing_list.csv` weekly.

1. List **three problems** this setup shares with pre-database file systems.  
2. For each problem, state which layer fixes it: **database**, **DBMS**, or **RDBMS** (be precise).  
3. Sketch a minimal **relational schema** (table names + primary/foreign keys only) for users and orders.

<details>
<summary>Hint (after you try)</summary>

Think: redundancy, consistency, concurrent writes, orphan orders, no standard query language.
</details>

---

### Exercise 2 — DBMS vs RDBMS decision

For each requirement, argue **RDBMS** vs **non-relational DBMS** (e.g., document/graph) in 2–3 sentences:

| System | Requirements |
|--------|----------------|
| (a) Core banking ledger | Multi-table accounts, transfers, strict ACID, regulatory audit |
| (b) Real-time chat message feed | High write rate, flexible message payloads (text, image, reactions) |
| (c) Product catalog with highly variable attributes (furniture has dimensions; books have ISBN) | Some fields only apply to some categories |

Then write **one SQL constraint** that only an RDBMS-style engine enforces natively in the example (a), and explain what goes wrong without it.

<details>
<summary>Hint (after you try)</summary>

(a) → RDBMS. (b)/(c) → often document/flexible schema — but hybrid designs exist. Constraint: `FOREIGN KEY`, `CHECK (balance >= 0)`, or transaction isolation preventing double spend.
</details>

---

## 9. Chapter summary

| Concept | One sentence |
|---------|----------------|
| **Data** | Raw facts without inherent meaning |
| **Database** | Organized, persistent, shared collection of related data |
| **DBMS** | Software that defines, stores, secures, and recovers that data for many apps |
| **RDBMS** | DBMS using tables, keys, SQL, and relational integrity |

Master these four terms before normalization, ER diagrams, and transaction isolation — every later topic assumes you know **what layer** you are talking about.

---

*Next: [Chapter 02 — Why file systems fail at data management](./chapter-02-file-systems-inadequacy.md)*
