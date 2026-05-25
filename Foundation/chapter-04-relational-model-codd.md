# Chapter 04 — The Relational Model (E.F. Codd)

*Senior DBMS lecture notes — IIT-style treatment*

**Prerequisites:**

- [Chapter 01 — Data, Database, DBMS, RDBMS](./chapter-01-data-database-dbms-rdbms.md)
- [Chapter 03 — DBMS architecture](./chapter-03-dbms-architecture.md) (schema vs instance)

**Series:** [Ch 01](./chapter-01-data-database-dbms-rdbms.md) · [Ch 02](./chapter-02-file-systems-inadequacy.md) · [Ch 03](./chapter-03-dbms-architecture.md) · [Ch 04](./chapter-04-relational-model-codd.md)

---

## Historical context: why Codd mattered

In 1970, **Edgar F. Codd** (IBM Research) published *“A Relational Model of Data for Large Shared Data Banks.”* His goal was to escape **navigational** databases (hierarchical/network) where programs chased physical pointers.

**Core idea:** Data should look like **mathematics** (sets of tuples) to the user, while the system hides **how** bits are placed on disk.

**Running example — ShopKart (e-commerce):**

Throughout this chapter we model a fictional marketplace **ShopKart** with customers, products, orders, and payments — the same domain as Chapter 01’s e-commerce sketch, now with **relational precision**.

---

## 1. What is a relation?

### Mathematical definition

A **relation** \(R\) over attribute names \(A_1, A_2, \ldots, A_n\) is a **subset** of the Cartesian product of domains:

\[
R \subseteq D_1 \times D_2 \times \cdots \times D_n
\]

where each \(D_i\) is the **domain** (set of atomic legal values) for attribute \(A_i\).

Equivalently (Codd’s view): a relation is a **set of n-tuples** \((a_1, a_2, \ldots, a_n)\) where \(a_i \in D_i\).

**Key mathematical properties:**

- A relation is a **set** → **no duplicate tuples**  
- Tuples in a set are **unordered**  
- Attribute positions are identified by **name**, not by storage order (conceptually)

### Intuitive definition

A **relation** is a **flat table of facts** about one kind of thing in the business, where:

- Each **row** is one fact (one customer, one order line, one product SKU record)  
- Each **column** is one **property** all rows share (email, price, status)  
- Every cell holds **one atomic value** from a defined domain  

**ShopKart example — relation `Customers`:**

| customer_id | full_name | email | joined_on |
|-------------|-----------|-------|-----------|
| C-1001 | Priya Nair | priya@mail.com | 2024-01-15 |
| C-1002 | Arjun Mehta | arjun@mail.com | 2025-11-02 |

Mathematically, one tuple is:

\[
(\text{C-1001},\ \text{Priya Nair},\ \text{priya@mail.com},\ \text{2024-01-15}) \in \text{Customers}
\]

The **whole table body** (all rows at a moment) is the **relation instance**. The **heading** (column names + domains) is the **relation schema**.

---

## 2. Relation vs table — critical distinction

Students conflate them. In exams and industry, the difference matters.

| Aspect | **Relation** (model) | **Table** (implementation / SQL) |
|--------|----------------------|----------------------------------|
| **Nature** | Abstract mathematical **set** | Physical/logical storage structure in an RDBMS |
| **Duplicates** | **Forbidden** by definition | May exist unless `PRIMARY KEY` / `UNIQUE` enforced |
| **Order** | Rows and columns **unordered** | SQL may return rows in arbitrary order; `ORDER BY` imposes presentation order |
| **Nulls** | Debated extension; not in pure math sets | SQL `NULL` marker in cells |
| **Headers** | Attributes identified by **name** | Column list + optional display order in tools |
| **Access** | Relational algebra operators | SQL + indexes + pages under the hood |

**ShopKart illustration:**

**Pure relation mindset:** `Orders` is a set of 4-tuples; tuple `(O-501, C-1001, 2026-05-20, 2499.00)` either is in the set or is not — never twice.

**SQL table mindset:**

```sql
CREATE TABLE orders (
  order_id    CHAR(6) PRIMARY KEY,
  customer_id CHAR(6) NOT NULL REFERENCES customers(customer_id),
  order_date  DATE NOT NULL,
  total_inr   DECIMAL(12,2) NOT NULL CHECK (total_inr >= 0)
);
```

The **table** is how PostgreSQL **implements** the relation, plus extras:

- Storage order on disk (heap, clustered index)  
- System columns (`ctid`, `xmin` in PostgreSQL) invisible to the model  
- `NULL` allowed unless `NOT NULL`  
- Triggers, policies, fillfactor — **not** part of Codd’s abstract relation

