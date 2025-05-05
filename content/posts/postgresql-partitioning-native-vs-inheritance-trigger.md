+++
date = '2025-04-29T07:16:22+07:00'
draft = false
title = 'PostgreSQL Partitioning: Native vs Inheritance + Trigger'
tags = ["PostgreSQL", "Partitioning", "Database"]
categories = ["Database"]
+++

Partitioning is a powerful technique in PostgreSQL for managing large datasets. It can significantly enhance **query performance**, **data management**, and **scalability**.

But which method should you choose: **Native Partitioning** or the older **Inheritance + Trigger** approach?

This article will analyze both techniques, compare their mechanisms, pros and cons, and practical applications to help you make an informed decision.

---

## What is Partitioning and Why is it Important?

**Partitioning** is the process of dividing a large table into smaller, more manageable pieces called *partitions*. The benefits include:

- Faster queries due to reduced I/O.
- Simplified table management.
- Efficient storage or deletion of old data.

> Think of partitioning as organizing a massive filing cabinet by month — it’s much easier to find and remove specific sections of data.

---

## Historical Context in PostgreSQL

Partitioning in PostgreSQL has evolved over time:

| PostgreSQL Version | Partitioning Method            | Notes                            |
|--------------------|--------------------------------|----------------------------------|
| ≤ 9.6             | Inheritance + Trigger         | Manual, flexible but complex     |
| ≥ 10              | Native Partitioning (built-in) | Simple, efficient, and integrated|

---

## How Each Method Works

### 🛠 Inheritance + Trigger

This older method uses table inheritance and triggers to route data.

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
- The `logs` table is abstract and does not store data directly.
- The trigger checks `log_time` and routes rows to the appropriate child table.

---

### ⚙ Native Partitioning (PostgreSQL ≥ 10)

Native Partitioning simplifies the setup using built-in features.

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
- No need for triggers or manual routing logic.
- PostgreSQL automatically inserts into the correct partition.
- Query performance is optimized through **pruning**.

---

## Pros and Cons Comparison

| Feature                        | Inheritance + Trigger     | Native Partitioning          |
|--------------------------------|---------------------------|-------------------------------|
| PostgreSQL Version Support     | ≤ 9.6                     | ≥ 10                          |
| Automatic Query Pruning        | ❌ No                     | ✅ Yes                        |
| Insert Routing Logic           | Manual via triggers       | Automatic                     |
| Maintenance Simplicity         | ❌ Difficult              | ✅ Simple                     |
| Custom Logic Flexibility       | ✅ High                   | ❌ Limited                    |

---

## Performance

In most benchmarks:

- **Insert Performance**: Native is 20–50% faster.
- **Query Performance**: Native benefits from partition pruning, scanning only relevant partitions.

---

## When to Use Each Method

### ✅ Use **Native Partitioning** when:
- You are using PostgreSQL 10 or later.
- You prioritize performance and simplicity.
- Your partitioning model fits range/list/hash.

### ✅ Use **Inheritance + Trigger** when:
- You are limited to PostgreSQL ≤ 9.6.
- You need custom insert logic not supported natively.

---

## Migrating from Inheritance to Native

When migrating:

- Plan your schema transition carefully.
- Use native commands like `ATTACH PARTITION` and `DETACH PARTITION`.
- Check your constraints, indexes, and foreign keys.

---

## Conclusion

Both partitioning methods have their value depending on the context:

- **Native Partitioning** is ideal for modern setups — faster, simpler, and better integrated.
- **Inheritance + Trigger** remains useful for legacy systems or when custom logic is required.