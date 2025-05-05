+++
date = '2025-04-29T07:16:22+07:00'
draft = true
title = 'PostgreSQL Partitioning: Native vs Inheritance + Trigger'
tags = ["PostgreSQL", "Partitioning", "Database"]
categories = ["Database"]
+++


Partitioning is a powerful technique in PostgreSQL to handle large-scale datasets. It can significantly improve **query performance**, **data maintenance**, and **scalability**.

But which method should you choose: **Native Partitioning** or the older **Inheritance + Trigger** approach?

This article dives into both techniques, comparing their mechanics, pros and cons, and real-world applicability to help you make an informed decision.

---

## What is partitioning and why does it matter?

**Partitioning** is the process of splitting a large table into smaller, more manageable pieces called *partitions*. Benefits include:

- Faster queries due to reduced I/O.
- Simplified table management.
- Efficient archival or deletion of old data.

> Think of partitioning like organizing massive file cabinets by month — it's easier to locate and remove specific data chunks.

---

## Historical context in PostgreSQL

Partitioning in PostgreSQL has evolved over time:

| PostgreSQL Version | Partitioning Method        | Notes                           |
|--------------------|----------------------------|---------------------------------|
| ≤ 9.6              | Inheritance + Trigger      | Manual, flexible but complex    |
| ≥ 10               | Native Partitioning (built-in) | Simple, efficient, and integrated |

---

## 3. How each method works

### 🛠 Inheritance + Trigger

This older method uses table inheritance and insert triggers.

```sql
CREATE TABLE logs (
    id serial,
    log_time timestamp NOT NULL,
    log_message text
);

CREATE TABLE logs_2024_01 () INHERITS (logs);
CREATE TABLE logs_2024_02 () INHERITS (logs);

CREATE FUNCTION logs_insert_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF (NEW.log_time >= '2024-01-01' AND NEW.log_time < '2024-02-01') THEN
        INSERT INTO logs_2024_01 VALUES (NEW.*);
    ELSIF (NEW.log_time >= '2024-02-01' AND NEW.log_time < '2024-03-01') THEN
        INSERT INTO logs_2024_02 VALUES (NEW.*);
    ELSE
        RAISE EXCEPTION 'Date out of range';
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER insert_logs_trigger
BEFORE INSERT ON logs
FOR EACH ROW EXECUTE FUNCTION logs_insert_trigger();
```

✅ **Explanation**:
- The `logs` table is abstract and holds no data directly.
- The trigger checks `log_time` and routes the row to the correct child table.

---

### ⚙ Native Partitioning (PostgreSQL ≥ 10)

Native partitioning simplifies the setup using built-in features.

```sql
CREATE TABLE logs (
    id serial,
    log_time timestamp NOT NULL,
    log_message text
) PARTITION BY RANGE (log_time);

CREATE TABLE logs_2024_01 PARTITION OF logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE logs_2024_02 PARTITION OF logs
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

✅ **Key Benefits**:
- No triggers or manual routing logic.
- PostgreSQL automatically inserts into the correct partition.
- Optimized query performance through **pruning**.

---

## Pros and Cons comparison

| Feature                        | Inheritance + Trigger     | Native Partitioning          |
|-------------------------------|----------------------------|-------------------------------|
| PostgreSQL Version Support     | ≤ 9.6                      | ≥ 10                          |
| Automatic Query Pruning        | ❌ No                      | ✅ Yes                         |
| Insert Routing Logic           | Manual via triggers        | Automatic                     |
| Ease of Maintenance            | ❌ Tedious                 | ✅ Simple                      |
| Custom Logic Flexibility       | ✅ High                    | ❌ Limited                    |

---

## Performance Benchmarks

In most benchmarks:

- **Insert Performance**: Native is 20–50% faster.
- **Query Performance**: Native benefits from partition pruning, scanning only the relevant partitions.

---

## When to use which?

### ✅ Use **Native Partitioning** when:
- You're using PostgreSQL 10 or later.
- You prioritize performance and simplicity.
- Your partitioning scheme fits range/list/hash models.

### ✅ Use **Inheritance + Trigger** when:
- You're stuck with PostgreSQL ≤ 9.6.
- You need custom insert logic not supported natively.

---

## Migrating from Inheritance to Native

When transitioning:

- Plan your schema migration carefully.
- Use native commands like `ATTACH PARTITION` and `DETACH PARTITION`.
- Test your constraints, indexes, and foreign keys.

---

## Conclusion

Both partitioning methods are valid depending on your context:

- **Native partitioning** is ideal for modern setups — faster, simpler, and better integrated.
- **Inheritance + Trigger** remains useful for legacy systems or when custom logic is essential.