**Professor’s line:** “Every relation can be **represented** as a table, but not every SQL table behaves like a pure relation unless you enforce keys and interpret NULL carefully.”

---

## 3. Tuples and attributes

### Attributes (columns)

An **attribute** is a **named role** in a relation, drawing values from one **domain**.

- **Schema notation:** `Customers(customer_id, full_name, email, joined_on)`  
- **Degree** = number of attributes \(n\) (here 4)  
- **Cardinality** (in one sense) = number of tuples in the **instance** (changes over time)

**ShopKart:** `Products(sku, title, category, price_inr, stock_qty)`

| Attribute | Meaning | Domain (informal) |
|-----------|---------|-------------------|
| `sku` | Stock keeping unit | alphanumeric strings |
| `title` | Display name | Unicode strings |
| `category` | Taxonomy label | enum-like strings |
| `price_inr` | Current list price | non-negative decimals |
| `stock_qty` | Warehouse count | non-negative integers |

### Tuples (rows)

A **tuple** is one element of the relation: an **n-tuple** \((a_1,\ldots,a_n)\).

- Identified by **values** on candidate key attributes (e.g. `customer_id = 'C-1001'`), not by row number  
- **Row number is not part of the model** — `SELECT` without `ORDER BY` does not guarantee stable physical order

**ShopKart order line tuple:**

```
(O-501, SKU-PHONE-9, 1, 49999.00)  ∈  OrderItems
```

Interpretation: order `O-501` includes one unit of `SKU-PHONE-9` at line price ₹49,999.

**Degree vs cardinality (exam vocabulary):**

| Term | Meaning | ShopKart example |
|------|---------|------------------|
| **Degree** | # attributes in schema | `OrderItems` has degree 4 |
| **Cardinality** | # tuples in current instance | 2.4 million line items today |

Do not confuse **relation cardinality** with **column cardinality** in SQL statistics (distinct values) — context disambiguates.

---

## 4. Domains and their importance

### Definition

A **domain** \(D\) is the set of **atomic** values that a given attribute may take, together with the **meaning** and **constraints** of those values.

Notation: `customer_id`: **Domain** = “ShopKart internal customer identifiers, format C-####”

Formally each attribute \(A_i\) has domain \(D_i\); the relation \(R \subseteq D_1 \times \cdots \times D_n\).

### Why domains matter

| Reason | ShopKart consequence without domains |
|--------|--------------------------------------|
| **Semantic clarity** | `price_inr` vs `weight_kg` — both numeric, not interchangeable |
| **Integrity** | Prevent `stock_qty = 'N/A'` or `email = 42` |
| **Operations** | Average of `price_inr` meaningful; average of `sku` meaningless |
| **Comparison safety** | Join `orders.total_inr` to `payments.amount_inr` only if same currency domain |
| **Implementation** | SQL `CHECK`, `ENUM`, `DECIMAL(12,2)`, custom `DOMAIN` types |

**SQL encoding of domains:**

```sql
CREATE DOMAIN rupees AS DECIMAL(12,2)
  CHECK (VALUE >= 0);

CREATE TABLE products (
  sku        VARCHAR(20) PRIMARY KEY,
  price_inr  rupees NOT NULL,
  stock_qty  INTEGER NOT NULL CHECK (stock_qty >= 0)
);
```

**Atomicity of domains (1NF preview):** A domain is **atomic** if you cannot decompose a value without losing meaning for that attribute. Bad: `full_address` storing `"12 MG Road, Pune 411001"` when you need pincode-based shipping rules — violates atomic domain use (multi-value / composite abuse).

**Professor’s line:** “Types in programming languages are domains; SQL `CHECK` and `REFERENCES` are domain constraints at the database boundary.”

---

## 5. Relation schema vs relation instance

| | **Relation schema** | **Relation instance** |
|---|---------------------|------------------------|
| **What** | Structure: relation name + attributes + domains + constraints | Actual set of tuples **at a time** |
| **Changes** | `ALTER TABLE` (infrequent, controlled) | `INSERT`, `UPDATE`, `DELETE` (continuous) |
| **Analogy** | Class definition | Objects alive in memory now |
| **Chapter 03 link** | Conceptual-level description | Snapshot of data |

### ShopKart schema (stable blueprint)

```text
Customers(customer_id, full_name, email, joined_on)
Products(sku, title, category, price_inr, stock_qty)
Orders(order_id, customer_id, order_date, total_inr)
OrderItems(order_id, sku, qty, line_price_inr)
Payments(payment_id, order_id, amount_inr, method, paid_at)
```

**Constraints on schema (not in each tuple’s syntax):**

- `customer_id` is **candidate key** (unique identifier)  
- `Orders.customer_id` **references** `Customers`  
- `line_price_inr` may differ from catalog `price_inr` (price at time of order — historical domain)

### ShopKart instance (Monday 2026-05-23)

Concrete tuples in `Customers`, `Orders`, etc. — finite set, changes every checkout.

```sql
INSERT INTO customers VALUES ('C-1003', 'Sneha Rao', 'sneha@mail.com', '2026-05-23');
-- Instance grows by one tuple; schema unchanged
```

**Database = collection of relation schemas + current instances** (plus views, constraints, catalog metadata).

---

## 6. Properties of relations (Codd’s rules on structure)

### Property 1: No duplicate tuples

Because \(R\) is a **set**, tuple \(t\) appears **at most once**.

**ShopKart:** Two rows both saying `(C-1001, Priya, priya@mail.com, 2024-01-15)` cannot both belong to `Customers` as distinct elements — they are **the same tuple**.

**Enforcement in SQL:**

```sql
PRIMARY KEY (customer_id)
-- or UNIQUE on full composite if natural key is composite
```

Without a key, the **table** might store duplicates; the **relation** you *intend* is violated.

---

### Property 2: Unordered tuples

Set \(\{t_1, t_2, t_3\} = \{t_3, t_1, t_2\}\).

**Implication:** There is no “first customer” in the model — only “the customer with id C-1001.”

**ShopKart:**

```sql
SELECT * FROM customers;  -- order not guaranteed
SELECT * FROM customers ORDER BY joined_on;  -- presentation order for UI only
```

**Exam trap:** `LIMIT 1` without `ORDER BY` → **arbitrary** row, not “first inserted.”

---

### Property 3: Unordered attributes (by name)

Tuple values are bound to **attribute names**, not column position 3 vs 4.

**ShopKart — these are the same tuple:**

```text
(customer_id=C-1001, full_name=Priya Nair, email=priya@mail.com, joined_on=2024-01-15)
(full_name=Priya Nair, joined_on=2024-01-15, customer_id=C-1001, email=priya@mail.com)
```

SQL allows `SELECT email, full_name FROM customers` — projection by **name**.

**Caveat:** SQL `INSERT INTO t VALUES (...)` uses **positional** values — must match table definition order. Prefer:

```sql
INSERT INTO customers (customer_id, full_name, email, joined_on)
VALUES ('C-1004', 'Dev Shah', 'dev@mail.com', '2026-05-23');
```

---

### Property 4: Atomic values (First Normal Form spirit)

Every attribute value in every tuple must be **atomic** for that domain — no repeating groups, no nested tables inside a cell in the pure model.

**Bad (non-relational cell):**

| order_id | line_items |
|----------|------------|
| O-600 | `{SKU-A:2, SKU-B:1}` |

**Good — two relations:**

`Orders(O-600, …)` and `OrderItems(O-600, SKU-A, 2, …)`, `OrderItems(O-600, SKU-B, 1, …)`.

(JSON columns in modern SQL are a **pragmatic compromise** — still know the classical rule.)

---

### Summary table

| Property | Model says | ShopKart practice |
|----------|------------|-------------------|
| No duplicate tuples | Set semantics | `PRIMARY KEY` / `UNIQUE` |
| Unordered tuples | No row index in logic | Never rely on physical insert order |
| Unordered attributes | Name-based | Named columns in INSERT/SELECT |
| Atomic domains | One value per cell | Split `OrderItems`; avoid multi-value cells |

---

## 7. NULL — meaning and problems

### What NULL is (in SQL)

**NULL** is a **marker** meaning “unknown,” “not applicable,” or “missing” — it is **not** a value in the domain (not zero, not empty string).

Codd later introduced NULL in SQL; purists note it complicates the pure relational algebra.

### ShopKart examples

| Attribute | NULL meaning |
|-----------|--------------|
| `customers.referral_code` | Customer did not use a referral — **unknown vs absent** |
| `products.discount_end_date` | No active discount — **not applicable** |
| `orders.coupon_code` | Order placed without coupon |
| `payments.paid_at` | Payment initiated but **not yet confirmed** |

```sql
INSERT INTO products (sku, title, category, price_inr, stock_qty, discount_end_date)
VALUES ('SKU-BOOK-1', 'DBMS Textbook', 'Books', 899.00, 50, NULL);
```

---

### Problem 1: Three-valued logic (3VL)

SQL logic: `TRUE`, ` `FALSE`, `UNKNOWN` (where NULL propagates).

**ShopKart query — find products with unknown discount status:**

```sql
SELECT sku FROM products WHERE discount_end_date = '2026-12-31';
-- Rows with NULL discount_end_date → UNKNOWN → excluded
```

**Pitfall:**

```sql
WHERE discount_end_date <> '2026-12-31'
-- Also excludes NULL!  NULL comparisons yield UNKNOWN, not TRUE
```

**Correct pattern:**

```sql
WHERE discount_end_date IS NULL
   OR discount_end_date <> '2026-12-31';
```

**Interview classic:** `WHERE x NOT IN (1, 2, NULL)` → **no rows** if any NULL in list, because `x = NULL` is UNKNOWN.

---

### Problem 2: Aggregates skewed

```sql
SELECT AVG(price_inr) FROM products;
-- Ignores NULL rows in column; COUNT(*) vs COUNT(price_inr) differ
```

If `price_inr` is NULL for discontinued listings, average is over **known prices only** — may mislead pricing dashboards unless documented.

---

### Problem 3: Duplicate semantics under UNIQUE

In SQL, **NULL ≠ NULL** for uniqueness (usually multiple NULLs allowed in unique column unless `NOT NULL`).

**ShopKart:**

```sql
UNIQUE (referral_code)  -- two customers with referral_code NULL often both allowed
```

Business wanted “at most one unknown” — **cannot** express with NULL alone; use sentinel or separate table.

---

### Problem 4: Design laziness

Developers use NULL instead of:

- Separate optional relation (`product_discounts`)  
- Explicit enum `discount_status IN ('active','none')`  
- Default value with clear meaning  

**ShopKart anti-pattern:** `shipped_at NULL` for both “not shipped yet” and “digital product — never ships” — same marker, different meanings → **semantic ambiguity**.

---

### Codd’s stance vs practice

- **Benefit:** Distinguish “email unknown” from empty string `''` (which might mean “customer cleared field”)  
- **Cost:** Algebra laws (De Morgan, NOT IN) break unless you learn 3VL  

**Professor’s guidance:** Use NULL sparingly, document meaning per column, prefer `NOT NULL` + explicit optional tables for critical paths (payments, inventory).

---

## 8. ShopKart mini catalog (pulling it together)

```text
RELATION SCHEMA (conceptual)
├── Customers(customer_id, full_name, email, joined_on)
├── Products(sku, title, category, price_inr, stock_qty, discount_end_date?)
├── Orders(order_id, customer_id, order_date, total_inr, coupon_code?)
├── OrderItems(order_id, sku, qty, line_price_inr)     -- PK (order_id, sku)
└── Payments(payment_id, order_id, amount_inr, method, paid_at?)

INSTANCE: finite sets of tuples satisfying FKs and domains
TABLE: SQL realization + keys + NULL + indexes + storage
```

**Relational algebra preview (next topics):** `σ` (select), `π` (project), `⋈` (join) operate on **relations as sets**, not on “row 5.”

---

## 9. Interview questions (with deep answers)

### Q1. “Is a relation the same as a table? Can I have a relation without a primary key?”

**Deep answer:**

**Same idea, different layer.** A **relation** is the mathematical ideal: duplicate-free set of tuples over named attributes. A **table** is the DBMS artifact implementing that ideal, often with loopholes.

Without `PRIMARY KEY` / `UNIQUE`:

- The **table** can contain duplicate rows (storage reality).  
- The **relation you intend** is then **undefined** as a set — which duplicate is “the” customer Priya?

**ShopKart:** `Customers` must have `customer_id` as **candidate key** — minimal set of attributes uniquely identifying each tuple. Primary key = chosen candidate key for enforcement.

**You can store data in a heap table without a key**, but you no longer have a well-defined relation — only a **bag** (multiset). Some theory texts use **bags** for SQL realism; Codd’s model uses **sets**.

**Sound bite:** “Table is the vessel; key constraints make the vessel hold a true relation.”

---

### Q2. “Why are tuples and attributes unordered? Doesn’t INSERT order matter?”

**Deep answer:**

**Model level:** Relations are sets of n-tuples; sets are unordered. Attributes are fields of a record identified by **name** in relational calculus (`{c | Customers(c) ∧ c.email = ...}`).

**Physical level:** PostgreSQL may store heap tuples in insertion order for a while, or reorganize after `VACUUM FULL`. **Clustered index** imposes physical order for performance — **not** logical order.

**ShopKart UI** sorts by `order_date DESC` for “recent orders” — that is **application presentation**, not a property of the `Orders` relation.

**INSERT order matters only operationally** (which row gets which `ctid`), not **semantically** (which customer is which business entity).

**Exam:** “First row from `SELECT`” without `ORDER BY` is **wrong question** at logical level; “any row matching predicate” is correct.

---

### Q3. “Explain NULL. Why is `WHERE column = NULL` wrong? How does it affect JOINs?”

**Deep answer:**

`NULL` means **missing or inapplicable** — not a value in the domain. Equality test is undefined → SQL uses `IS NULL` / `IS NOT NULL`.

**3VL:** `TRUE OR UNKNOWN = TRUE`, `FALSE OR UNKNOWN = UNKNOWN`, `NOT UNKNOWN = UNKNOWN`.

**ShopKart:**

```sql
-- Wrong
SELECT * FROM orders WHERE coupon_code = NULL;

-- Right
SELECT * FROM orders WHERE coupon_code IS NULL;
```

**JOIN effect:** `FROM orders o JOIN payments p ON o.order_id = p.order_id` **drops** orders with no payment row (inner join). Orders with `paid_at NULL` still appear if payment row exists. **Left join** needed for “all orders, payment if any”:

```sql
SELECT o.order_id, p.paid_at
FROM orders o
LEFT JOIN payments p ON p.order_id = o.order_id;
```

**Outer join + NULL:** Non-matching side fills with NULL markers — still not domain values.

**NOT IN trap:** Subquery returning NULL in list makes membership test UNKNOWN for all rows → empty result. Use `NOT EXISTS` or filter `WHERE x IS NOT NULL` in subquery.

**Design:** Prefer explicit status columns over NULL for workflow states (payment pending vs failed vs succeeded).

---

## 10. Exercises (attempt before hints)

### Exercise 1 — Schema, instance, and properties

ShopKart adds **Reviews(review_id, sku, customer_id, rating, review_text, verified_purchase)**.

1. Write the **relation schema** (attribute names) and state **degree**.  
2. Give **two sample tuples** as named attribute-value pairs (not only positional).  
3. Which **candidate key(s)** are plausible? Choose a **primary key**.  
4. One attribute where **NULL** is reasonable and one where it should be **NOT NULL** — justify with ShopKart business rules.  
5. Explain why storing `tags` as comma-separated `"fast,good,cheap"` in one cell violates **atomicity** — propose a normalized fix (relation name + key only).

<details>
<summary>Hint (after you try)</summary>

PK: review_id. Candidate: (customer_id, sku, review_date) if one review per pair per day. NULL: review_text empty optional; NOT NULL: sku, customer_id, rating. Tags → ReviewTags(review_id, tag).
</details>

---

### Exercise 2 — NULL and three-valued logic

Given `Products` with some `discount_end_date` NULL:

```sql
SELECT sku FROM products
WHERE discount_end_date NOT IN ('2026-06-01', '2026-12-31');
```

1. A product has `discount_end_date = NULL`. Does it appear in the result? Prove using 3VL.  
2. Rewrite the query to include SKUs with **NULL** discount dates **or** dates not in that list.  
3. `COUNT(*)` vs `COUNT(discount_end_date)` on the same table — when do they differ on ShopKart? What mistake would a PM make interpreting `AVG(price_inr)` if many NULL prices mean “discontinued”?

<details>
<summary>Hint (after you try)</summary>

NULL NOT IN (... ) → UNKNOWN → filtered out. Use IS NULL OR NOT IN or NOT EXISTS. COUNT(*) includes all rows; COUNT(col) excludes NULLs. AVG ignores NULLs — discontinued skew.
</details>

---

## 11. Chapter summary

| Concept | ShopKart one-liner |
|---------|-------------------|
| **Relation** | Set of customer/order tuples — no duplicates |
| **Table** | SQL storage of that set + keys + NULL + disk |
| **Tuple / attribute** | Row / column bound to a **domain** |
| **Domain** | Legal values + meaning (`price_inr` ≠ `stock_qty`) |
| **Schema vs instance** | Blueprint vs today’s live data |
| **Unordered** | Identify customer by `customer_id`, not row # |
| **NULL** | Unknown coupon — beware `= NULL`, `NOT IN`, aggregates |

Codd’s model gave DBMS theory a **clean mathematical spine**; SQL engineering adds **pragmatic friction** you must know for interviews and production.

---

*Prev: [Chapter 03 — DBMS architecture](./chapter-03-dbms-architecture.md) · Next: [Chapter 05 — ER modeling](../Data_modeling/chapter-05-er-modeling-university.md)*
